# transport — the custom gRPC + libp2p P2P transport (start here)

How one DePIN node opens an authenticated gRPC connection to another, over either
a **public TCP** socket or a **libp2p relay** stream — without the caller having to
know which. This is the onboarding guide: how the layers fit and what to expect.
For the dependency-injection wiring that builds these objects see
[`configuration.md`](configuration.md); for the mTLS / node-ID checks the
transport relies on see [`identity-trust.md`](identity-trust.md).

> The transport is a thin custom stack over standard gRPC. Every connection is
> TLS, and the TLS layer is the same regardless of whether the bytes travel over a
> raw TCP socket or a libp2p stream — that uniformity is the whole point.

---

## 1. What it is & why

Three moving parts, top to bottom:

- **`Dialer`** (`internal/grpcutil/dial`) — the public handle. `DialNode` /
  `DialNodeURL` take a `vo.PeerURL`, hand back a `*Conn` (a `grpc.ClientConnInterface`).
  It owns the connection **pool**, the **mTLS** config, and dial **timeouts**.
- **`Connector`** (`internal/grpcutil/connector`) — decides *which wire* to open:
  a TCP socket (`TCPConnector`), a libp2p stream (`P2PConnector`), or **route between
  them by address** (`HybridConnector`). It performs the TLS client handshake and
  returns a `ConnectorConn`.
- **`PeerURL`** (`internal/vo`) — the address value object. It can carry a TCP
  `host:port`, a libp2p `p2pID@/multiaddr`, or both. The connector reads it to decide
  the route.

Why not just `grpc.Dial`? Workers behind NAT have no public TCP address, so they are
reachable only through a **libp2p circuit relay**. But the system still wants one
identity model (node-ID-pinned mTLS) and one dial API across both paths. Wrapping a
libp2p stream as a `net.Conn` lets the *exact same* TLS + gRPC stack run over either
transport — the dual-stack lives entirely in the `Connector`.

---

## 2. Mental model

```
caller
  │  DialNode(ctx, PeerURL)
  ▼
Dialer (dial/dial.go)                 — pool + mTLS config + timeout
  │  ClientTLSConfig(nodeURL.ID)      → pins peer node ID
  ▼
Connector.DialContext(ctx, tlsConfig, url)
  │
  ├─ HybridConnector ─ routes by ADDRESS the PeerURL advertises:
  │        url.TcpAddress() != "" → TCP   (else)   url.P2PAddress() != "" → P2P
  │
  ├─ TCPConnector → net.Dial("tcp") ───────────────┐
  │                                                 ▼
  └─ P2PConnector → host.NewStream over libp2p ─→ raw net.Conn
                    (circuit relay v2 if NAT'd)     │
                                                    ▼
                                          tls.Client(rawConn, tlsConfig)
                                          HandshakeContext  ← mTLS, node-ID pin
                                                    ▼
                                          grpcconn.New(conn) → gRPC over the stream
                                                    ▼
                                          *Conn  (pooled, reusable)
```

The key insight: below `tls.Client`, both branches are just a `net.Conn`. The libp2p
stream is wrapped as one (`p2p.LibP2PConn`), so TLS and gRPC neither know nor care
which transport carries them.

---

## 3. Glossary

| Term | Meaning | Where |
| ---- | ------- | ----- |
| **`Dialer`** | public dial handle: pool + mTLS + timeout; `DialNode`/`DialNodeURL` | `internal/grpcutil/dial` `dial.go` |
| **`Connector`** | interface `DialContext(ctx, *tls.Config, PeerURL) (ConnectorConn, error)` | `internal/grpcutil/connector` `connector.go` |
| **`TCPConnector`** | dials a raw TCP socket, then `tls.Client` | `connector/tcp_connector.go` |
| **`P2PConnector`** | opens a libp2p stream (wrapped as `net.Conn`), then `tls.Client` | `connector/p2p_connector.go` |
| **`HybridConnector`** | **routes** TCP-if-advertised else P2P; **not** a try-then-fallback retry | `connector/hybrid_connector.go` |
| **`PeerURL`** | address VO: `ID`, `Host`/`Port` (TCP), `P2PID`/`P2PAddr` (libp2p) | `internal/vo/peer_url.go` |
| **`ConnectorConn`** | the encrypted `net.Conn` a connector returns (carries `ConnectionState`) | `connector/conn.go` |
| **`DialFunc`** | low-level `func(ctx, network, url, ...DialOption) (net.Conn, error)` | `connector/dial.go` |
| **`LibP2PConn`** | libp2p `network.Stream` wrapped to satisfy `net.Conn` | `connector/p2p/conn.go` |
| **`LibP2PListener`** | accepts inbound libp2p streams as `net.Conn`s (server side) | `connector/p2p/conn.go` |
| **circuit relay v2** | NAT'd worker is reached via a relay node (`p2p-circuit` multiaddr) | `connector/dial.go` `Replay` |
| **pool** | LRU of live gRPC conns keyed by `"node:"+url`; reused across dials | `dial/dial.go` `NewDefaultConnectionPool` |

---

## 4. Life of a connection (the spine)

### Dial

`Dialer.DialNode(ctx, nodeURL, opts)` (`dial/dial.go`) builds a pool key
`"node:"+nodeURL.String()` and calls `dialPool`. The pool either returns a live
connection for that key or invokes the dial closure, which calls
`dialEncryptedConn(ctx, nodeURL, d.TLSOptions.ClientTLSConfig(nodeURL.ID))`. Note the
TLS config is built **with the target's node ID** — that's the identity pin (see
`identity-trust.md`). `DialNodeURL` is the same with default options.

### Route selection (Hybrid)

`HybridConnector.DialContext` (`hybrid_connector.go`) is a pure `switch` on the
**address the `PeerURL` advertises**:

```go
case url.TcpAddress() != "": return c.tcpConnector.DialContext(...)
case url.P2PAddress() != "": return c.p2pConnector.DialContext(...)
default:                     return nil, EmptyAddressError
```

This is **routing, not failover** — there is no "try TCP, then retry over P2P." A
node that advertises a TCP address is dialed over TCP; a node that advertises only a
libp2p address is dialed over the relay. `PeerURL.TcpAddress()` returns `""` unless
both `Host` and `Port` are set, so a relay-only worker naturally falls to the P2P arm.

### TCP path

`TCPConnector.DialContext` (`tcp_connector.go`) → `DialContextUnencrypted` →
`net.Dial("tcp", url.Address())`, sets `TCP_USER_TIMEOUT` (linux, `SetUserTimeout`,
default 15 min), then `tls.Client(rawConn, tlsConfig)` + `HandshakeContext`, and
returns a `tlsConnWrapper`.

### P2P path

`P2PConnector.DialContext` (`p2p_connector.go`) dials `url.P2PAddress()` via the
libp2p host. The default dialer `DefaultP2PDialer(host)` (`connector/dial.go`) parses
the `peerID@serverAddr` form, optionally **reserves** a relay slot (`client.Reserve`)
or builds a **`/p2p-circuit/p2p/<id>`** multiaddr to reach a NAT'd peer through a
relay (the `Replay` `DialOption`), `host.Connect`s, then `LibP2PDialer.Dial` opens a
`host.NewStream(ctx, pid, ProtocolID)` where `ProtocolID = "/myapp/grpc/1.0.0"`. The
stream is wrapped as a `LibP2PConn` (`p2p/conn.go`) so `tls.Client` runs over it
identically to the TCP case.

### TLS + gRPC

Both connectors finish by handing the encrypted `net.Conn` to `grpcconn.New(conn)`
(in `dialEncryptedConn`), producing the pooled `pool.RawConn`. The returned `*Conn`
exposes `GetState` so callers can read the verified `tls.ConnectionState`.

### Server side

`server.New` builds a libp2p host with `libp2p.New(...)` —
`worker/server/server.go` enables `EnableAutoNATv2()`, `EnableRelay()`, and
`EnableHolePunching()` so a NAT'd worker can be reached via the relay; the coordinator
host (`coord/server/server.go`) is plainer. Inbound libp2p streams are accepted by a
`p2p.NewLibP2PListener(host, p2p.ProtocolID)` that yields `net.Conn`s into the gRPC
server. libp2p itself runs with `insecure` security (`libp2p.Security(insecure.ID, …)`)
**on purpose** — the real authentication is the TLS layer we add on top, not libp2p's.

---

## 5. Key types / entry points

```go
// dial — the public handle (internal/grpcutil/dial/dial.go)
func NewDefaultDialer(c connector.Connector, o *tlsopts.Options) Dialer
func NewDefaultPooledDialer(c connector.Connector, o *tlsopts.Options) Dialer
func (d Dialer) DialNode(ctx, nodeURL vo.PeerURL, opts DialOptions) (*Conn, error)
func (d Dialer) DialNodeURL(ctx, nodeURL vo.PeerURL) (*Conn, error)

// connector — the route layer (internal/grpcutil/connector)
type Connector interface {
    DialContext(ctx, *tls.Config, vo.PeerURL) (ConnectorConn, error)
}
func NewDefaultTCPConnector() *TCPConnector
func NewDefaultP2PConnector(host host.Host) *P2PConnector
func NewHybridConnector(*TCPConnector, *P2PConnector) *HybridConnector

// address VO (internal/vo/peer_url.go)
func ParsePeerURL(s string) (PeerURL, error)        // "id@tcp:host:port;p2p:id:/multiaddr"
func (u *PeerURL) TcpAddress() string                // "" unless Host && Port set
func (u *PeerURL) P2PAddress() string                // "<multiaddr>/p2p/<id>"

// libp2p<->net.Conn bridge (internal/grpcutil/connector/p2p/conn.go)
func NewLibP2PListener(h host.Host, proto protocol.ID) *LibP2PListener
const ProtocolID = protocol.ID("/myapp/grpc/1.0.0")
```

Wiring lives in the peers: `coord/peer.go` (`setupServer`) builds a
`NewHybridConnector(NewDefaultTCPConnector(), NewDefaultP2PConnector(srv.P2PHost()))`
and feeds it to `NewDefaultDialer`; `worker/peer.go` picks **one** connector — TCP if
`HavePublicAddress`, else P2P — since a single worker reaches peers over a single path.

---

## 6. Gotchas / edge cases

- **Hybrid routes, it does not fail over.** A common misread: it does *not* try TCP
  and retry on P2P. It branches once on `url.TcpAddress()`/`url.P2PAddress()`. If you
  want a worker reached via relay, its advertised `PeerURL` must carry *only* the P2P
  address — a stray TCP `host:port` will force the TCP arm.
- **libp2p security is `insecure` by design.** The TLS layer above provides the
  authentication; libp2p's own crypto is disabled (`libp2p.Security(insecure.ID, …)`)
  to avoid double-encrypting. Do not "fix" this by enabling Noise/TLS in libp2p.
- **`TcpAddress()` returns `""` unless both host and port are set** — that empty
  string is exactly what steers `HybridConnector` to the P2P branch.
- **The audit/scale-out coordinator role is TCP-only.** `coord/peer.go`'s
  `setupDialer` builds a bare `NewDefaultTCPConnector()` and binds no server, so
  relay-only workers are **not dialable** from that role.
- **`DialAddressUnencrypted` is unsupported** — it returns an error; everything goes
  through TLS.
- **The pool keys on the URL string** (`"node:"+url.String()`), so two `PeerURL`s
  that stringify differently won't share a pooled connection even to the same node.
- **`DefaultP2PDialer` has a hard 10s connect timeout** independent of the Dialer's
  `DialTimeout` (default 20s) — relay setup can dominate latency.

---

## 7. Where to go next

- [`../internal/grpcutil`](../internal/grpcutil) — the connector/dial/pool source.
- [`../internal/peertls`](../internal/peertls) — the mTLS layer the handshake uses.
- [`identity-trust.md`](identity-trust.md) — what `ClientTLSConfig(nodeURL.ID)`
  actually verifies (node-ID pinning, cert chains, the five auth checkpoints).
- [`configuration.md`](configuration.md) — how `Dialer`/`Connector`/server are
  wired into a peer via cfgstruct + mud.
- [`../SOURCE_STRUCTURE.md`](../SOURCE_STRUCTURE.md) — the layered peer architecture
  the transport plugs into.

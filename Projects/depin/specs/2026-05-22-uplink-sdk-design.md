# uplink-sdk — Design

A new top-level Go package, `uplink-sdk/`, that gives external callers a small,
stateless library for talking to coord. It supersedes nothing yet — the existing
`uplink/` application stays intact and may migrate to use this SDK later.

## Goals

- Single Go package external programs can import to upload, download, and
  manage files on coord.
- Authenticate using a `*identity.FullIdentity` (peer identity) plus a
  `*vo.PrivKey` (signing key), the same pair `uplink/peer.go:36` already loads.
- Stateless: no SQLite, no on-disk staging, no multipart bookkeeping.
- No changes to `uplink/`. Code that currently sits in `uplink/internal/eestream2`
  and `uplink/application/service/workerclient` is **copied** into
  `uplink-sdk/internal/` (drift is the accepted trade-off; consolidation can
  follow later).

## Non-Goals

- Resumable / multipart uploads (existing `uplink/` keeps that responsibility).
- HTTP delivery, CLI, background workers, persistence — none of that lives in
  the SDK.
- Language bindings other than Go.

## Package Layout

```
uplink-sdk/
├── client.go        Client struct + accessors + Close
├── options.go       ClientOption funcs (WithCoordAddr, WithIdentity, WithSigner, …)
├── auth.go          gRPC auth credentials (AuthenticationData signer)
├── upload.go        UploadFile: Begin → encrypt → EC → PutPiece → Commit
├── download.go      DownloadFile: GetDownloadInfo → GetPiece → EC decode → decrypt
├── placement.go     ListPlacements
├── account.go       RegisterAccount, GetAccount
├── internal/
│   ├── eestream/    Copied from uplink/internal/eestream2
│   ├── ers_schema/  Copied from uplink/internal/ers_schema
│   └── workerclient/ Copied from uplink/application/service/workerclient
└── README.md
```

Package name: `uplinksdk`. Import path: `aioz-depin/uplink-sdk`.

## Public API

```go
type Client struct { /* unexported */ }

type ClientOption func(*options)

func WithCoordAddr(addr string) ClientOption        // required
func WithIdentity(*identity.FullIdentity) ClientOption // required
func WithSigner(*vo.PrivKey) ClientOption           // required
func WithLogger(*zap.Logger) ClientOption           // optional
func WithEncryptionParameters(common.EncryptionParameters) ClientOption // optional

func New(ctx context.Context, opts ...ClientOption) (*Client, error)
func (c *Client) Close() error

func (c *Client) UploadFile(ctx context.Context, r io.Reader, p UploadParams) (vo.UUID, error)
func (c *Client) DownloadFile(ctx context.Context, fileID vo.UUID, w io.Writer) error
func (c *Client) ListPlacements(ctx context.Context) ([]*placementpb.Placement, error)
func (c *Client) RegisterAccount(ctx context.Context, email string) error
func (c *Client) GetAccount(ctx context.Context) (*clientpb.Client, error)
```

`UploadParams` carries `Filename`, `Size`, `Metadata`, optional `PlacementID`,
optional `ContractID`.

## Identity & Auth

- `WithIdentity` provides the peer's `FullIdentity` (cert chain, ID). Stored
  on the client; surfaced for callers that need their own ID.
- `WithSigner` provides the cosmos-style `vo.PrivKey` used to sign
  `AuthenticationData{PublicKey, Timestamp}` for the existing per-call
  authorization header (ported from `uplink/application/service/hubclient.go:108`).
- gRPC dial uses `grpcutil.GrpcConnBuilder` with `WithAuthorization` and the
  signer; TLS toggling deferred (current callers use insecure).

## Upload Flow

1. `FileService.CreateFile` — get file ID + segment/encryption params.
2. For each segment:
   - `FileService.BeginSegment` — get order limits + derived key + redundancy.
   - Encrypt with `pkg/encryption` (block cipher + nonce derived from
     `BeginSegment.Nonce`).
   - If `partSize ≤ InlineThreshold`: read encrypted bytes,
     `FileService.MakeInlineSegment`.
   - Otherwise: erasure-encode via `internal/eestream`; concurrently
     `PutPiece` to each `OrderLimit` via `internal/workerclient`;
     cancel remaining once `OptimalThreshold` reached; require at least
     `RequiredCount` successes; `FileService.CommitSegment` with the
     successful piece hashes.
3. `FileService.CommitFile`.

## Download Flow

1. `FileService.GetDownloadInfo` for the file.
2. For each segment: fetch enough pieces via `workerclient.GetPiece`,
   EC-decode, decrypt, stream to writer.

## Errors

- Constructor: returns wrapped error if any required option missing or dial
  fails.
- Per-call methods: surface coord/worker errors with package-prefixed errs
  (`uplinksdk:`).

## Out of Scope for This Spec

- TLS configuration on the coord dial.
- Multipart / resumable uploads.
- Metrics / tracing.
- Concurrent file uploads coordinated by the SDK (callers can run their own).

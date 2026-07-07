---
type: doc
project: backend
tags: [backend, networking, socks5, ssh, microsocks]
created: 2026-07-05
---

# SOCKS5 Proxy via microsocks + SSH Tunnel

Route HTTP requests (curl, Go, etc.) through a remote machine's IP using a
SOCKS5 proxy (microsocks) secured behind an SSH tunnel, with username/password
authentication.

## Architecture

```
[Go app / curl]                       [Remote host]
127.0.0.1:1080  ──SSH tunnel──>  127.0.0.1:1080 (microsocks)  ──> external IP
   (local)        (encrypted)         (SOCKS5 + user/pass)
```

- **SSH** secures the transport and is the real access gate.
- **microsocks user/pass** is an extra layer so a random local process on the
  remote host can't blindly use the proxy.

---

## 1. Install microsocks (remote host)

```bash
# Debian/Ubuntu
sudo apt install microsocks

# or build from source
git clone https://github.com/rofl0r/microsocks
cd microsocks && make
sudo cp microsocks /usr/local/bin/
```

## 2. Run microsocks with user/password (remote host)

```bash
microsocks -i 127.0.0.1 -p 1080 -u myuser -P mypass
```

| Flag | Meaning |
|------|---------|
| `-i 127.0.0.1` | Bind to localhost only — **not** exposed to the internet; reachable only via SSH tunnel |
| `-p 1080` | Listen port |
| `-u myuser` | SOCKS5 username |
| `-P mypass` | SOCKS5 password |

Quick background run (dies on reboot):

```bash
nohup microsocks -i 127.0.0.1 -p 1080 -u myuser -P mypass > /tmp/microsocks.log 2>&1 &
```

### Recommended: systemd service

Create `/etc/systemd/system/microsocks.service`:

```ini
[Unit]
Description=microsocks SOCKS5 proxy
After=network.target

[Service]
ExecStart=/usr/local/bin/microsocks -i 127.0.0.1 -p 1080 -u myuser -P mypass
Restart=always
User=nobody

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now microsocks
sudo systemctl status microsocks
```

## 3. Open the SSH tunnel (local machine)

```bash
# interactive (keeps a shell open)
ssh -L 1080:127.0.0.1:1080 user@remote-host

# background, forward-only
ssh -N -f -L 1080:127.0.0.1:1080 user@remote-host
```

| Flag | Meaning |
|------|---------|
| `-L 1080:127.0.0.1:1080` | Forward local port 1080 → remote's 127.0.0.1:1080 |
| `-N` | Don't run a remote command |
| `-f` | Go to background |

### Recommended: auto-reconnecting tunnel with autossh

Plain `ssh -N -f` dies on network blips:

```bash
sudo apt install autossh

autossh -M 0 -N -f \
  -o "ServerAliveInterval 30" \
  -o "ServerAliveCountMax 3" \
  -L 1080:127.0.0.1:1080 \
  user@remote-host
```

## 4. Test

```bash
# should print the REMOTE machine's public IP
curl --socks5 myuser:mypass@127.0.0.1:1080 http://ifconfig.me

# alternative syntax
curl --proxy socks5://myuser:mypass@127.0.0.1:1080 http://ifconfig.me
```

If you see your local IP instead, the proxy is not being used.

---

## 5. Use from Go

```bash
go get golang.org/x/net/proxy
```

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"time"

	"golang.org/x/net/proxy"
)

func main() {
	auth := &proxy.Auth{User: "myuser", Password: "mypass"}

	dialer, err := proxy.SOCKS5("tcp", "127.0.0.1:1080", auth, proxy.Direct)
	if err != nil {
		panic(err)
	}

	// proxy.SOCKS5 returns a type that also implements proxy.ContextDialer
	contextDialer, ok := dialer.(proxy.ContextDialer)
	if !ok {
		panic("dialer is not a ContextDialer")
	}

	client := &http.Client{
		Transport: &http.Transport{
			DialContext: contextDialer.DialContext,
		},
		Timeout: 30 * time.Second,
	}

	resp, err := client.Get("http://ifconfig.me")
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	fmt.Println(string(body)) // remote machine's public IP
}
```

### Zero-code alternative: environment variable

Go's default transport honors `ALL_PROXY`:

```bash
export ALL_PROXY=socks5://myuser:mypass@127.0.0.1:1080
```

Any `http.Client` using `http.ProxyFromEnvironment` (including the default
client) routes through the proxy automatically.

---

## Notes

- **HTTPS works transparently** — with SOCKS5 the TLS handshake happens
  end-to-end through the tunnel; the proxy only forwards bytes.
- **DNS resolution**: Go's `proxy.SOCKS5` always resolves hostnames at the
  proxy side (equivalent to curl's `socks5h://`), which is usually what you
  want for a tunnel.
- **Security boundary**: binding microsocks to `127.0.0.1` + SSH tunnel is the
  real protection. The SOCKS5 user/pass is defense in depth, not the primary
  gate — never bind microsocks to `0.0.0.0` with only password auth.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `connection refused` on 127.0.0.1:1080 | Tunnel not up — re-run the `ssh -L` / `autossh` command |
| `socks: authentication failed` | User/pass mismatch with microsocks `-u`/`-P` flags |
| Response shows local IP | Client not using the proxy — check `--socks5` flag / `ALL_PROXY` |
| Tunnel keeps dying | Use `autossh` with `ServerAliveInterval` (see step 3) |
| microsocks gone after reboot | Use the systemd service (see step 2) |

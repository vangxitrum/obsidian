# Edgeserver: set HTTP response headers from file metadata

## Context

The coordinator already stores and emits per-file HTTP metadata, but it never
reaches the browser. Today `edgeserver/handler.go` hardcodes
`Content-Type: application/octet-stream` and streams the body straight to the
client, discarding everything else.

Trace of what exists vs. what's missing:

- **Coord (done):** `coord/file/metadata.go` `FileMetadata{SystemHeaders, UserMetadata}`,
  allow-listed to 6 system headers (`content-type`, `cache-control`,
  `content-disposition`, `content-encoding`, `content-language`, `expires`),
  keys lowercased (`Normalize`) and CR/LF/control-char injection-rejected
  (`Validate`, `MaxUserMetadataBytes = 2048`). `buildManifest`
  (`coord/file/endpoint.go:734-740`) already stamps `SystemHeaders`/`UserMetadata`
  onto every `DownloadManifest` (proto fields 4/5).
- **go-sdk proto (stale):** `go-sdk/pkg/pb/coord/file/v1/file.pb.go:1071`
  `DownloadManifest` has only `FileId`, `FileSize`, `Segments` - it lacks fields
  4/5, so coord's metadata is silently dropped on unmarshal at the SDK boundary.
- **go-sdk API (gap):** `DownloadByTicket(ctx, ticket, w io.Writer)`
  (`go-sdk/download.go:89`) returns only `error`; the manifest is never surfaced
  to the caller.
- **edgeserver (gap):** `handler.go:16 handleDownload` sets only the hardcoded
  content type.

Outcome: an object downloaded through the edge server carries its stored
`Content-Type`, `Cache-Control`, `Content-Disposition`, etc., plus arbitrary
user metadata as `X-User-Meta-<key>` headers.

## Decisions (confirmed with user)

1. **Header scope:** the 6 allow-listed system headers set directly, plus each
   `user_metadata` entry emitted as `X-User-Meta-<key>`.
2. **Download-only:** do NOT add an upload-side metadata path to the go-sdk.
   Tests set metadata on the coord file row via a fixture.
3. **SDK API:** a header-sink callback option on `DownloadByTicket` -
   `WithHeaderSink(func(system, user map[string]string))`, fired after the
   manifest resolves and before the first body byte.

`Content-Length` is intentionally NOT set (response stays streamed/chunked);
it isn't in the allow-list and avoids any plaintext-vs-encrypted size question.

## Work

### Layer 1 - go-sdk proto refresh (`/home/tuan/work/depin-workspace/go-sdk`)

Bring `DownloadManifest` (and the rest of the `coord/file/v1` pb package) up to
date with depin so fields 4/5 unmarshal. The maintained mechanism is
`hack/sync-from-depin.sh` steps 6 / 7.5 / 8, but that script early-exits because
it guards on the now-deleted `uplink-sdk/`. Apply its transforms targeted to the
one changed package:

1. Copy depin's generated `pkg/pb/coord/file/v1/*.go` over go-sdk's same path.
2. Rewrite import prefixes (sed, as the script does):
   `aioz-depin/internal/` -> `aioz-depin/go-sdk/internal/`,
   `aioz-depin/pkg/` -> `aioz-depin/go-sdk/pkg/` (also fixes the `vo.UUID`
   customtype in struct tags).
3. Delete every `proto.RegisterEnum(` line (the duplicate-registration guard the
   migration note documents).
4. `gofmt -w` + `make build` in go-sdk to confirm it compiles.

Only `map<string,string>` fields are added - no new type dependencies.

### Layer 2 - go-sdk download API (`go-sdk/download.go`)

Add an options type and thread it through `DownloadByTicket` only
(`DownloadManifest`, `DownloadFile` untouched):

```go
type DownloadOption func(*downloadOptions)
type downloadOptions struct { headerSink func(system, user map[string]string) }
func WithHeaderSink(fn func(system, user map[string]string)) DownloadOption { ... }

func (c *Client) DownloadByTicket(ctx context.Context, ticket string,
    w io.Writer, opts ...DownloadOption) error {
    // ... resolve manifest via GetDownloadInfoByTicket ...
    if o.headerSink != nil {
        o.headerSink(resp.Manifest.GetSystemHeaders(), resp.Manifest.GetUserMetadata())
    }
    return c.DownloadManifest(ctx, resp.Manifest, w)
}
```

Note the current sibling signature already takes `ticket string` (commit
c018645, ahead of depin's pinned `bb4f12b`); decoding happens inside the SDK.

### Layer 3 - edgeserver handler (`edgeserver/handler.go`)

Rewrite `handleDownload` to pass the raw ticket string (drop the local
base64 decode - the SDK does it) and register a header sink:

```go
raw := r.URL.Query().Get("ticket")
if raw == "" { http.Error(w, "missing ticket query parameter", 400); return }

setHeaders := func(system, user map[string]string) {
    for k, v := range system {           // keys already lowercase-canonical
        w.Header().Set(k, v)             // http.Header canonicalizes
    }
    if _, ok := system["content-type"]; !ok {
        w.Header().Set("Content-Type", "application/octet-stream")
    }
    for k, v := range user {
        if !validHeaderToken(k) { continue }   // defensive: skip non-token keys
        w.Header().Set("X-User-Meta-"+k, v)
    }
}
if err := s.client.DownloadByTicket(r.Context(), raw, w,
    uplinksdk.WithHeaderSink(setHeaders)); err != nil {
    s.logger.Error("download failed", zap.Error(err))
}
```

- Coord already injection-validates values; `validHeaderToken(k)` is defense in
  depth against a user-metadata key that isn't a legal HTTP field-name token
  (e.g. contains a space). Use `golang.org/x/net/http/httpguts.ValidHeaderFieldName`
  on `"X-User-Meta-"+k`, or a small local token check.
- The sink fires before streaming, so all headers are buffered before the
  implicit `WriteHeader` on first body write.

### Re-pin go-sdk into depin

For fast local iteration, temporarily point the replace at the checkout:
`go mod edit -replace aioz-depin/go-sdk=../go-sdk` then `go mod tidy`. To
finalize, follow the established recipe (memory `edgeserver-go-sdk-migration`):
commit/push go-sdk, compute `v0.0.0-<UTC-date>-<12-char-sha>`, update the
`replace` in `go.mod:404`, `go mod tidy`.

## Files touched

- `go-sdk/pkg/pb/coord/file/v1/file.pb.go` (+ `file.proto` if present) - resynced
- `go-sdk/download.go` - `DownloadOption` / `WithHeaderSink` + sink call
- `edgeserver/handler.go` - header sink, drop local decode
- `depin/go.mod` - re-pin go-sdk replace
- new test file (below)

## Verification (end-to-end, preferred over unit tests)

Add a testplanet-backed test (in `edgeserver/` or `internal/testplanet/`), run
with `DEPIN_TEST_POSTGRES` against the running `coord-db` container
(`localhost:5445`, db `hub`, admin/admin123) as the migration note describes:

1. Boot a planet (coord + workers).
2. Upload a file via the go-sdk client (`UploadFile`).
3. Fixture: set `SystemHeaders` (e.g. `content-type: text/plain; charset=utf-8`,
   `cache-control: max-age=60`) and `UserMetadata` (e.g. `foo: bar`) on that
   file's coord DB row - reach the gorm `*Coord` DB handle (`testplanet/coord.go`)
   and update `system_headers`/`user_metadata`. (Upload path can't set them by
   decision 2.)
4. Mint a ticket (`CreateDownloadTicket`).
5. Drive the real handler over `httptest`: `GET /download?ticket=...`, then assert
   `Content-Type == text/plain; charset=utf-8`, `Cache-Control == max-age=60`,
   `X-User-Meta-Foo == bar`, and the body bytes equal the uploaded content.
6. Negative case: a file with no metadata still yields
   `Content-Type: application/octet-stream`.

Also: `make build`/`make test` in go-sdk, and `go build ./...` in depin after
the re-pin.

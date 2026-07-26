---
type: fact
tags: [storj, nodeselection, node-tags, trust, security]
created: 2026-07-21
agent: main
---

Storj storage nodes CAN self-tag: config `contact.self-signed-tags` takes
`key=value,key2=value2`, and `storagenode/contact/tags.go:GetTags` signs them with
the node's own identity, merging them with authority-signed `contact.tags` into a
`pb.SignedNodeTagSets` pushed to the satellite on each check-in.

Satellite side `satellite/contact/service.go:processNodeTags` verifies against
`append(service.nodeTagAuthority, self)` - the node itself is an accepted signee,
so self-signed tags are stored in the overlay with `Signer = <the node's own ID>`.

**Gotcha (non-obvious trust hole):** whether a self-signed tag can influence
placement depends on which consumption path a rule uses.
- `TagFilter` (`tag("<signerID>","name","value")`, `filter.go:357`) compares
  `tag.Signer == t.signer`, so a self-signed tag only matches a rule that names
  that exact node ID - useless for generic rules. Safe.
- `CreateNodeAttribute("tag:<name>")` with **no signer prefix** resolves to
  `AnyNodeTagAttribute` (`node.go:129`), which ignores the signer entirely. Any
  rule written as `node_attribute("tag:server_name")` or an `AttributeFilter` on
  a bare `tag:x` will happily consume a value a node made up about itself.
  Signer-scoped form is `tag:<signerID>/<name>`.

`trusted_node=true` is the one special-cased name and is explicitly gated on
`nodeTagAuthority.Include(signerID)`, so it cannot be self-claimed.

Repo has a longer write-up at `worker-tags.md` (untracked, repo root).

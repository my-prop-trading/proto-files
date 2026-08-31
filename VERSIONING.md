# Versioning

Every merge to `main` is tagged `vMAJOR.MINOR.PATCH`. Consumers pin a tag instead of tracking `main`,
so a merge here can no longer break anyone else's build.

## Why pinning matters

Adding a field to a message is wire-compatible but **breaks Rust compilation**: `prost` generates a
struct with all fields public, so every struct literal in every consumer stops compiling until the
new field is mapped. Without a pin, that breakage lands in unrelated branches within minutes of a
merge.

## Bump rules

The tag is produced automatically from conventional-commit subjects since the previous tag.

| Bump | Trigger | Examples |
|---|---|---|
| MAJOR | `feat!:` / `BREAKING CHANGE:` in the body | removing or renaming a field, rpc, message or enum; changing a field type or number |
| MINOR | `feat:` | new field, new rpc, new message, new enum variant |
| PATCH | anything else | comments, formatting, `reserved` on already-removed numbers |

MINOR is wire-safe but source-breaking for Rust consumers — that is expected, and is exactly what the
pin protects them from.

## Compatibility rules

- Never reuse a field number. Mark removed ones `reserved`, together with the name.
- Never change the type or number of an existing field. Add a new field and deprecate the old one.
- Removing anything is a MAJOR bump and needs every consumer migrated first.
- New fields go at the end, with the next free number.

## Consuming a version

In the consumer's `build.rs`:

```rust
const PROTO_VERSION: &str = "v1.0.0";

fn main() {
    let url = format!(
        "https://raw.githubusercontent.com/my-prop-trading/proto-files/{}/proto/",
        PROTO_VERSION
    );
    ci_utils::sync_and_build_proto_file(&url, "TraderAccountsGrpcService.proto");
}
```

A consumer that keeps `main` in the URL keeps the old behaviour — it tracks HEAD and stays exposed to
the breakage above. Pinning is opt-in and per repository.

To build against an unmerged branch (verifying mappings before the proto is merged), point
`PROTO_VERSION` at the branch name or a commit sha — `raw.githubusercontent.com` resolves all three
the same way. Restore the tag before merging the consumer.

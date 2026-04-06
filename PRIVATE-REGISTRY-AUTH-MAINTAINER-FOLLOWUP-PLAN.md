# Private Registry Auth: Maintainer Follow-up Plan

This note captures the next round of changes after the maintainer review on the
private-registry auth work.

## Summary

There are three follow-up topics:

1. `credential-alias` lookup in provider resolution
2. removing `registries.<name>.token` from config-based auth resolution
3. cleanup of the token redaction/debug implementation

The first and third are straightforward implementation work.

The token-source point is no longer really ambiguous:

- Cargo documents `registries.<name>.token` as a config key
- the same docs also say the token should only appear in the credentials file

So the runtime auth path should not treat normal Cargo config as a token source.

## Implement Now

### 1. Resolve `credential-alias` before checking for `cargo:token`

Current behavior only checks the configured provider value directly.

That misses cases like:

```toml
[registries.a]
credential-provider = "a"

[credential-alias]
a = "cargo:token"
```

Planned change:

- add provider resolution in `crates/bin/src/registry_auth.rs`
- if a provider string does not directly match `cargo:token`, look it up in
  `[credential-alias]`
- then evaluate the resolved provider
- keep scope limited to the current phase:
  detect whether the effective provider enables the `cargo:token` path

Tests to add:

- alias resolves to `cargo:token`
- alias resolves to a non-token provider
- direct provider still works without alias indirection

### 2. Clean up secret redaction/debug output

Current code uses handwritten `Debug` impls to avoid printing token values.

Conclusion after checking `zeroize`:

- `zeroize::Zeroizing<T>` zeroizes on drop, but does not provide redacted
  `Debug` output
- there does not appear to be a built-in `zeroize` helper for this
- crates like `secrecy` or `redact` exist in the ecosystem, but adding a new
  dependency just for this cleanup would be unnecessary for this PR

Planned change:

- replace the handwritten impls with a small local redacting helper/wrapper
- keep zeroizing secret storage unchanged
- ensure token-bearing structs still render without leaking the secret value

Tests to add:

- debug output for config/credentials token-bearing structs is redacted

### 3. Remove config-token fallback from auth resolution

The maintainer called out the runtime use of `registries.<name>.token` from
Cargo config, and the Cargo docs support that concern: although the key is
documented, the docs say the value should only appear in the credentials file.

Planned change:

- remove token usage from `crates/bin/src/registry_auth.rs`
- remove the parsed token field from
  `crates/binstalk-manifests/src/cargo_config.rs`
- keep token resolution limited to env + credentials files

Tests to add:

- config token is ignored by auth resolution
- env token still works
- credentials-file token still works

## Verification

After the follow-up changes:

- `cargo test -p binstalk-manifests --lib`
- `cargo test -p cargo-binstall --lib registry_auth`
- `cargo fmt --check`

Add one focused regression test to ensure env + credentials still work and
config token no longer participates in resolution.

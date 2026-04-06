# Private Registry Auth: Final Review Plan

This note captures the final review round before the private-registry auth work
is ready to land.

## Summary

The remaining feedback falls into two buckets:

1. alias resolution correctness in `registry_auth`
2. secret token type cleanup in `binstalk-manifests`

These are both follow-up refinements on top of the already-working
`cargo:token` implementation.

## 1. Alias Resolution Tightening

Files:

- `crates/bin/src/registry_auth.rs`

### Changes

- keep exact support for `cargo:token`
- treat any other built-in `cargo:*` provider name as a non-match
- do not try to resolve built-in `cargo:*` names through `[credential-alias]`
- allow single-element provider arrays to be resolved through the same alias
  path as provider strings

### Why

- built-in Cargo provider names should not be overrideable
- `[\"alias-name\"]` should behave like `\"alias-name\"` when it is a
  single-element provider array

### Tests

- alias to `cargo:token` still works
- single-element array alias to `cargo:token` works
- `cargo:token-from-stdout` and other `cargo:*` built-ins do not get resolved
  through alias lookup

## 2. Secret Type Cleanup

Files:

- `crates/binstalk-manifests/src/helpers.rs`
- `crates/binstalk-manifests/src/cargo_credentials.rs`
- `crates/bin/src/registry_auth.rs`
- `crates/binstalk-registry/src/auth.rs`

### Changes

- replace the current option-specific redaction helper with a reusable
  redacting wrapper type
- define `SecretString` in terms of that wrapper and `Zeroizing<Box<str>>`
- make `SecretString` public so it can be used across crate boundaries
- return `Option<&SecretString>` from `get_registry_token`
- stop downgrading tokens to `&str` and then re-wrapping them into
  `Zeroizing<Box<str>>` later
- thread the shared secret type through registry auth creation

### Why

- keeps the token zeroized and redacted through the whole flow
- avoids re-wrapping secrets at each boundary
- makes accidental misuse harder
- gives token-bearing types a more consistent representation

### Tests

- token-bearing debug output remains redacted
- registry auth still resolves correctly from env and credentials
- config token remains ignored

## Suggested Commit Split

### Commit 1

`fix: tighten credential alias resolution for cargo token auth`

Includes:

- built-in `cargo:*` handling cleanup
- single-element array alias handling
- alias-related tests

### Commit 2

`refactor: keep registry tokens redacted and zeroized end-to-end`

Includes:

- reusable redacting wrapper
- public `SecretString`
- `get_registry_token` returning `&SecretString`
- removal of token re-wrapping across crate boundaries
- token/debug tests

## Verification

After the final round:

- `cargo test -p binstalk-manifests --lib`
- `cargo test -p cargo-binstall --lib registry_auth`
- `cargo test -p binstalk-registry --lib --no-run`
- `cargo fmt --check`

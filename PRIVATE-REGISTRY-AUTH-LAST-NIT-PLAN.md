# Private Registry Auth: Last Nit Plan

This note captures the final cleanup requested in review:

- move `Redacted` out of `binstalk-manifests`
- move `SecretString` out of `binstalk-manifests`
- avoid making `binstalk-registry` depend on `binstalk-manifests` just for the
  secret type

## Goal

The current implementation works, but it introduced an unnecessary crate edge:

- `binstalk-registry` now depends on `binstalk-manifests`
- the only reason for that dependency is `cargo_credentials::SecretString`

The maintainer wants the shared secret/redaction types to live in
`binstalk-types` instead, so the registry crate can depend on the shared types
crate rather than the manifests crate.

## Current State

Today:

- `Redacted<T>` lives in `crates/binstalk-manifests/src/helpers.rs`
- `SecretString` is defined in
  `crates/binstalk-manifests/src/cargo_credentials.rs` as:
  `Redacted<Zeroizing<Box<str>>>`
- `binstalk-registry` imports that type from `binstalk-manifests`
- `crates/binstalk-registry/Cargo.toml` has a direct
  `binstalk-manifests` dependency only because of that

## Planned Change

### 1. Add a shared secret-types module in `binstalk-types`

Files:

- `crates/binstalk-types/src/lib.rs`
- new module, likely `crates/binstalk-types/src/secrets.rs`
- `crates/binstalk-types/Cargo.toml`

Move/add:

- `Redacted<T>`
- `SecretString = Redacted<Zeroizing<Box<str>>>`

`binstalk-types` will need a direct `zeroize` dependency.

Why:

- this makes the secret wrapper part of the common shared type layer
- both manifests and registry crates can import it without creating an awkward
  dependency edge

### 2. Update `binstalk-manifests` to use the shared secret types

Files:

- `crates/binstalk-manifests/src/cargo_credentials.rs`
- `crates/binstalk-manifests/src/helpers.rs`

Changes:

- import `SecretString` (and `Redacted` if needed) from `binstalk-types`
- remove the local `Redacted<T>` definition from `helpers.rs`
- keep credentials parsing behavior the same

Why:

- manifests should consume the shared type, not define it privately

### 3. Update `binstalk-registry` to use `binstalk-types` directly

Files:

- `crates/binstalk-registry/src/auth.rs`
- `crates/binstalk-registry/Cargo.toml`

Changes:

- import `SecretString` from `binstalk-types`
- remove the direct `binstalk-manifests` dependency if nothing else needs it

Why:

- this removes the dependency the maintainer called out
- the registry crate should not depend on the manifests crate for a generic
  shared secret type

### 4. Update `cargo-binstall` auth resolution imports

Files:

- `crates/bin/src/registry_auth.rs`

Changes:

- import `SecretString` from `binstalk-types`
- keep credentials loading from `binstalk-manifests`
- keep behavior unchanged

Why:

- the binary crate still needs both:
  - manifests for credentials parsing
  - shared types for the secret wrapper

## Expected Result

After this change:

- `Redacted<T>` lives in `binstalk-types`
- `SecretString` lives in `binstalk-types`
- `binstalk-manifests` uses the shared type
- `binstalk-registry` uses the shared type
- `binstalk-registry` no longer needs `binstalk-manifests` as a dependency

## Suggested Commit Shape

One commit is enough for this final nit:

`refactor: move shared secret types into binstalk-types`

Why one commit:

- this is one cohesive dependency-boundary cleanup
- splitting it would mostly create mechanical churn without making review easier

## Verification

After the change:

- `cargo test -p binstalk-manifests --lib`
- `cargo test -p cargo-binstall --lib registry_auth`
- `cargo test -p binstalk-registry --lib --no-run`
- `cargo fmt --check`
- verify `crates/binstalk-registry/Cargo.toml` no longer depends on
  `binstalk-manifests`

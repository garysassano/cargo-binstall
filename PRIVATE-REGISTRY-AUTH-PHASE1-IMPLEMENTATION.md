# Private Registry Authentication Phase 1 Implementation

## Purpose

This note documents what was implemented for phase 1 of private registry authentication, why each change exists, and how a reviewer can read the patch with the least amount of context switching.

The scope of phase 1 was intentionally limited to the reported failure mode:

- registry selected by Cargo name
- `cargo:token`-style auth
- token supplied by Cargo env vars or Cargo credentials file
- authenticated sparse index/bootstrap and authenticated crate downloads

Out of scope for this step:

- `cargo:token-from-stdout`
- external credential-process plugins
- OS-native credential providers
- full Cargo config file discovery/merge behavior
- non-standard auth schemes such as Shipyard's old `User-Agent` workaround

## Summary Of What Changed

The implementation has three parts:

1. Parse enough more of Cargo config and credentials to resolve phase-1 auth.
2. Preserve registry identity long enough to resolve auth correctly.
3. Attach `Authorization` only in registry flows that Cargo says require it.

This is a slightly broader change than a one-line token patch, but it is still the minimum change set that preserves the intended architecture.

## Review Order

If reviewing this patch, the lowest-friction order is:

1. `crates/bin/src/registry_auth.rs`
2. `crates/bin/src/entry.rs`
3. `crates/binstalk-manifests/src/cargo_config.rs`
4. `crates/binstalk-manifests/src/cargo_credentials.rs`
5. `crates/binstalk-registry/src/auth.rs`
6. `crates/binstalk-registry/src/sparse_registry.rs`
7. `crates/binstalk-registry/src/git_registry.rs`
8. `crates/binstalk-registry/src/common.rs`
9. `crates/binstalk/src/ops.rs`

That order follows the runtime flow from CLI setup to request execution.

## Detailed Changes

### 1. Cargo config parsing was extended

Files:

- `crates/binstalk-manifests/src/cargo_config.rs`
- `crates/binstalk-manifests/src/lib.rs`

Added parsing for:

- `registries.<name>.credential-provider`
- `registry.credential-provider`
- `registry.global-credential-providers`
- `[credential-alias]`
- optional token fields already used by Cargo config shapes

Also added:

- `Config::get_registry(name)` so the resolved registry entry can be retrieved after `replace-with` resolution

Why this was necessary:

- phase 1 needs to know whether `cargo:token` is actually active
- `CARGO_REGISTRIES_<NAME>_TOKEN` should not be treated as globally valid if the configured provider path does not include `cargo:token`

### 2. Cargo credentials file support was added

File:

- `crates/binstalk-manifests/src/cargo_credentials.rs`

Added parsing for:

- `$CARGO_HOME/credentials.toml`
- legacy `$CARGO_HOME/credentials`

Why this was necessary:

- `cargo login` stores tokens there
- phase 1 would be incomplete if it only supported env vars

### 3. A small CLI-side registry auth resolver was added

Files:

- `crates/bin/src/registry_auth.rs`
- `crates/bin/src/lib.rs`

This module does phase-1 auth resolution only. It currently:

- normalizes Cargo registry env var names
- checks whether `cargo:token` is enabled for the target registry
- reads registry token from:
  - `CARGO_REGISTRIES_<NAME>_TOKEN`
  - Cargo credentials file
  - config token field if present
- supports crates.io token lookup via `CARGO_REGISTRY_TOKEN` and `[registry]`

It intentionally does not yet:

- spawn external providers
- resolve aliases into executable commands
- support `cargo:token-from-stdout`

Why this lives in `crates/bin`:

- this is where CLI args, Cargo config, env vars, and cargo home are already available
- it avoids leaking Cargo-specific configuration logic into the downloader

### 4. Registry selection now preserves auth information

Files:

- `crates/bin/src/entry.rs`
- `crates/bin/src/initialise.rs`
- `crates/binstalk/src/ops.rs`

What changed:

- `initialise` now preserves `cargo_home`
- `entry.rs` resolves a `ResolvedRegistry` instead of a bare `Registry`
- `Options.registry` now carries that richer object

Why this was necessary:

- a plain index URL is not enough to resolve registry auth correctly
- registry name must still be available when deciding whether `cargo:token` applies

### 5. A small registry wrapper was introduced

Files:

- `crates/binstalk-registry/src/auth.rs`
- `crates/binstalk-registry/src/lib.rs`

Added:

- `RegistryAuth` to hold the resolved token
- `ResolvedRegistry` to pair a `Registry` with optional auth
- a new internal method to fetch crate metadata with optional auth context

Why this exists:

- it keeps auth attached to a registry, not to the global HTTP client
- it allows `binstalk` to keep using a single registry field without a wider API rewrite

### 6. Registry config parsing now reads `auth-required`

Files:

- `crates/binstalk-registry/src/common.rs`
- `crates/binstalk-registry/src/git_registry.rs`
- `crates/binstalk-registry/src/sparse_registry.rs`

Added parsing for:

- `auth-required` in registry `config.json`

Why this was necessary:

- Cargo uses `auth-required` as the signal that HTTP downloads and sparse index requests should include `Authorization`

### 7. Sparse registry auth flow was implemented

File:

- `crates/binstalk-registry/src/sparse_registry.rs`

Behavior now:

1. request `config.json` without auth
2. if response is `401`, retry with resolved auth if available
3. parse `auth-required`
4. if `auth-required` is true, require auth for sparse index reads and downloads

Why this shape:

- it matches Cargo's documented sparse bootstrap behavior
- it avoids sending tokens to anonymous registries unnecessarily

### 8. Git registry download auth was implemented

File:

- `crates/binstalk-registry/src/git_registry.rs`

Behavior now:

- git index cloning is unchanged
- `config.json` read from the git index now captures `auth-required`
- crate downloads include `Authorization` when `auth-required` is true

Why:

- the current issue is about HTTP crate download auth
- git index auth itself is a separate concern and did not need to be changed for phase 1

### 9. Download request auth attachment was kept local

Files:

- `crates/binstalk-registry/src/common.rs`
- `crates/binstalk-registry/src/crates_io_registry.rs`

What changed:

- manifest download now accepts optional auth and builds the request explicitly
- a small `apply_auth` helper adds the `Authorization` header

Why this was done here instead of in the downloader:

- the downloader should stay generic
- whether auth is required is a registry concern, not a transport concern

## Was This The Minimum Change Possible?

### Short answer

Yes, if the constraint is:

- fix the feature in a way that matches the recommended architecture
- keep the code reviewable
- do not knowingly hardcode the wrong abstraction

### What would have been smaller

A smaller patch was possible if the goal were only:

- detect `CARGO_REGISTRIES_<NAME>_TOKEN`
- attach it to all requests for non-crates.io registries

That would have been fewer lines, but it would have been a poor implementation because:

- it would ignore whether `cargo:token` is actually enabled
- it would spread auth behavior across unrelated request paths
- it would not support credentials file tokens
- it would not use `auth-required`
- it would make phase 2 harder, not easier

So that version would have been smaller by diff size, but larger in technical debt.

### How this patch was kept minimal anyway

The implementation was intentionally constrained in several ways:

- only `cargo:token` is supported in code
- no subprocess-based providers yet
- no Cargo config merge rewrite
- no downloader-wide auth refactor
- no change to git clone auth
- no new public CLI flags
- no attempt to solve every private-registry edge case at once

In other words, the patch is architecturally correct, but feature-scoped.

## Why A Human Reviewer Should Not Be Over-Burdened

The change touches multiple crates, but each crate change is narrow:

- `binstalk-manifests`: parse more Cargo data
- `crates/bin`: resolve phase-1 auth once
- `binstalk-registry`: apply auth in registry-specific request paths
- `binstalk`: type plumbing only

There is no large-scale refactor, no downloader redesign, and no change to unrelated installation logic.

The reviewer should be able to evaluate the patch by asking only these questions:

1. Is `cargo:token` detection correct?
2. Is token lookup precedence reasonable for phase 1?
3. Is auth only attached where Cargo semantics require it?
4. Are public registries unchanged?
5. Is the phase-2 path still open?

That is the real measure of whether the patch is reviewable, not just raw line count.

## Verification Performed

Used for local verification:

- `cargo test -p binstalk-manifests --lib`
- `cargo test -p cargo-binstall --lib registry_auth`
- `cargo test -p binstalk-registry --lib --no-run`
- `cargo fmt --check`

Notes:

- `binstalk-registry` has existing live-network tests, so `--no-run` was used there to validate compilation of the changed auth paths without relying on external network timing.

## Remaining Phase 2 Work

Still not implemented:

- `cargo:token-from-stdout`
- external credential-process plugins
- alias expansion into provider commands
- OS-backed providers
- end-to-end private-registry test fixture

That work should build on this patch without needing to restructure the phase-1 design.

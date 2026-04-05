# Private Registry Auth Review Plan

## Purpose

This note captures the follow-up plan based on review feedback on the private registry authentication PR.

The goal is to separate:

- changes that are clearly correct and should be implemented now
- changes that would widen the PR scope and should wait for reviewer confirmation

This keeps the patch moving without accidentally turning a focused `cargo:token` fix into a broader credential-provider rewrite.

## Review Feedback Summary

The main themes from review so far are:

1. secrets should be zeroized earlier
2. env/config loading order should be corrected
3. a few small implementation details should be cleaned up
4. some behaviors in the current PR may overstate support for provider configuration beyond the intended `cargo:token` scope

## Changes To Implement Now

These are straightforward fixes that improve correctness, safety, or code quality without meaningfully expanding scope.

### 1. Zeroize parsed token fields

Files likely affected:

- `crates/binstalk-manifests/src/cargo_config.rs`
- `crates/binstalk-manifests/src/cargo_credentials.rs`
- tests that deserialize these types

Current issue:

- token values are parsed into regular string containers first
- they only become `Zeroizing` later during auth resolution

Planned fix:

- change token-bearing fields to a zeroizing string type at parse/storage time
- keep the secret in a zeroizing container for as much of its lifetime as possible

Why this should be done now:

- this is a real security improvement
- the reviewer is correct
- it does not change the feature scope

### 2. Fix credential-provider env precedence

File likely affected:

- `crates/bin/src/registry_auth.rs`

Current issue:

- the current logic can skip the crates.io-specific env precedence path depending on how config lookup returns early

Planned fix:

- make the precedence explicit and ordered as:
  1. `CARGO_REGISTRIES_<name>_CREDENTIAL_PROVIDER`
  2. if applicable, `CARGO_REGISTRY_CREDENTIAL_PROVIDER`
  3. config lookup

Why this should be done now:

- this is a correctness bug in precedence
- it stays within the existing phase-1 model

### 3. Hoist repeated suffix construction out of env scans

File likely affected:

- `crates/bin/src/registry_auth.rs`

Current issue:

- `format!("_{suffix}")` is being rebuilt inside env iteration

Planned fix:

- compute the suffix once before the loop and reuse it

Why this should be done now:

- tiny cleanup
- no behavioral risk
- directly addresses review feedback

### 4. Replace `exists()` with `NotFound` handling

File likely affected:

- `crates/binstalk-manifests/src/cargo_credentials.rs`

Current issue:

- the code probes `credentials.toml` with `exists()` before opening it

Planned fix:

- attempt the open/load directly
- branch on `io::ErrorKind::NotFound`

Why this should be done now:

- cleaner and more race-safe
- small change
- directly addresses review feedback

### 5. Remove or narrow crates.io token/provider special-casing

File likely affected:

- `crates/bin/src/registry_auth.rs`

Current issue:

- the PR currently includes crates.io-specific token/provider handling even though this install/read path does not really need crates.io auth

Planned fix:

- remove the special-casing, or narrow it enough that the PR remains focused on named private registries

Why this should be done now:

- it simplifies the patch
- it makes the scope more honest
- it responds to a valid reviewer concern

## Changes That Should Wait For Reviewer Confirmation

These are reasonable concerns, but implementing them immediately would widen the PR in ways that should be explicitly agreed on first.

### 1. `credential_alias` lookup

Files likely affected:

- `crates/bin/src/registry_auth.rs`
- possibly `crates/binstalk-manifests/src/cargo_config.rs`

Current issue:

- the PR parses `credential_alias`
- but phase 1 only answers a narrow question: whether the configured provider resolves to the `cargo:token` path
- aliases are not actually evaluated into provider commands

Why this should wait:

- real alias support pushes the PR toward broader credential-provider semantics
- that is beyond the original phase-1 `cargo:token` goal

Decision needed from reviewer:

- keep this PR limited to `cargo:token`, and document aliases as out of scope
- or expand this PR to partially resolve aliases

### 2. Unsupported-provider errors

Current issue:

- today, unsupported providers are effectively treated as "not the `cargo:token` path"
- the reviewer suggested surfacing an explicit unsupported-authentication error instead

Why this should wait:

- returning a hard error changes the behavioral contract of the PR
- this is a policy decision, not just a cleanup

Decision needed from reviewer:

- should this PR remain permissive and only enable `cargo:token`
- or should it start failing explicitly on unsupported provider configurations

### 3. Git-index auth beyond the final HTTP fetch

Current issue:

- the PR intentionally leaves git clone/fetch auth to the underlying git transport
- it only adds auth handling for the final HTTP download path

Why this should wait:

- expanding into git transport auth would materially broaden the PR
- it is separate from the specific token-auth regression this patch set already fixes

Decision needed from reviewer:

- is the current separation acceptable for this PR
- or should full git-index auth behavior be treated as in scope now

## Recommended Execution Order

To keep the patch reviewable:

1. zeroize token fields
2. fix provider env precedence
3. remove/narrow crates.io special-casing
4. apply small cleanup nits
5. wait for reviewer answer on alias / unsupported-provider behavior

## Recommended Review Posture

The clearest posture for the next revision is:

- accept the correctness and security fixes immediately
- avoid silently expanding the PR into generic provider support
- ask explicitly whether alias evaluation / unsupported-provider errors should be in scope for this PR or follow-up work

That keeps the branch moving while staying honest about what phase 1 is meant to cover.

## Current State

`cargo-binstall` now supports Cargo's built-in `cargo:token-from-stdout` provider for private registries.

This includes:

- provider selection for `cargo:token-from-stdout`
- `credential-alias` resolution
- registry-specific env/config precedence
- `global-credential-providers`, with later entries winning
- built-in `cargo:*` names treated as built-ins rather than alias targets
- `{index_url}` substitution in provider arguments
- setting:
  - `CARGO`
  - `CARGO_REGISTRY_INDEX_URL`
  - `CARGO_REGISTRY_NAME_OPT`
- reading the token from stdout and converting it into `SecretString`
- preserving `--registry` / `--index` during `cargo install` fallback

## Review Follow-Up Status

Against `PRIVATE-REGISTRY-TOKEN-FROM-STDOUT-REVIEW-PLAN.md`:

### 1. Document `{index_url}` substitution source

Implemented.

- `crates/bin/src/registry_auth.rs` now carries a short comment explaining that Cargo's `BasicProcessCredential` performs this substitution.

### 2. Set `CARGO` explicitly for the subprocess

Implemented in spirit, with one clarification.

- The subprocess now always receives `CARGO`.
- The implementation uses:
  - the existing `CARGO` environment variable when present
  - otherwise a plain `"cargo"` fallback

This differs from the original plan wording "`current executable path if available`", because the current executable may be `cargo-binstall`, not Cargo itself. The current implementation is the more correct behavior.

### 3. Tighten token handling around stdout buffering

Implemented.

- The temporary stdout buffer is now `Zeroizing<String>`.
- The final result is still converted into `SecretString`.

### 4. Treat missing piped stdout as an internal invariant

Implemented.

- `child.stdout.take()` now uses `unwrap()`.

## Real Validation

Validated end-to-end against AWS CodeArtifact using:

- plain `cargo install --registry private-test ...`
- `cargo-binstall --registry private-test ...`
- `cargo-binstall --force` fallback

Observed result:

- plain Cargo succeeded with `cargo:token-from-stdout`
- `cargo-binstall` resolved the crate from CodeArtifact without `401 Unauthorized`
- fallback preserved `--registry private-test`
- the fallback install completed successfully

## What Comes Next

The next logical phase is support for generic external credential-process providers beyond Cargo's built-in `cargo:token-from-stdout`.

Recommended order:

1. Generic external credential-process providers using Cargo's protocol
2. OS-native built-in Cargo providers such as:
   - `cargo:libsecret`
   - `cargo:macos-keychain`
   - `cargo:wincred`

Why this order:

- generic protocol support covers vendor-specific and internal auth helpers with one mechanism
- OS-native built-ins are still useful, but are narrower and platform-specific

## Remaining Non-Goals For This PR

- generic JSON-based credential-process providers
- OS-native built-in providers
- any new auth handling for arbitrary private binary asset URLs outside the existing registry flow

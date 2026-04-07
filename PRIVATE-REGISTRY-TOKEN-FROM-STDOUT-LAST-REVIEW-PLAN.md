## Goal

Address the latest maintainer feedback on the `cargo:token-from-stdout` implementation with one small follow-up focused on correctness and local cleanup.

## Review Items

### 1. Reuse `seen_aliases` in global provider resolution

Problem:
- `resolve_global_supported_provider` already reuses one `Vec`, but the reviewer also wants the invariant called out explicitly.

Plan:
- Add `debug_assert!(seen_aliases.is_empty())` before the `find_map`.

Expected result:
- Clearer invariant around alias tracking in the global-provider scan.

### 2. Reuse the computed registry index argument

Problem:
- `registry.cargo_install_index_arg()` is currently called more than once in the subprocess setup.

Plan:
- Compute the index argument once and reuse it for:
  - `{index_url}` substitution
  - `CARGO_REGISTRY_INDEX_URL`

Expected result:
- Simpler code and avoids repeated allocation.

### 3. Use `SecretString` directly for the temporary token buffer

Problem:
- The reviewer suggested using `SecretString` directly instead of a separate zeroizing `String`.

Plan:
- Add `DerefMut` for `Redacted<T>` in `binstalk-types`.
- Use `SecretString` as the mutable buffer in the stdout-read path.

Expected result:
- The token stays in the shared secret type during processing.

### 4. Fix Windows newline handling

Problem:
- The current code only searches for `\n`, which can mis-handle `\r\n`.

Plan:
- Change the line handling so the first line is extracted correctly for both:
  - `\n`
  - `\r\n`
- Continue rejecting output with more than one line.

Expected result:
- Correct token parsing on Windows.

### 5. Clean up format strings

Problem:
- Two error messages still use positional formatting for `executable`.

Plan:
- Rewrite them with inline `{executable}` formatting.

Expected result:
- Slightly cleaner error formatting with no behavior change.

## Commit Shape

Single commit:

- `fix: polish token-from-stdout provider handling`

This is one small follow-up on the same code path, so splitting it further would mostly create review noise.

## Verification

```bash
cargo fmt --check
cargo test -p cargo-binstall --lib registry_auth
cargo clippy -p cargo-binstall --lib --no-deps -- -D clippy::all
```

## Goal

Address the four review comments on the `cargo:token-from-stdout` PR with the smallest follow-up that improves Cargo compatibility and keeps the implementation maintainable.

## Review Items

### 1. Document `{index_url}` substitution source

Problem:
- The current implementation replaces `{index_url}` in `cargo:token-from-stdout` command arguments, but the reviewer did not find this in the Cargo Book.

Plan:
- Add a short code comment in `crates/bin/src/registry_auth.rs` pointing to Cargo's implementation as the source of this behavior.
- Use Cargo source as the reference, since this behavior appears in the implementation rather than the Book docs.

Expected result:
- The behavior remains unchanged.
- The code clearly explains why `{index_url}` substitution exists.

### 2. Set `CARGO` explicitly for the subprocess

Problem:
- Cargo documents that the credential-provider subprocess receives `CARGO`, but `cargo-binstall` may be invoked directly, so that variable may not already exist in the environment.

Plan:
- Set `CARGO` explicitly on the `cargo:token-from-stdout` subprocess.
- Use the current executable path if available.

Expected result:
- The subprocess environment matches Cargo more closely even when `cargo-binstall` is invoked directly.

### 3. Tighten token handling around stdout buffering

Problem:
- The token currently lives in a plain `String` during the read/trim step before being converted into `SecretString`.
- The reviewer is correct on the goal, but converting to `SecretString` immediately at the suggested line is awkward because trimming still needs to happen first.

Plan:
- Keep the final output type as `SecretString`.
- Change the intermediate buffer handling so the token does not sit in a plain non-zeroizing `String` longer than necessary.
- Use a zeroizing intermediate before the final `SecretString` conversion.

Expected result:
- Preserve the current trimming logic.
- Reduce the time the token spends in a plain `String`.

### 4. Treat missing piped stdout as an internal invariant

Problem:
- The code currently returns an `io::Error` if `child.stdout.take()` is `None`, but this should be impossible once `stdout(Stdio::piped())` is set.

Plan:
- Replace the defensive branch with `unwrap()`.

Expected result:
- Simpler code that reflects the actual invariant.

## Recommended Commit Shape

Single commit:

- `fix: tighten token-from-stdout provider compatibility`

This is one cohesive follow-up on the same implementation, and splitting it further would mostly create review noise.

## Verification

```bash
cargo fmt --check
cargo test -p cargo-binstall --lib registry_auth
cargo clippy -p cargo-binstall --lib --no-deps -- -D clippy::all
```

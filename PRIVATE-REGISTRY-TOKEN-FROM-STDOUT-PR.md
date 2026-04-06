Addresses #2312

## Summary

This PR adds phase-2 private registry authentication support to `cargo-binstall`.

In this PR, "phase 2" means extending the existing private-registry auth support from `cargo:token` to Cargo's built-in `cargo:token-from-stdout` provider, without yet taking on the full external credential-process surface area.

Concretely, phase 2 includes:

- `cargo:token-from-stdout`
- `credential-alias` resolution for `cargo:token-from-stdout`
- Cargo-compatible provider selection and precedence
- preserving `--registry` / `--index` during `cargo install` fallback

Phase 2 still does not include:

- generic external credential-process providers using Cargo's JSON protocol
- OS-native Cargo credential providers

The main goal is to make private registry flows work in the same cases where `cargo install` already works with `cargo:token-from-stdout`, especially for registries like AWS CodeArtifact that mint short-lived tokens on demand through an external command.

## What Changed

### `cargo:token-from-stdout` support

- Refactor private-registry credential selection from a `cargo:token` boolean check into explicit provider resolution
- Add support for Cargo's built-in `cargo:token-from-stdout` provider
- Support both provider forms Cargo accepts:
  - space-separated string values
  - array values
- Resolve `credential-alias` before deciding whether the effective provider enables the token path
- Respect Cargo's precedence rules:
  - registry-specific env override
  - registry-specific config
  - `registry.global-credential-providers`, with later entries winning
- Treat built-in `cargo:*` provider names as built-ins, not alias targets

### Provider execution behavior

- Execute the configured command and read the token from stdout
- Match Cargo's single-line token behavior
- Replace `{index_url}` in subprocess arguments
- Set:
  - `CARGO_REGISTRY_INDEX_URL`
  - `CARGO_REGISTRY_NAME_OPT`

### Fallback behavior

- Keep the existing fallback fix from phase 1: when `cargo-binstall` falls back to `cargo install`, it preserves the selected registry or index instead of silently defaulting back to crates.io

## Real Validation

Validated manually against AWS CodeArtifact using a sparse private registry configured with `credential-provider = "cargo:token-from-stdout ..."` and no pre-stored login requirement.

Validation flow:

1. Configure a CodeArtifact registry with `cargo:token-from-stdout`
2. Verify plain `cargo install --registry private-test ...`
3. Verify `cargo-binstall --registry private-test ...`
4. Verify `cargo-binstall --force` falls back using the same private registry

Observed result:

- plain Cargo succeeded with `cargo:token-from-stdout`
- `cargo-binstall` resolved the crate from CodeArtifact without `401 Unauthorized`
- fallback to `cargo install` preserved `--registry private-test`
- the fallback install completed successfully

## Test Commands

```bash
cargo test -p cargo-binstall --lib registry_auth
cargo fmt --check
```

Manual CodeArtifact smoke test:

```bash
cargo uninstall codeartifact-smoke || true
cargo install --registry private-test codeartifact-smoke --version 0.1.0
cargo run -p cargo-binstall -- --registry private-test codeartifact-smoke@0.1.0 -y --force
```

Example registry configuration used for validation:

```toml
[registries.private-test]
index = "sparse+https://<domain>-<account>.d.codeartifact.<region>.amazonaws.com/cargo/<repo>/"
credential-provider = [
  "cargo:token-from-stdout",
  "aws", "codeartifact", "get-authorization-token",
  "--domain", "<domain>",
  "--domain-owner", "<account>",
  "--region", "<region>",
  "--query", "authorizationToken",
  "--output", "text",
]
```

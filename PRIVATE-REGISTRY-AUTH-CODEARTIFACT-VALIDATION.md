# Private Registry Auth CodeArtifact Validation

## Purpose

This note records the manual validation that was run against AWS CodeArtifact after implementing phase 1 private registry authentication, plus the small follow-up fix for source fallback.

This is not just a design note. It documents:

- what was tested
- what failed first
- what was fixed
- what CodeArtifact auth modes exist
- which auth mode phase 1 supports

## CodeArtifact Auth Modes Relevant Here

AWS documents Cargo integration with CodeArtifact using a Cargo registry entry like:

```toml
[registries.private-test]
index = "sparse+https://private-ACCOUNT_ID.d.codeartifact.REGION.amazonaws.com/cargo/REPOSITORY/"
credential-provider = "..."
```

For this feature, there are two important authentication modes.

### 1. `cargo:token`

Example:

```toml
[registries.private-test]
index = "sparse+https://private-491085427433.d.codeartifact.eu-central-1.amazonaws.com/cargo/test/"
credential-provider = "cargo:token"
```

How it works:

- Cargo reads the registry token from:
  - `CARGO_REGISTRIES_<NAME>_TOKEN`
  - Cargo credentials file created by `cargo login`
  - related Cargo token config

This is the auth mode implemented in phase 1.

### 2. `cargo:token-from-stdout`

Example from the AWS guide:

```toml
[registries.private-test]
index = "sparse+https://private-491085427433.d.codeartifact.eu-central-1.amazonaws.com/cargo/test/"
credential-provider = "cargo:token-from-stdout aws codeartifact get-authorization-token --domain private --domain-owner 491085427433 --region eu-central-1 --query authorizationToken --output text"
```

How it works:

- Cargo spawns the configured command
- reads the token from stdout
- this is useful for CodeArtifact because tokens expire after 12 hours

This is **not** implemented in phase 1.

## Which Auth Mode Was Used For Validation

Validation used `cargo:token`, not `cargo:token-from-stdout`.

That was intentional because phase 1 only implements `cargo:token`.

The local Cargo config used for testing was effectively:

```toml
[registries.private-test]
index = "sparse+https://private-491085427433.d.codeartifact.eu-central-1.amazonaws.com/cargo/test/"
credential-provider = "cargo:token"
```

Notably, the test did **not** depend on:

- `[registry].default`
- `[source.crates-io].replace-with`

Those were avoided during the smoke test so the behavior under test stayed isolated.

## Manual Validation Steps

### 1. Verify CodeArtifact access

The repository endpoint was successfully retrieved:

```bash
aws codeartifact get-repository-endpoint \
  --domain private \
  --domain-owner 491085427433 \
  --repository test \
  --format cargo \
  --region eu-central-1 \
  --query repositoryEndpoint \
  --output text
```

This returned:

```text
https://private-491085427433.d.codeartifact.eu-central-1.amazonaws.com/cargo/test/
```

### 2. Get a CodeArtifact auth token

```bash
aws codeartifact get-authorization-token \
  --domain private \
  --domain-owner 491085427433 \
  --region eu-central-1 \
  --query authorizationToken \
  --output text
```

### 3. Store the token using Cargo's `cargo:token` flow

```bash
aws codeartifact get-authorization-token \
  --domain private \
  --domain-owner 491085427433 \
  --region eu-central-1 \
  --query authorizationToken \
  --output text | cargo login --registry private-test
```

This succeeded and stored the token in Cargo credentials.

### 4. Publish a test crate

A tiny crate named `codeartifact-smoke` was created and published:

```bash
cargo new --bin codeartifact-smoke
cd codeartifact-smoke
cargo publish --registry private-test --allow-dirty
```

This succeeded.

### 5. Verify plain Cargo install

```bash
cargo install --registry private-test codeartifact-smoke --version 0.1.0
```

This succeeded.

That established the baseline that:

- the registry endpoint was correct
- Cargo auth with `cargo:token` worked
- the crate was actually available from CodeArtifact

## What Happened With `cargo-binstall`

### First validation

Running:

```bash
cargo run -p cargo-binstall -- --registry private-test codeartifact-smoke@0.1.0 -y
```

reported that the crate was already installed.

This was already a useful signal because it meant:

- private registry resolution worked
- there was no `401 Unauthorized`
- `cargo-binstall` could inspect the private crate successfully

### Stronger validation with `--force`

Running:

```bash
cargo run -p cargo-binstall -- --registry private-test codeartifact-smoke@0.1.0 -y --force
```

initially exposed a second bug:

- `cargo-binstall` resolved the crate from the private registry correctly
- but when it fell back to `cargo install`, the fallback command did **not** preserve `--registry private-test`
- Cargo therefore defaulted to `crates.io`

Observed failure:

```text
error: could not find `codeartifact-smoke` in registry `crates-io` with version `=0.1.0`
```

This was **not** an auth failure.

It proved:

- phase 1 auth was already working
- fallback-to-source lost the selected registry context

## Follow-Up Fix Implemented

The follow-up fix preserved the original registry selection for source fallback.

### Behavior after the fix

If `cargo-binstall` resolved with:

- `--registry <name>` then fallback now uses `cargo install --registry <name>`
- `--index <url>` then fallback now uses `cargo install --index <url>`

### Why this fix was small

The patch was intentionally limited to:

- storing fallback registry/index selection in `Options`
- passing those values into fallback command construction
- adding focused tests for command argument propagation

No broader refactor was needed.

## Final Successful Validation

After the fallback fix, rerunning:

```bash
cargo run -p cargo-binstall -- --registry private-test codeartifact-smoke@0.1.0 -y --force
```

succeeded.

Important output:

```text
WARN The package codeartifact-smoke v0.1.0 will be installed from source (with cargo)
Updating `private-test` index
Installing codeartifact-smoke v0.1.0 (registry `private-test`)
...
INFO Cargo finished successfully
```

This confirms:

1. `cargo-binstall` authenticated successfully to CodeArtifact using phase-1 `cargo:token` support.
2. It resolved the crate from the private registry.
3. It did not hit the original `401 Unauthorized` failure.
4. Fallback to `cargo install` preserved the private registry selection and completed successfully.

## What Was Proven

The CodeArtifact validation proved all of the following:

- phase 1 auth support for `cargo:token` works in a real private registry
- tokens stored via `cargo login` are honored
- sparse registry bootstrap with auth works
- private crate resolution works
- source fallback now respects the selected private registry

## What Was Not Proven

This validation did **not** cover:

- `cargo:token-from-stdout`
- external credential-process plugins
- OS-native Cargo credential providers
- automatic token refresh after CodeArtifact token expiry
- crate downloads that require actual binary artifact fetches through `[package.metadata.binstall]`

Those remain phase 2 or later work.

## Practical Guidance For Review

If the question is "does phase 1 work against a real private registry?", the answer is yes for the `cargo:token` mode.

If the question is "does this implement the AWS-recommended `cargo:token-from-stdout` configuration?", the answer is no, not yet.

That distinction is important:

- supported now: `cargo:token`
- not supported yet: `cargo:token-from-stdout`

## Useful Commands From The Validation

Get endpoint:

```bash
aws codeartifact get-repository-endpoint \
  --domain private \
  --domain-owner 491085427433 \
  --repository test \
  --format cargo \
  --region eu-central-1 \
  --query repositoryEndpoint \
  --output text
```

Get token:

```bash
aws codeartifact get-authorization-token \
  --domain private \
  --domain-owner 491085427433 \
  --region eu-central-1 \
  --query authorizationToken \
  --output text
```

Store token with Cargo:

```bash
aws codeartifact get-authorization-token \
  --domain private \
  --domain-owner 491085427433 \
  --region eu-central-1 \
  --query authorizationToken \
  --output text | cargo login --registry private-test
```

Publish test crate:

```bash
cargo publish --registry private-test --allow-dirty
```

Test Cargo:

```bash
cargo install --registry private-test codeartifact-smoke --version 0.1.0
```

Test `cargo-binstall`:

```bash
cargo run -p cargo-binstall -- --registry private-test codeartifact-smoke@0.1.0 -y --force
```

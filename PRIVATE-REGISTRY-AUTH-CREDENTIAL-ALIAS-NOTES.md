# Private Registry Auth: Credential Alias Notes

This note captures the Cargo credential-provider details that came up during
review, especially around `credential-alias` and AWS CodeArtifact examples.

## What a credential alias is

A credential alias is a named shortcut for a credential provider definition in
Cargo config.

Instead of repeating a long provider value directly in every registry entry,
Cargo allows defining it once under `[credential-alias]` and referencing it by
name from `credential-provider`.

Example:

```toml
[registries.private]
credential-provider = "my-provider"

[credential-alias]
my-provider = ["cargo-credential-example", "--account", "prod"]
```

Here:

- `my-provider` is the alias name
- Cargo expands it to the actual provider definition when credentials are needed

## Why alias resolution matters

A registry can indirectly enable `cargo:token` through an alias:

```toml
[registries.a]
credential-provider = "a"

[credential-alias]
a = "cargo:token"
```

If code only checks the literal `credential-provider` value and does not resolve
aliases first, it will miss valid token-based registry configurations.

That is why the follow-up fix resolves aliases before deciding whether the
effective provider enables the `cargo:token` path.

## What `cargo:token-from-stdout` means

Cargo decides the provider mode from the provider value after alias expansion.

If the effective provider starts with `cargo:token-from-stdout`, Cargo knows:

- this is the `token-from-stdout` built-in mode
- the remaining command should be executed
- stdout from that command should be treated as the token

Example:

```toml
[registries.private-test]
credential-provider = [
  "cargo:token-from-stdout",
  "aws", "codeartifact", "get-authorization-token",
  "--domain", "private",
  "--domain-owner", "123456789012",
  "--region", "eu-central-1",
  "--query", "authorizationToken",
  "--output", "text",
]
```

The first element, `cargo:token-from-stdout`, is the signal that determines how
the rest of the provider value is interpreted.

## CodeArtifact and aliases

An alias makes sense for CodeArtifact when the provider command is long and
reused.

Example:

```toml
[registries.private-test]
index = "sparse+https://private-123456789012.d.codeartifact.eu-central-1.amazonaws.com/cargo/test/"
credential-provider = "codeartifact"

[credential-alias]
codeartifact = [
  "cargo:token-from-stdout",
  "aws", "codeartifact", "get-authorization-token",
  "--domain", "private",
  "--domain-owner", "123456789012",
  "--region", "eu-central-1",
  "--query", "authorizationToken",
  "--output", "text",
]
```

Why this is useful:

- the provider command is long
- it may be reused across multiple registries
- registry entries stay shorter and easier to read

## Current implementation scope

The current private-registry auth work supports:

- `cargo:token`
- aliases that resolve to `cargo:token`

It does **not** yet support:

- `cargo:token-from-stdout`
- external credential-process protocol providers beyond the current scope

So for AWS CodeArtifact today:

- phase 1 works with `cargo:token`
- a future phase would be needed for the full `cargo:token-from-stdout` flow

## Important correction

`cargo-credential-shipyard` was mentioned earlier as an example provider
binary. That should be treated only as an illustrative placeholder, not as a
documented standard Cargo tool.

The documented pieces are:

- Cargo registry authentication
- Cargo credential-provider protocol
- Cargo config support for `credential-alias`

Any actual external provider binary depends on the registry/vendor setup.

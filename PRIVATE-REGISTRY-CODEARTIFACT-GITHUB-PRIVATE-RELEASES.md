# CodeArtifact Registry + Private GitHub Releases

This note documents the most realistic direct-binary installation workflow for
private registries after adding support for:

- `cargo:token`
- `cargo:token-from-stdout`

## Scenario

The crate is published to a private Cargo registry such as AWS CodeArtifact,
but the prebuilt binaries are hosted in a private GitHub repository's Releases.

That means there are two separate auth paths:

1. Cargo registry auth
   - used to resolve crate metadata from the private registry
   - for CodeArtifact, this can be configured with:
     - `cargo:token`
     - `cargo:token-from-stdout`

2. GitHub release asset auth
   - used to download the prebuilt binary archive from a private GitHub repo
   - this is handled by `cargo-binstall`'s existing GitHub token support

## Why this workflow makes sense

Using CodeArtifact for the Cargo registry works well for private crate metadata
and source downloads, but it is not the best target for testing direct binary
downloads via `pkg-url`.

Direct binary installs in `cargo-binstall` depend on the crate's
`[package.metadata.binstall]` fields, especially:

- `pkg-url`
- `pkg-fmt`
- `bin-dir`

If `pkg-url` points to a private GitHub Releases asset URL, `cargo-binstall`
already has a dedicated GitHub-authenticated download path for that case.

That makes the combination:

- CodeArtifact for the private Cargo registry
- private GitHub Releases for the binary assets

more realistic and easier to validate than trying to use CodeArtifact itself as
both the Cargo registry and the binary asset host.

## Expected behavior

If a crate in CodeArtifact includes valid binstall metadata that points to a
private GitHub Releases asset URL, then `cargo-binstall` should be able to:

1. authenticate to CodeArtifact and resolve the crate metadata
2. read `[package.metadata.binstall]` from that crate
3. authenticate to GitHub using a GitHub token
4. download the release asset directly
5. install the binary without falling back to `cargo install`

If the crate does not have usable binstall metadata, or the GitHub asset is not
accessible, `cargo-binstall` will fall back to `cargo install`.

## Auth requirements

### CodeArtifact

The private Cargo registry auth can use either:

- `cargo:token`
- `cargo:token-from-stdout`

Example `cargo:token-from-stdout` setup:

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

### GitHub

The binary asset download from a private GitHub Releases URL requires a GitHub
token with access to the private repository.

That can come from:

- `--github-token`
- `GH_TOKEN`
- discovered credentials such as `gh auth token`

## Crate metadata example

The crate published in CodeArtifact would need binstall metadata like:

```toml
[package.metadata.binstall]
pkg-url = "https://github.com/<org>/<private-repo>/releases/download/v{ version }/{ name }-{ target }.tar.gz"
pkg-fmt = "tgz"
bin-dir = "{ bin }{ binary-ext }"
```

In this setup:

- the crate metadata is read from CodeArtifact
- the binary archive itself is downloaded from GitHub Releases

## What this tests

This workflow is the best end-to-end validation for "private registry + direct
binary install" because it exercises:

- private Cargo registry auth
- binstall metadata resolution from a private registry
- private GitHub Releases asset download
- direct binary install without source fallback

## What it does not test

This does not test:

- generic external credential-process providers
- OS-native Cargo credential providers
- arbitrary authenticated `pkg-url` hosts outside the GitHub-specific path

## Practical conclusion

For direct binary install testing, the most realistic setup is:

- CodeArtifact as the private Cargo registry
- private GitHub Releases as the binary asset host

For source-install fallback testing, CodeArtifact alone is sufficient.

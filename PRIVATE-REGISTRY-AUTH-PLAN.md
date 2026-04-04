# Private Registry Authentication Plan

## Goal

Make `cargo-binstall` authenticate to private Cargo registries in the same situations where `cargo install` already works, without regressing public registry behavior.

The immediate failure to address is this:

- a private registry is configured in Cargo
- the user or CI provides a token via Cargo-supported configuration such as `CARGO_REGISTRIES_<NAME>_TOKEN`
- `cargo install` succeeds
- `cargo-binstall` can resolve the package but fails on HTTP crate download with `401 Unauthorized`

## Current Behavior

The current code already does part of Cargo's registry resolution:

- `--registry <name>` is supported
- `CARGO_REGISTRIES_<NAME>_INDEX` is checked
- `.cargo/config.toml` `registries.<name>.index` is checked
- `registry.default` is respected
- `replace-with` for registries is supported

What is missing is registry authentication:

- no parsing of `registries.<name>.credential-provider`
- no parsing of `registry.credential-provider`
- no parsing of `registry.global-credential-providers`
- no parsing of Cargo credentials files
- no support for `CARGO_REGISTRIES_<NAME>_TOKEN`
- no support for external credential providers
- no propagation of an auth token to sparse index requests or crate download requests
- no handling of `auth-required` from registry `config.json`

In practice, `cargo-binstall` currently resolves the registry name to an index URL, then discards the rest of the registry identity. That is the main architectural reason this feature does not work.

## Cargo Semantics That Matter

Cargo authentication for registries is not "just read an env var and attach a bearer token". The important pieces are:

1. Registry identity matters.
   Cargo resolves credentials in the context of both:
   - registry name, if known
   - index URL

2. Credential provider selection matters.
   Cargo can use:
   - `cargo:token`
   - `cargo:token-from-stdout ...`
   - OS-backed built-ins such as `cargo:libsecret`, `cargo:macos-keychain`, `cargo:wincred`
   - external credential plugins using Cargo's credential provider protocol

3. `CARGO_REGISTRIES_<NAME>_TOKEN` only works if the active provider path includes `cargo:token`.

4. Registry `config.json` matters.
   Cargo registries can declare:
   - `auth-required: true`

   When that is true, Cargo includes the `Authorization` header on HTTP downloads and sparse index requests.

5. Sparse registries have a special bootstrap path.
   Cargo first tries to fetch `config.json`. If it gets `401`, it retries with credentials and may use `WWW-Authenticate` headers to guide the provider.

These details mean a correct implementation needs a real auth resolution layer, not scattered token plumbing.

## Why The Existing Architecture Falls Short

The critical path today looks roughly like this:

- `crates/bin/src/entry.rs` selects a registry index URL
- the selected URL is parsed into `binstalk_registry::Registry`
- registry fetch code uses the shared HTTP client anonymously
- crate download code also uses the shared HTTP client anonymously

That model works for:

- public sparse registries
- public git index plus public crate download endpoints

It fails for:

- private sparse registries
- private HTTP crate downloads from registries using token auth
- any registry relying on Cargo credential providers

The deepest issue is that the registry is currently modeled as "where is the index?" instead of "what Cargo registry is this, and how do I authenticate to it?".

## Recommended Design

### 1. Preserve Registry Identity

Instead of passing only a parsed `Registry`, carry a richer structure through setup:

- registry name, if present
- resolved index URL
- source of resolution for diagnostics

This should be resolved in `crates/bin/src/entry.rs`, because that is where CLI args, env vars, and Cargo config are already combined.

Why:

- provider lookup needs the registry name
- external provider protocol wants the index URL
- diagnostics are better when we know both

### 2. Add a Dedicated Auth Resolution Layer

Introduce a new module responsible for:

- reading Cargo auth-related config
- reading Cargo credentials files
- resolving which credential provider applies
- executing provider logic
- caching successful read credentials for the current process

This module should take a registry identity and return either:

- no auth available
- an auth token plus cache metadata
- a structured auth failure

Why:

- keeps Cargo-specific logic out of the downloader
- avoids duplicating provider selection logic in sparse and git registry paths
- makes testing possible without end-to-end network setup

### 3. Support Providers In Phases

Recommended initial support:

- `cargo:token`
- `cargo:token-from-stdout`
- external credential provider protocol plugins

Deferred unless explicitly needed:

- `cargo:libsecret`
- `cargo:macos-keychain`
- `cargo:wincred`

Why this split:

- `cargo:token` directly fixes the reported CI use case
- `cargo:token-from-stdout` is low-cost and fits the same abstraction
- external protocol plugins are the main extensibility path for private registries
- OS-native built-ins are more platform-specific and can be added later

### 4. Parse More of Cargo Config

`binstalk-manifests` currently parses only a narrow subset of `.cargo/config.toml`.

It should be extended to parse:

- `registries.<name>.credential-provider`
- `registry.credential-provider`
- `registry.global-credential-providers`
- `[credential-alias]`

It should also gain a separate parser for Cargo credentials:

- `$CARGO_HOME/credentials.toml`
- legacy `$CARGO_HOME/credentials`

Why:

- this is the minimum data needed to mimic Cargo's provider resolution
- keeping parsing in `binstalk-manifests` matches the existing crate boundaries

### 5. Apply Auth At The Right HTTP Boundaries

Authentication is needed in three places:

1. Sparse `config.json`
2. Sparse crate metadata file fetches
3. HTTP crate downloads

For git registries, git index access should stay as-is, but crate downloads still need auth when `auth-required` is set in the registry's `config.json`.

Why:

- the reported failure happens on the crate download URL
- private sparse registries may also require auth for index reads
- the `auth-required` flag is the standard Cargo signal for when downloads must carry auth

### 6. Keep The Downloader Generic

Avoid baking Cargo registry semantics into the generic HTTP client.

Preferred shape:

- auth resolution happens in registry code
- the resolved token is passed into request-building at call sites
- the generic client remains an HTTP transport primitive

Why:

- downloader is used by non-registry flows too
- auth policy is registry-specific, not client-global
- future changes are easier if auth is attached per request

## Sparse Registry Behavior To Implement

For sparse registries:

1. Attempt `config.json` without auth.
2. If it returns `401 Unauthorized`, collect response headers.
3. Resolve credentials for the registry.
4. Retry `config.json` with `Authorization`.
5. Parse `auth-required`.
6. If `auth-required: true`, include auth on subsequent sparse index reads and crate downloads.

Why this shape:

- it matches Cargo's documented sparse auth behavior
- it avoids sending secrets unnecessarily to public registries
- it preserves compatibility with anonymous registries

## Git Registry Behavior To Implement

For git registries:

- leave git clone/fetch behavior alone
- parse `config.json` from the git index as today
- extend the config parser to read `auth-required`
- if `auth-required: true`, attach auth to HTTP crate downloads

Why:

- the registry index itself is not fetched through the HTTP downloader in this path
- the reported failure is usually the download endpoint, not the git clone step
- this gives value even if git index auth is handled separately by the user's git setup

## Shipyard Compatibility

This design should support modern Shipyard-style Cargo auth, provided the registry is configured using standard Cargo mechanisms.

It should cover:

- registry tokens supplied by env vars
- `cargo login` stored credentials via `cargo:token`
- credential providers/plugins
- authenticated download endpoints
- `auth-required`-driven auth attachment

It does not automatically include Shipyard's older temporary `User-Agent` workaround. That behavior is non-standard and should only be added if explicitly requested.

One separate concern remains:

- private git index authentication is not the same as registry HTTP auth

If "support Shipyard" means fully supporting private Shipyard registries end-to-end, then git index auth should be validated separately. That may already work depending on the user's SSH or git credential setup.

## Backward Compatibility Requirements

The implementation should preserve:

- current behavior for public registries
- current registry selection semantics
- no auth header on registries that do not require auth
- no change to existing GitHub API token handling

New code should fail clearly when:

- a registry requires auth
- no usable provider is configured
- a configured provider fails

Diagnostics should include:

- registry name when known
- index URL
- provider used or attempted
- any login hint derived from `WWW-Authenticate`

## Recommended Rollout

### Phase 1

Implement the minimum path that fixes the reported CI issue:

- preserve registry identity
- parse auth config and Cargo credentials
- implement `cargo:token`
- parse `auth-required`
- attach auth to sparse requests and crate downloads
- add tests for env-var token resolution

This should be enough for the concrete `CARGO_REGISTRIES_<NAME>_TOKEN` failure.

### Phase 2

Add broader Cargo compatibility:

- `cargo:token-from-stdout`
- external credential provider protocol
- provider ordering and alias expansion
- more detailed auth diagnostics

This is where support becomes much closer to "works like Cargo" for private registries.

### Phase 3

Optional follow-up:

- OS-native built-in providers
- deeper Cargo config discovery/merging if needed
- more realistic end-to-end private registry fixtures

## Testing Strategy

### Unit Tests

Add tests for:

- parsing credential-provider settings
- parsing `global-credential-providers`
- parsing `credential-alias`
- parsing Cargo credentials files
- `cargo:token` env precedence
- provider ordering
- external provider protocol request/response handling

### Registry Tests

Add tests for:

- sparse `401 -> retry with auth`
- `auth-required: true` enabling auth on metadata fetch
- `auth-required: true` enabling auth on crate download
- git index config parsing including `auth-required`

### End-to-End Tests

Add a dedicated private registry auth e2e test with:

- fake sparse registry requiring auth
- token via `CARGO_REGISTRIES_<NAME>_TOKEN`
- token via Cargo credentials file
- credential provider command fixture

If possible, add a git-index plus authenticated download fixture as well.

## Open Questions

These should be decided before implementation starts:

1. Is full Cargo config discovery and merge behavior in scope, or only `$CARGO_HOME/config.toml` plus env vars?
2. Are OS-native built-in providers in scope for v1?
3. Is private git index authentication considered part of this issue, or only registry HTTP authentication?
4. Do we want to support non-standard registry auth schemes such as Shipyard's historical `User-Agent` workaround?

## Recommended Answer To Those Questions

If the goal is to fix the reported issue quickly and safely:

- scope v1 to `$CARGO_HOME/config.toml`, Cargo credentials files, and env vars
- support `cargo:token`, `cargo:token-from-stdout`, and external credential plugins
- treat private git index auth as adjacent but separate unless the issue explicitly requires it
- do not implement non-standard `User-Agent` auth in v1

Why:

- this fixes the concrete CI failure with the least complexity
- it aligns with Cargo's current documented auth model
- it avoids shipping partial platform-specific support that is hard to validate
- it keeps the first patch reviewable

## Why I Would Implement It This Way

I would not start by threading a bare token through the downloader from CLI setup. That would be the fastest patch to write, but it would be the wrong abstraction:

- it would not model provider precedence correctly
- it would not support external providers
- it would not know when auth should be attached
- it would likely spread Cargo-specific rules across unrelated modules

I would instead build a small registry-auth subsystem and keep request attachment local to the registry fetch paths. That is a slightly larger first change, but it gives a coherent model:

- registry identity in
- Cargo-compatible credential resolution
- explicit application of auth at sparse/download boundaries

That structure is the best balance between fixing the immediate bug and not painting the project into a corner.

## References

- Cargo Registry Authentication:
  https://doc.rust-lang.org/cargo/reference/registry-authentication.html
- Cargo Configuration:
  https://doc.rust-lang.org/cargo/reference/config.html
- Cargo Registry Index Format:
  https://doc.rust-lang.org/cargo/reference/registry-index.html
- Cargo Credential Provider Protocol:
  https://doc.rust-lang.org/nightly/cargo/reference/credential-provider-protocol.html
- Shipyard credential-process docs:
  https://docs.shipyard.rs/authentication/credential-process.html
- Shipyard authenticated downloads:
  https://docs.shipyard.rs/configuration/authenticated-downloads.html

# Reproducible integration demonstrations

Each directory beneath `demos/` represents one coherent, runnable user story.
The Sandbox is the integration and demonstration layer; Atlas3, WebAPI, and
Study Agent remain independently versioned source repositories.

## Rules

- Do not copy component source trees into this repository.
- Build component images in their owning repositories and publish immutable OCI
  images from CI.
- A runnable demonstration pins each component by OCI **digest**, never by a
  mutable branch or image tag.
- Record both the upstream base commit and the feature commit that was built.
- Commit templates and non-secret defaults only. Runtime secrets are supplied
  through the environment, a secret manager, or an ignored local `.env` file.
- Keep database bootstrap scripts idempotent and use demonstration-only data.

## Layout

```text
demos/
  <demo-name>/
    demo-manifest.json       # committed release provenance
    compose.yaml             # pinned integration stack
    .env.example             # variable names and safe sample values only
    README.md                # scenario, setup, smoke test, limitations
    scripts/                 # optional idempotent setup and smoke checks
```

[`study-agent-demo/demo-manifest.example.json`](study-agent-demo/demo-manifest.example.json)
is the template for the first integration demonstration. Copy it to
`demo-manifest.json` only when actual component commits and image digests are
available.

## Component CI contract

The Atlas3 and WebAPI forks should each build an OCI image after a feature-branch
commit is validated. CI should publish a human-friendly tag and an immutable
digest, and apply standard OCI labels including:

```text
org.opencontainers.image.source
org.opencontainers.image.revision
org.opencontainers.image.created
org.opencontainers.image.version
```

The image may be tagged `sha-<short-sha>` for discovery, but the Sandbox
`compose.yaml` must use the reported `@sha256:...` digest. A feature branch can
advance without changing a released demo.

## Release flow

1. Start feature branches from the documented upstream-base tag/commit.
2. Component CI tests, builds, and publishes images for a selected feature
   commit.
3. Create or update this demo's `demo-manifest.json` with the component
   provenance and image digests.
4. Update `compose.yaml` to consume those exact digests, then run the documented
   smoke test.
5. Commit the manifest, compose changes, and smoke-test evidence together; tag
   that Sandbox commit as the demonstration release.

The `validate-demo-manifests` workflow checks the required provenance fields for
every committed `demos/*/demo-manifest.json` file. It validates metadata only;
later demo-specific workflows should start the stack and run its smoke tests.

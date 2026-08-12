# symplast-openapi-specs

Central store for the **generated** OpenAPI specs of every Symplast public API surface, composed into a
single unified spec for the developer portal (scalar-ui).

## How it works

```text
each *-api service                         this repo                         dev portal
──────────────────                         ─────────                         ──────────
build Api.Public                           specs/<service>/openapi.json  ┐
  (OpenApiGenerateDocumentsOnBuild)  ──►   specs/<service>/openapi.json  ├─ redocly join ─► published/openapi.json ─► scalar-ui
  push on dev/main                         specs/<service>/openapi.json  ┘        (compose.yaml)
```

1. Each public API surface builds its own OpenAPI document at build time and **pushes** it here to
   `specs/<service>/openapi.json` from its own `publish.yaml` (e.g. `appointments-api`).
2. `.github/workflows/compose.yaml` runs on every push under `specs/**`, merges **all**
   `specs/*/openapi.json` into ONE unified document with [`redocly join`](https://redocly.com/docs/cli/commands/join),
   and commits the result to `published/openapi.json`.

## Rules

- **Specs are GENERATED — never hand-edit `specs/**` or `published/**`.** The built and running API is the
  single source of truth. Any manual change is overwritten on the next push from the owning service.
- One file per service: `specs/<service>/openapi.json` (latest spec only — no per-version history here).
- Services own **distinct path prefixes**, so there are no path collisions in the composite.
- Shared 3.1 primitive / array component schemas (`Int32`, `Boolean`, `String`, `DateTime`, `List*`, …)
  that are byte-identical across services are **de-duplicated** into a single component. Only genuinely
  conflicting component names (same name, different definition) are prefixed per service.

## Composition

`redocly join` produces `published/openapi.json`:

- `servers[0].url` is a single placeholder — `https://public-api.dev.symplast.cloud`.
- The composite is committed back to the default branch by the workflow using the repo's own
  `GITHUB_TOKEN`.

Run the composition locally the same way the workflow does:

```bash
npx --yes @redocly/cli@latest join specs/*/openapi.json -o published/openapi.json --prefix-tags-with-info-prop title
```

## Deferred (planned future enhancement)

Per-environment versioned promotion — separate `published/<env>/openapi.json` for dev / uat / prod with
per-env `servers[].url` and version pinning — is **deferred**. Today there is exactly one composite
(`published/openapi.json`) with a single placeholder server. When promotion lands, the `servers` block and
per-env fan-out will be reintroduced here.

## Adding a new service

1. Add a push step to the service's `publish.yaml` that builds its `Api.<Public>` OpenAPI document and
   pushes it to `specs/<new-service>/openapi.json` (see `appointments-api/.github/workflows/publish.yaml`).
2. That push triggers `compose.yaml`, which folds the new service into `published/openapi.json`
   automatically. No change needed in this repo.

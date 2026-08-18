# symplast-openapi-specs

Central store for the **generated** OpenAPI specs of every Symplast public API surface, composed into a
single unified spec **per audience** for the developer portal (scalar-ui).

## How it works

```text
each *-api surface                              this repo                              dev portal
──────────────────                              ─────────                              ──────────
build Api.Public                          specs/appointments-api-public/openapi.json  ┐
  (OpenApiGenerateDocumentsOnBuild)  ──►  specs/financials-api-public/openapi.json    ├─ redocly join ─► published/public-api.json ─► scalar-ui
  push on dev/main                        specs/webhooks-api-public/openapi.json       ┘        (compose.yaml)
```

1. Each public API surface builds its own OpenAPI document at build time and **pushes** it here to
   `specs/<service>-<audience>/openapi.json` (via the shared
   [`publish-openapi-spec`](https://github.com/SymplastLLC/devops-actions/tree/main/.github/actions/publish-openapi-spec)
   action in its own `publish.yaml` — e.g. `appointments-api`).
2. `.github/workflows/compose.yaml` runs on every push under `specs/**`, groups the specs **by audience**,
   merges each audience's specs into ONE composite with
   [`redocly join`](https://redocly.com/docs/cli/commands/join), and commits the result to
   `published/<audience>-api.json` (e.g. `published/public-api.json`).

## Store layout

A surface directory is named `<service>-<audience>`; the **audience is the segment after the final `-`**:

```text
specs/appointments-api-public/openapi.json      → audience: public   ┐
specs/financials-api-public/openapi.json        → audience: public   ├─► published/public-api.json
specs/webhooks-api-public/openapi.json          → audience: public   ┘
specs/appointments-api-practice/openapi.json    → audience: practice ──► published/practice-api.json  (future)
specs/appointments-api-patient/openapi.json     → audience: patient  ──► published/patient-api.json   (future)
```

Today there is exactly one audience (`public`) and one service in it (`appointments-api-public`), so the
composite is `published/public-api.json`. Additional services in the `public` audience fold into the same
composite automatically; a **new audience** lights up the moment its first spec lands **and** a row is
added to the `AUD` map in `compose.yaml` (see below).

## Rules

- **Specs are GENERATED — never hand-edit `specs/**` or `published/**`.** The built and running API is the
  single source of truth. Any manual change is overwritten on the next push from the owning service.
- One file per surface: `specs/<service>-<audience>/openapi.json` (latest spec only — no per-version
  history here).
- Services own **distinct path prefixes**, so there are no path collisions within an audience composite.
- Shared 3.1 primitive / array component schemas (`Int32`, `Boolean`, `String`, `DateTime`, `List*`, …)
  that are byte-identical across services are **de-duplicated** into a single component. Only genuinely
  conflicting component names (same name, different definition) are prefixed per service.

## Composition

`compose.yaml` produces one `published/<audience>-api.json` per audience. Each composite's identity is
stamped from the `AUD` map in the workflow — `audience → "Title|ServerUrl"`:

```bash
declare -A AUD=(
  [public]="Symplast Public API|https://api.symplast.com"
)
```

- The composite's `servers` is forced to the audience's canonical URL — `https://api.symplast.com` for
  `public` — overriding whatever the source specs carry (build defaults such as `localhost`, or per-env
  hosts). This is the single authoritative host for every consumer.
- An audience that has specs but **no `AUD` row** is skipped with a warning (never published with a wrong
  title/host) — adding the audience is a deliberate one-line change.
- Composites are committed back to the default branch by the workflow using the repo's own `GITHUB_TOKEN`.

Run the composition for the `public` audience locally the same way the workflow does:

```bash
npx --yes @redocly/cli@latest join specs/*-public/openapi.json -o published/public-api.json --prefix-tags-with-info-prop title
jq '.servers = [{ "url": "https://api.symplast.com" }] | .info.title = "Symplast Public API"' \
   published/public-api.json > tmp && mv tmp published/public-api.json
```

## Versioning

The repo carries an [nbgv](https://github.com/dotnet/Nerdbank.GitVersioning) `version.json`. Every time
`compose.yaml` publishes new composites, it stamps an immutable git tag `v<nbgv-version>` (e.g. `v0.1.7`)
on that commit. The dev portal pins a specific tag instead of the moving `main`:

```
https://raw.githubusercontent.com/SymplastLLC/symplast-openapi-specs/v0.1.7/published/public-api.json
```

Promotion is then a portal values-file bump of the pinned tag — no change in this repo. Per-environment
tag resolution (which tag each of dev / uat / prod serves) lives in the portal.

## Deferred (planned future enhancement)

Per-environment versioned promotion — separate `published/<env>/<audience>-api.json` for dev / uat / prod
with per-env `servers[].url` and version pinning — is **deferred**. Today each audience has exactly one
composite (`published/<audience>-api.json`) whose server is fixed to the production URL. When promotion
lands, per-env `servers` injection and the per-env fan-out will be added here.

## Adding a new service (existing audience)

Point the service's `publish.yaml` at the shared `publish-openapi-spec` action with
`service-name: <service>-<audience>` (e.g. `financials-api-public`). Its push to
`specs/<service>-<audience>/openapi.json` triggers `compose.yaml`, which folds it into that audience's
composite automatically. **No change needed in this repo.**

## Adding a new audience

1. A surface pushes to `specs/<service>-<newaudience>/openapi.json`.
2. Add one row to the `AUD` map in `.github/workflows/compose.yaml`:
   `[<newaudience>]="<Title>|<ServerUrl>"`.
3. The next compose produces `published/<newaudience>-api.json`; point the portal at it.

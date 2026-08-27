# symplast-openapi-specs

Central store for the **generated** OpenAPI specs of every Symplast public API surface, composed into a
single unified spec **per audience** for the developer portal (scalar-ui).

> Working in this repo with an AI agent? [`AGENTS.md`](./AGENTS.md) is the agent-facing operating guide
> (per [RFC-002](https://app.notion.com/p/3449a00fb25b8150899be3d70ff6e62d)). This README is the human one.

## How it works

```text
each *-api surface                              this repo                              dev portal
──────────────────                              ─────────                              ──────────
build Api.Public                          specs/appointments-api-public/openapi.json  ┐
  (OpenApiGenerateDocumentsOnBuild)  ──►  specs/financials-api-public/openapi.json    ├─ bin/compose ─► published/public-api.json ─► scalar-ui
  push on dev/main                        specs/legacy-api-public/openapi.json         ┘   (redocly join)
```

1. Each public API surface builds its own OpenAPI document at build time and **pushes** it here to
   `specs/<service>-<audience>/openapi.json` (via the shared
   [`publish-openapi-spec`](https://github.com/SymplastLLC/devops-actions/tree/main/.github/actions/publish-openapi-spec)
   action in its own `publish.yaml` — e.g. `appointments-api`).
2. `.github/workflows/compose.yaml` runs on every push under `specs/**`, `canonical/**`, or `bin/compose`.
   It delegates to **`bin/compose`**, which groups the specs **by audience**, unifies any schema shared by
   more than one spec, merges each audience's specs into ONE composite with
   [`redocly join`](https://redocly.com/docs/cli/commands/join), and commits the result to
   `published/<audience>-api.json` (e.g. `published/public-api.json`).

## Repository layout

```text
bin/compose                              the composer — all composition logic, runnable locally
canonical/<audience>/<Name>.json         hand-authored docs for models shared by several specs
specs/<service>-<audience>/openapi.json  GENERATED inputs, pushed by each owning service
published/<audience>-api.json            GENERATED output, composed and committed by CI
.github/workflows/compose.yaml           resolves the version, runs bin/compose, commits, tags
version.json                             nbgv config (publicReleaseRefSpec, pathFilters)
```

`canonical/` is the only content here that is meant to be hand-edited. Everything under `specs/` and
`published/` is generated.

## Store layout

A surface directory is named `<service>-<audience>`; the **audience is the segment after the final `-`**:

```text
specs/appointments-api-public/openapi.json      → audience: public   ┐
specs/financials-api-public/openapi.json        → audience: public   ├─► published/public-api.json
specs/legacy-api-public/openapi.json            → audience: public   ┘
specs/appointments-api-practice/openapi.json    → audience: practice ──► published/practice-api.json  (future)
specs/appointments-api-patient/openapi.json     → audience: patient  ──► published/patient-api.json   (future)
```

The `public` rows are real; the `practice` / `patient` rows illustrate how a new audience would fan out.

`public` is currently the only audience, so `published/public-api.json` is the only composite. Every
service in an audience folds into that audience's composite automatically — adding one needs **no change in
this repo**. A **new audience** lights up the moment its first spec lands **and** a case is added to
`audience_meta()` in `bin/compose` (see below).

To see which surfaces are in the store right now, list the directories rather than trusting a doc:

```bash
ls -d specs/*/
```

## Rules

- **Specs are GENERATED — never hand-edit `specs/**` or `published/**`.** The built and running API is the
  single source of truth. Any manual change is overwritten on the next push from the owning service.
- One file per surface: `specs/<service>-<audience>/openapi.json` (latest spec only — no per-version
  history here).
- Services own **distinct path prefixes**, so there are no path collisions within an audience composite.
- **Model and tag names are never prefixed.** The composite reads as one unified API, not a stitched-together
  set of services — `PublicCreditNote`, not `Symplast_Financials_Public_API_PublicCreditNote`.
- A schema name defined by **more than one spec in the same audience** must resolve to exactly ONE model:
  - **Same shape** → unified into a single component. Its documentation comes from
    `canonical/<audience>/<Name>.json` when that overlay exists, otherwise from the first spec (with a
    warning telling you to add the overlay).
  - **Different shapes** → the compose **fails and publishes nothing**. Two different shapes under one name
    in a published contract is a contract bug; it must be reconciled upstream or renamed, not masked by a
    per-service prefix.

## Composition

`bin/compose` produces one `published/<audience>-api.json` per audience. Each composite's identity is
stamped from `audience_meta()` in that script — `audience → "Title|ServerUrl"`:

```bash
audience_meta() {
  case "$1" in
    public) echo "Symplast Public API|https://api.symplast.com" ;;
    *)      return 1 ;;
  esac
}
```

- The composite's `servers` is forced to the audience's canonical URL — `https://api.symplast.com` for
  `public` — overriding whatever the source specs carry (build defaults such as `localhost`, or per-env
  hosts). This is the single authoritative host for every consumer.
- An audience that has specs but **no `audience_meta` case** is skipped with a warning (never published
  with a wrong title/host) — adding the audience is a deliberate one-line change.
- Composites are committed back to the default branch by the workflow using the repo's own `GITHUB_TOKEN`.

Run the composition locally exactly the way the workflow does — same script, same output:

```bash
COMPOSITE_VERSION=0.0.0-local bin/compose
```

`COMPOSITE_VERSION` is the only input; CI supplies this repo's nbgv version, and any placeholder works
locally. The script needs `jq` and `npx`, exits non-zero on an unresolved schema conflict, and writes
nothing to `published/` when it fails.

## Canonical model overlays

When several specs of one audience define the same schema, the composite needs ONE set of docs for the
unified model. That lives in `canonical/<audience>/<SchemaName>.json`:

```text
canonical/public/ProblemDetails.json     → docs for the unified `ProblemDetails` in published/public-api.json
canonical/practice/ProblemDetails.json   → docs for the unified `ProblemDetails` in published/practice-api.json
```

- **Overlays are per-audience and never shared across audiences.** There is no global overlay directory and
  no fallback between surfaces — `public` will not borrow `practice`'s docs or vice versa, because the
  surfaces evolve independently.
- An overlay may only restate **annotations** (`description`, `example`, …). Its shape is verified against
  what the services actually emit and a mismatch **fails the compose**, so `canonical/` can never become a
  place where the published contract drifts from the running APIs.
- Overlays are the one hand-authored artifact here — unlike `specs/**` and `published/**`, they are meant to
  be edited and reviewed in PRs.

## Versioning

The composite is **this repo's own artifact**, so its version is **this repo's** [nbgv](https://github.com/dotnet/Nerdbank.GitVersioning)
version — never any single downstream service's build number. `compose.yaml` resolves the nbgv
`SimpleVersion` (`0.1.<height>`) once per run and uses it for **both**:

- the composite's `info.version` (stamped into every `published/<audience>-api.json`), overriding each
  service's own image-tag `info.version`; and
- an immutable git tag `v<nbgv-version>` (e.g. `v0.1.7`) on the publishing commit.

So the version a consumer reads *inside* the spec always equals the tag they pin it by. `version.json`'s
`pathFilters` exclude `published/**`, so the bot's own compose commits do **not** inflate the height — the
number counts real spec revisions and stays stable across re-runs / manual dispatches.

`version.json` marks `main` as a public release ref, matching the rest of the org. `SimpleVersion` is clean
on every branch, so this does not affect the composite's version today — it only matters if the version step
ever switches to `SemVer2`.

> **Why not the shared [`calculate-version`](https://github.com/SymplastLLC/devops-actions/tree/main/.github/actions/calculate-version)
> action?** Because `devops-actions` is **private** and this repo is **public** (it has to be, so the portal
> can fetch `published/**` over `raw.githubusercontent.com`). GitHub does not let a public repo consume an
> action from a private one, even in the same org — the job dies at "Set up job" with
> `Unable to resolve action`. Every repo using that action successfully is private.

> The **per-service** specs under `specs/**` keep their own image-tag `info.version` (which build produced
> them) as provenance — only the composed `published/**` version is rewritten to this repo's version.

The dev portal pins a specific tag instead of the moving `main`:

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
2. Add one case to `audience_meta()` in `bin/compose`:
   `<newaudience>) echo "<Title>|<ServerUrl>" ;;`.
3. The next compose produces `published/<newaudience>-api.json`; point the portal at it.
4. If two of that audience's specs share a schema name, add `canonical/<newaudience>/<Name>.json` to own the
   unified model's documentation. Overlays are never shared between audiences.

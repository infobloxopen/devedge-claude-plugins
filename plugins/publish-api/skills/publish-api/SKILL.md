---
name: publish-api
description: >-
  Publish a devedge service's public API and wire up a typed client. Use this
  whenever the user wants to publish, release, or share a service's API as OpenAPI
  "through apx" or "to the catalog"; generate or regenerate a TypeScript/Angular
  API client from a service; wire a micro-frontend to a backend with a generated
  client; or iterate on a client against an unreleased API — for example "publish
  the orders API", "generate a typed client for notesd", "connect the uFE to the
  backend", or "regenerate the client after I changed the proto". apx orchestrates
  the codegen; this skill drives the publish → generate → consume flow end to end.
argument-hint: <service name/dir and, optionally, the domain and version to publish>
---

# Publish a devedge API and generate its typed client

Take a devedge-sdk service from its OpenAPI spec to a published API in the apx catalog
and a **consumable, typed `@<scope>/<svc>-client` npm package** a micro-frontend imports —
no hand-written client, no manual `ng-openapi-gen`. There is also a **local hot-loop**:
regenerate the client from the on-disk spec (or an unreleased upstream API) and consume it
over a `file:` link with no publish.

## Where the truth lives (read this first)

Do not guess flags or doc paths — the CLI help is authoritative and always matches the
installed version:

- **`de api publish --help`** and **`apx client generate --help`** / **`apx client publish --help`**
  — the exact flags, defaults, and the steps each command performs. Read these before running
  anything.
- **Hosted how-to (prose + rationale):** the devedge-sdk docs site,
  `how-to/operate/publish-openapi` (from <https://infobloxopen.github.io/devedge-sdk/docs/>).
  It walks the whole path: annotate `google.api.http` → `make generate` → publish → generate
  the client → host it in a micro-frontend.
- **A worked example:** `examples/fullstack-oss` in `infobloxopen/devedge-ufe-sdk` — a real
  backend + Angular uFE consuming the generated client. Read its `README.md` and
  `frontend/notes-ufe/src/app/notes.service.ts` for the consumption pattern.

apx is the API schema **lifecycle** tool; it does not reimplement codegen. `apx client generate`
**orchestrates `ng-openapi-gen`** and wraps the output in a buildable npm package. Prefer these
commands over hand-running `ng-openapi-gen`.

## 1. Confirm prerequisites

- **`apx` on `PATH`** (the whole flow shells out to it): `go install github.com/infobloxopen/apx@latest`.
  Client generation also needs **Node ≥ 18** (`npx`) — apx runs `ng-openapi-gen` via `npx`.
- The service already emits its spec at **`openapi/<svc>.openapi.yaml`** (produced by
  `make generate` — `buf` → `protoc-gen-openapiv2` → the v2→v3 converter). If it is stale or
  missing, run `make generate` in the service dir first.

## 2. Publish the API to the apx catalog

`de api publish` runs `make generate`, arranges the spec into the apx layout
(`openapi/<domain>/<svc>/<line>/`), and runs `apx release prepare`. Without `--submit` it prints
the follow-on `apx release submit` / `finalize` commands for you to run after reviewing the PR it
opens on the canonical repo.

```bash
de api publish \
  --domain <domain> \
  --api-id openapi/<domain>/<svc>/<line> \
  --version vX.Y.Z \
  --lifecycle <alpha|beta|stable> \
  --canonical-repo github.com/<org>/apis
# add --submit to also open the release PR in one shot
```

Settle `--domain`/`--api-id`/`--version`/`--lifecycle` with the user (the proto package
`infoblox.<domain>.v1` already implies the domain). Confirm the flags against `de api publish --help`.

## 3. Generate the typed client

Two equivalent ways — pick one:

- **Chained with the publish** (one command): add `--client` to step 2. It writes an
  `@<scope>/<svc>-client` module to `--client-out` (default `clients/<svc>-client`), with
  `--client-scope` setting the npm scope.
- **Standalone** (the common inner-loop): run the generator directly.

```bash
apx client generate \
  --input openapi/<svc>.openapi.yaml \
  --scope @<org> --package <svc>-client \
  --output clients/<svc>-client \
  --build          # npm install + tsc, so dist/ (main/types) exists for a file: consumer
```

The result is a real npm package: `dist/` + `.d.ts`, `@angular/*` and `rxjs` as **peer**
dependencies, `publishConfig` → GitHub Packages, a barrel exporting the models, the
per-operation functions, `ApiConfiguration`, and **`provideApiConfiguration(rootUrl)`**.

## 4. Consume it in a micro-frontend (the hot-loop)

Locally, link the package with no publish:

```jsonc
// the uFE's package.json
"dependencies": { "@<org>/<svc>-client": "file:../../clients/<svc>-client" }
```

Wire the base URL and let the app's HttpClient interceptors (e.g. the devedge-ufe bearer
interceptor) apply — the generated operations use the injected `HttpClient`:

```ts
import { provideApiConfiguration } from '@<org>/<svc>-client';
// in the NgModule/standalone providers:
provideApiConfiguration(environment.apiBaseUrl)
```

The hot-loop is: edit the proto → `make generate` → `apx client generate` (rebuild the client) →
the uFE picks up the new models/operations on its next build. No publish, no version bump.

## 5. Publish the client (when you want to share it)

To publish the `@<scope>/<svc>-client` module to GitHub Packages, add `--publish-client` to
`de api publish`, or run `apx client publish --dry-run=false`. The default `apx client publish`
is a **dry-run** that validates the tarball without publishing — use it to check publishability.
A real publish needs npm auth for the registry (a `GITHUB_TOKEN` / PAT with `packages:write`,
normally supplied in CI on a tag). Consumers then depend on `^X.Y.Z` from GitHub Packages
instead of the `file:` link.

## 6. Iterate against an UNRELEASED API (cross-team hot-loop)

To generate a client from an API that is not released yet — a local checkout or a colleague's
branch/fork — pin the dependency to that source, then point the generator at it:

```bash
apx add openapi/<domain>/<svc>/<line> --path ../their-apis      # local checkout
# or: apx add openapi/<domain>/<svc>/<line> --git github.com/<org>/apis --ref <branch>
apx client generate --from openapi/<domain>/<svc>/<line> --scope @<org> --package <svc>-client
```

This resolves the spec from the override (local dir or a shallow git clone of the ref/fork) and
generates a client from it. When the upstream API is released, swap the override back to a
released version with `apx update` / `apx unlink`.

## Gotchas to expect (verified on the WS-020 ship; check your apx version)

- **`dist/` is a build artifact — build the client before a `file:` consumer resolves it.** The
  package resolves `main`/`types` to `dist/`, and a `file:` dependency is not built for you. Use
  `--build` (or `npm install && npm run build` in the client dir) so `dist/` exists. In a service
  repo, treat `dist/` as generated output (git-ignored), not committed source.
- **Set the base URL with `provideApiConfiguration(rootUrl)`** — it is exported from the package
  barrel. Because the generated operations take the app's injected `HttpClient`, your existing
  interceptors (bearer auth, etc.) apply automatically; the client carries no auth code.
- **A real publish needs `packages:write`.** `apx client publish` defaults to `--dry-run` on
  purpose; `--dry-run=false` only succeeds with npm auth for GitHub Packages (usually CI-on-tag).
  A dry-run proving a valid tarball is the local success signal.
- **Releasing is blocked while an unreleased override is pinned.** After `apx add --path/--git`,
  `apx release prepare|submit|finalize|promote` fail closed with a clear message — that is the
  guardrail keeping unreleased sources out of a release. Replace the override with a released
  version before releasing.
- **Don't hand-run `ng-openapi-gen`.** `apx client generate` orchestrates it and produces the
  packaged, consumable module; the raw generator output is not a package.

## Guardrails

- Read `de api publish --help` / `apx client generate --help` for exact flags; never invent them.
- Prefer the automated `apx client generate` path over the older manual `ng-openapi-gen` recipe.
- Keep generated client source out of a service repo's tree as committed code; it is regenerated.
- If a documented step does not work, say so plainly and file it against `devedge-sdk`/`apx` —
  do not paper over it with an undocumented workaround.

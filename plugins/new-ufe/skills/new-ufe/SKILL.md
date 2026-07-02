---
name: new-ufe
description: >-
  Scaffold a micro-frontend (uFE) with devedge and wire it to a backend and a
  session. Use this whenever the user wants to scaffold a micro-frontend or uFE
  "with devedge", create a frontend for their service, add a uFE to the shell,
  wire an Angular app to a backend with devedge, or asks for "de ufe new" — for
  example "scaffold a discovery uFE with devedge", "create a frontend for my
  orders service", "add a tags uFE to the shell", or "wire an Angular app to my
  backend". The generated uFE is Angular-15 + single-spa on the open-core
  @infobloxopen/devedge-ufe-* SDK, correct on first run, and registered into a
  kind:Shell roster. Pairs with the publish-api skill for a typed backend client.
argument-hint: <uFE name and, optionally, its route, dev-port, and which shell roster to join>
---

# Scaffold a devedge micro-frontend (uFE)

Take a developer from "I need a frontend" to a running Angular-15 + single-spa uFE that
renders inside a shell, carries the session into Angular DI, and calls a backend API with a
bearer token — no hand-wired lifecycle, no silently-dropped nav, no Angular-2-era deadweight.
The uFE is **correct on first run**; your job is to frame it, scaffold it, understand the
wiring, connect it to an API, and prove it renders + calls the backend.

## Where the truth lives (read this first)

Do not guess flags or doc paths — the CLI help is authoritative and matches the installed
version:

- **`de ufe new --help`** and **`de ufe --help`** — the exact flags, defaults, and what the
  scaffold produces. Read these before running anything. Confirm `de project up --help` before
  telling the user how to bring a shell up.
- **Frontend SDK reference + hosting model (prose):** the devedge-ufe-sdk site,
  <https://infobloxopen.github.io/devedge-ufe-sdk/> — the SDK packages, how a shell hosts many
  uFEs, and how a uFE loads (native import map + hash routes). Navigate from the site home
  rather than hard-coding deep paths.
- **Where `de ufe new` fits in the full-stack flow:** the devedge portal,
  <https://infobloxopen.github.io/devedge/>.
- **A worked example — read it:** `examples/fullstack-oss` in `infobloxopen/devedge-ufe-sdk` —
  a real Angular uFE wired to a backend + session + a generated client. Read its `README.md`
  and `frontend/notes-ufe` for the consumption pattern.

The SDK is open-core, five packages: `@infobloxopen/devedge-ufe-core` (lifecycle,
nav-contribution validation, `SessionProvider`/`authedFetch`, manifest), `-oidc` (generic
OIDC session provider), `-single-spa` (loader + shell-owns-session gate), `-angular`
(`SESSION_PROVIDER` + `provideDevedgeSession` + origin-scoped bearer interceptor), and
`-dev-loop` (the `devedge-ufe doctor` diagnostic + import-map override for hot-reload).

## 1. Frame the uFE with the user

From the user's request, settle just enough to scaffold — ask only if genuinely unclear:

- The **name** (a short noun, e.g. `discovery`, `tags`). It defaults the hash **route** and the
  CDN path segment unless you override `--route`.
- The **route** (`--route`) and **dev-server port** (`--dev-port`, default `4201`) if the
  defaults collide with another uFE.
- **Which shell roster** it joins (`--shell`, default `shell.yaml`) — an existing shell config,
  a new one (created if the file is absent), or **`--shell ""` to scaffold only** with no
  roster wiring.
- Whether to apply an **Infoblox-CTO preset**. The public CLI ships **no** proprietary preset;
  the `infoblox-cto` overlay comes from the private `Infoblox-CTO/devedge-ufe-sdk-internal`
  repo via `--preset-dir <repo>/preset/infoblox-cto`. Only mention this if the user is inside
  Infoblox and has that repo.

State the plan back in one line before scaffolding.

## 2. Scaffold with `de ufe new`

```bash
de ufe new <name> \
  --route <route> \
  --dev-port <port> \
  --shell <shell.yaml|"">    # "" to skip roster wiring
# add --preset-dir ../devedge-ufe-sdk-internal/preset/infoblox-cto only inside Infoblox
```

Confirm every flag against `de ufe new --help`. The scaffold produces the Angular-15 +
single-spa app on the `@infobloxopen/devedge-ufe-*` SDK, and — unless `--shell ""` — upserts a
roster entry `{id: <name>, route: <route>, upstream: http://127.0.0.1:<dev-port>}` into the
`kind: Shell` file **by id** (an existing same-id entry is updated in place, never duplicated).
If `--shell` names a file that does not exist, a sensible default shell containing just this
uFE is created.

## 3. Understand the generated wiring

Walk the reader through what makes it correct on first run — do not re-implement any of it:

- **Nav group + GroupRegistry.** The uFE contributes a nav group whose default value is one
  the dev `GroupRegistry` recognizes, so it **renders** rather than being silently dropped
  (see the gotcha). If you change the group, validate it against the registry.
- **App route ↔ manifest.** The app's mount route matches the manifest the shell reads, so the
  shell routes to it.
- **Session in Angular DI.** The session is provided into DI (`SESSION_PROVIDER` /
  `provideDevedgeSession` from `-angular`); the shell owns the session and hands it down
  (shell-owns-session gate from `-single-spa`).
- **Bearer interceptor.** An origin-scoped bearer interceptor on the shared `HttpClient` adds
  the session's token to outbound calls — so a generated API client carries auth with no
  auth code of its own.
- **The `kind: Shell` roster entry** from step 2 is how a shell discovers and hosts the uFE.

## 4. Wire it to a backend API

Cross-reference the **publish-api** skill — it generates the typed client this uFE consumes.
Once you have a `@<scope>/<svc>-client` package (from `de api publish --client` or
`apx client generate`), link it and set the base URL:

```jsonc
// the uFE's package.json
"dependencies": { "@<org>/<svc>-client": "file:../../clients/<svc>-client" }
```

```ts
import { provideApiConfiguration } from '@<org>/<svc>-client';
// in the uFE's standalone/NgModule providers:
provideApiConfiguration(rootUrl)   // rootUrl = the backend's base URL
```

Because the generated operations use the app's injected `HttpClient`, the scaffold's bearer
interceptor applies automatically — the client needs no auth code. See the publish-api skill
for how to generate/regenerate the client (including the local `file:`-link hot-loop).

## 5. Run it and prove it works

Build/serve the uFE, then bring the shell up so the uFE is hosted:

- Build or consume the `@infobloxopen/devedge-ufe-*` packages, then `ng build` / `ng serve` the
  uFE (follow the example's README + the ufe-sdk site — the SDK packages must resolve first).
- Bring the shell up so it hosts the uFE:
  `de project up -f <shell>.yaml` (confirm against `de project up --help`; `-f`/`--file` selects
  the shell config, `--watch` keeps its routes/leases alive).
- Run `devedge-ufe doctor` (from `-dev-loop`) to diagnose loading/import-map issues.

**Verify** the uFE renders in the shell (its nav group shows up, not silently dropped) and that
a real call to the backend returns data with the bearer token attached. Capture that for the user.

## Gotchas to expect (confirm against your SDK/CLI version)

- **The silent-nav-drop trap.** A nav `group` that is not in the recognized set renders
  **nothing, with no error**. The scaffold's validated default avoids it; if you change the
  group, validate it against the dev `GroupRegistry` (loud validation in `-core`) or it will
  vanish silently.
- **uFEs share an origin/host model — Go services do not.** A shell FQDN hosts many uFEs, which
  load via a **native import map + hash routes**, with `cdn.dev.test` serving uFE assets. This
  is a different hosting model from a backend Go service; do not assume one uFE == one origin.
- **Local hot-reload uses an import-map override.** To iterate on a uFE against a running shell,
  use the `-dev-loop` import-map override to point the shell at your local dev server instead of
  the CDN build — that is the supported hot-reload path.
- **`--shell ""` scaffolds without roster wiring.** Use it when you are not ready to join a
  shell; the roster entry is added later (re-run with `--shell <file>`; it upserts by id).
- **The public CLI ships no preset.** `--preset <name>` has no built-in value in the public CLI;
  the `infoblox-cto` overlay is `--preset-dir <internal-repo>/preset/infoblox-cto`. An unknown
  preset or a malformed `preset.json` fails with a clear error.

## Guardrails

- Read `de ufe new --help` for exact flags; never invent them.
- Do not hardcode a proprietary preset in public guidance — the public CLI ships none; the
  `infoblox-cto` overlay is private (`--preset-dir`).
- Prefer the scaffold's correct-on-first-run wiring (nav validation, session DI, bearer
  interceptor, manifest, roster entry) over hand-editing. If you must change the nav group,
  validate it.
- If a documented step does not work, say so plainly and file it against `devedge-ufe-sdk` /
  `devedge` — do not paper over it with an undocumented workaround.

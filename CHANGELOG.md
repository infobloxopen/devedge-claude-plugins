# Changelog

## v0.5.0 — 2026-07-09

- **`new-shell` (0.1.0, new)** — scaffold and run a single-spa **shell** that hosts your devedge uFEs
  behind a left **side-nav menu** with `de ufe shell`. It reads the `kind:Shell` roster `de ufe new`
  writes and renders each uFE in the shell (active-route nav, signed-in user, mount-in-content) — the
  self-contained "see it render" loop, with no example-shell copying. Documents the **commercial shell**
  (`--preset-dir …/infoblox-cto-shell`: Okta session + the grouped INFOBLOX_GROUPS nav + PDS) and the
  **integrated-env** dev loop (`de ufe override` — inject a locally-served uFE into a live hosted shell
  via browser import-map-override, no proxy). Requires devedge `de ufe shell` (v0.16.0+) / `de ufe
  override` (v0.17.0+). Pairs with `new-ufe` (scaffold the uFEs) and `run-locally` (the dev edge).

## v0.4.0 — 2026-07-03

Corrections from the devedge DX hardening cadence (Runs 16–19), aligned with devedge-sdk v0.52.0 /
de v0.13.0 / devedge-ufe-sdk v0.1.3.

- **`compose-services` (0.2.0)** — corrected the composition-member contract. A member exposes the
  seam **`NewModule(db *gorm.DB) servicekit.Module` + `Models() []any`** (emitted by
  `devedge-sdk new service` at v0.52.0+, gorm), which the composed host builds over one shared DB —
  **not** the previously-documented zero-arg `Module()`, which never compiled as a composed binary.
  Documents `de compose add --path <dir>` for local unpublished members, and that `de compose test`
  now fails loud (no false pass) for an unverifiable member.
- **`new-ufe` (0.1.1)** — added a local-dev auth note: the dev authorizer reads raw
  `account-id`/`groups` metadata, so a bearer token alone gets `PermissionDenied` locally; the
  scaffold wires the dev-only `devAuthInterceptor` (devedge-ufe-sdk v0.1.3+) that stamps those
  headers for the local round-trip.

## v0.3.0 — 2026-07-02

- **`model-domain` (0.1.0)** — a domain-modeling skill with Domain-Driven Design rigor,
  spined on the **core-model-vs-surfaces** separation. It drives the modeling judgment:
  classify entities vs value objects; find invariants and draw the smallest **aggregate**
  boundary (root as sole write entry point, members, containment vs cross-aggregate
  **references**); keep aggregates small and reference other aggregates by ID; and expose read
  **surfaces** (multiple API versions/admin views over one backing model, read-only rollups,
  member lookups) separately from the write model. It maps that judgment onto devedge's native
  DDD machinery — `infoblox.ddd.v1` `aggregate`/`member`/`references`, the `AggregateRepository`
  `Load`/`Save`, the root `Validate(ctx)` invariant hook, the fail-closed `AssertAggregateBoundaries`
  gate, `storage.v1.model` multi-surface projections, and eventual cross-aggregate consistency via
  the transactional outbox — grounded in `concepts/aggregates.md`, the `ddd`/`field`/`storage`
  proto source, and the `testdata/iam` worked example. Composes with `publish-api` (surfaces) and
  is distinct from `new-service` (bootstrap). Marketplace bumped to 0.3.0.

## v0.2.0 — 2026-07-02

Four new plugins covering the common devedge operations beyond bootstrapping. Each is a single
skill matching the `new-service` shape: a trigger-focused description, a "where the truth lives"
grounding section (hosted docs + `de`/`apx --help` as authoritative), a numbered workflow,
verified gotchas, and guardrails. `new-service` is unchanged (stays at 0.1.2).

- **`publish-api` (0.1.0)** — publish a service's public API as OpenAPI v3 through apx and
  generate/consume a typed `@<scope>/<svc>-client` TypeScript/Angular package (apx orchestrates
  ng-openapi-gen). Covers `de api publish [--client|--publish-client]`, standalone
  `apx client generate`/`publish`, consuming the client in a uFE via `provideApiConfiguration`,
  and the unreleased-API hot-loop (`apx add --path/--git` + `apx client generate --from`).
- **`run-locally` (0.1.0)** — bring a service up on the local dev edge and round-trip it:
  `de install` (daemon + `*.dev.test` DNS + TLS trust), daemon lifecycle (`start`/`status`/
  `doctor`), route registration (`de project up` from `devedge.yaml`, or `de register`), reaching
  the service, and diagnosing daemon-vs-route-vs-trust failures.
- **`new-ufe` (0.1.0)** — scaffold an Angular-15 + single-spa micro-frontend with `de ufe new`,
  wired to the open-core `@infobloxopen/devedge-ufe-*` SDK (validated nav group, session in
  Angular DI, bearer interceptor) and registered into a `kind:Shell` roster. Cross-references
  `publish-api` for a typed backend client.
- **`compose-services` (0.1.0)** — compose several service modules into one host process with
  `de compose` (init → add → tidy → build → test → up/chart). Covers the module/host split,
  static composition (no Go plugins), descriptor/version conflict checks, DB module-namespacing
  under tenant isolation, and the `failurePolicy`/chart topologies.

## v0.1.2 — 2026-07-01

### new-service skill

**P1 — Gotcha #1 (handler-override) corrected.** The previous text claimed the generated
module had no handler-override option and told developers to hand-roll a parallel
`servicekit.Module`. As of devedge-sdk v0.43.0 this is false. The generated `module/module.go`
ships a first-class `<Svc>ServiceModuleOptions{ Handler: ... }` field with an inline commented
recipe. The skill now correctly directs developers to set that option and embed the generated
`<Svc>CRUDHandler`, not to bypass the generated module.

**P1 — Gotcha #2 (empty-id Create) inverted.** The previous text told developers to
`uuid.NewString()` before `Repo.Create` because the framework did not auto-generate ids. As of
v0.43.0 the `id` field defaults to `SERVER_GENERATED` (UUID7) and `Create` returns a populated
id automatically. The workaround is now anti-guidance. Replaced with a correct one-liner plus a
brief note on the `USER_SETTABLE` strategy for the rare caller-supplied-id case.

**P1 — ent/gorm field-modeling guidance added.** A new paragraph in section 4 warns about the
storage constraints developers hit when a literal domain spec meets generate-time walls:
`repeated string` is not a column (use `map<string,string>` or a child resource), writable
timestamps/nested messages are not scalar columns, and `OUTPUT_ONLY` fields produce no setters
so custom-method state must live in a writable scalar field.

**P2 — Offline docs pointer fixed.** `docs/content/docs` does not ship in the published Go
module. The "Where the truth lives" snippet now points at the hosted docs site (primary), the
module's `AGENTS.md`, and `CLAUDE.md` (which do ship), keeping the correct `proto/infoblox`
pointer unchanged.

**P2 — HTTP gateway Create-body gotcha added.** For a `body: "<resource>"` mapped Create RPC,
the correct JSON body is resource fields at the top level. The nested form (`{"bookmark":{…}}`)
is silently accepted but drops every user field.

**P2 (minor) — Custom-methods how-to URL corrected** to `how-to/model-and-persist/custom-methods/`.

**P2 (minor) — Default backend corrected.** The CLI default is `gorm`, not `ent`. The skill
now says so explicitly and recommends always passing `--backend`.

## v0.1.1

Initial release.

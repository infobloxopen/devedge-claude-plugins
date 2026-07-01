---
name: new-service
description: >-
  Bootstrap and build a new microservice with devedge. Use this whenever the user
  wants to build, create, scaffold, start, or initialize a service, microservice,
  API, or backend "with devedge" or "with devedge-sdk" — for example "build a
  shopping cart microservice with devedge" or "create an orders service using
  devedge-sdk". The user supplies the business domain; this skill drives the
  mechanics end to end.
argument-hint: <service name and/or a one-line description of the domain>
---

# Bootstrap a devedge service

Drive a greenfield developer from a one-line business idea to a running, fail-closed
devedge-sdk service. The user brings the domain (a shopping cart, orders, inventory,
payments, …); you figure out the mechanics from the docs, the tooling, and the SDK source.

## Where the truth lives (read this first)

Two sources, used for different things — do not guess API shapes:

- **Hosted docs (prose, rationale, the documented flow):**
  <https://infobloxopen.github.io/devedge-sdk/docs/>. Landmarks: Getting Started
  (Installation, Quickstart); Concepts (annotations, tenant-isolation, aggregates); the
  Guides/How-to section (Define a service; Model a resource; **Add a custom method or second
  resource**; Run seccheck); Reference (`servicekit`, `codegen`, `server`, `persistence`,
  `middleware`). Navigate from the docs home rather than hard-coding deep paths.
- **The SDK source — GROUND TRUTH for exact annotations and behavior.** After you install or
  `go get` the SDK it is vendored locally; read the real `.proto` and `.go` there:

  ```bash
  SDK_DIR="$(go env GOMODCACHE)/github.com/infobloxopen/devedge-sdk@${SDK_VERSION}"
  ls "$SDK_DIR/proto/infoblox"            # authz / field / storage annotation definitions
  cat "$SDK_DIR/AGENTS.md"               # agentic conventions and module layout
  cat "$SDK_DIR/CLAUDE.md"               # Claude Code instructions (both ship in the module)
  ```

  Note: `docs/content/docs` does **not** ship in the published Go module — use the hosted docs
  site above for prose. The `proto/infoblox` tree and the root `AGENTS.md`/`CLAUDE.md` do ship
  and are the safe offline references.

  When you need the exact field-relationship or storage-projection annotations
  (`has_many`, `belongs_to`, `(infoblox.storage.v1.model)`, `(infoblox.field.v1.opts)`), read
  `proto/infoblox/field/v1/field.proto` and `proto/infoblox/storage/v1/storage.proto`
  directly. Prefer reading source over a fetched-and-summarized doc page — a paraphrase of an
  annotation is not safe to build on. If the docs and the source/tools disagree, trust the
  source and tell the user (that is a real doc bug worth filing).

## 1. Frame the domain with the user

From the user's one line, settle just enough to scaffold — ask only if genuinely unclear:

- The **service name** (a short noun, e.g. `cartd`) and Go **module path**.
- The **owner resource** (e.g. `Cart`) and its fields, including which need `unique`,
  `secret`, `soft-delete`, an `etag`, and tenant scoping (`account_id`).
- Any **child resources** (e.g. `CartItem` under `Cart`) and **custom, non-CRUD methods**
  the domain needs (e.g. `AddItem`, `Checkout`). Pure CRUD needs none of this.
- The **backend**: `ent` or `gorm`. The CLI default is `gorm`; always pass `--backend` explicitly so there is no ambiguity.

State the plan back in one or two sentences before building.

## 2. Make the toolchain reproducible — pin the SDK version

Do **not** install with `@latest` (the proxy version list can lag and silently give you a
stale binary). Resolve the latest released tag and pin it explicitly:

```bash
# Resolve the newest released SDK version, then pin it for every fetch/install.
SDK_VERSION="$(go list -m -versions github.com/infobloxopen/devedge-sdk 2>/dev/null | tr ' ' '\n' | grep '^v' | tail -1)"
echo "Using devedge-sdk ${SDK_VERSION}"
GOPROXY=direct go install "github.com/infobloxopen/devedge-sdk/cmd/devedge-sdk@${SDK_VERSION}"
go version -m "$(command -v devedge-sdk)" | grep devedge-sdk   # confirm the pinned version
```

Ensure the rest of the prerequisites from the Installation page are on `PATH` (`go`, `buf`,
`apx`, and the base `protoc-gen-*` plugins). The scaffold calls `apx init app`, so `apx` and
`buf` must be present.

## 3. Scaffold the service

Use the one documented command (it produces an apx-governed proto, authz rules on every RPC,
generated persistence, a server, OpenAPI, deploy manifests, and a passing smoke test — no
hand edits for pure CRUD):

```bash
devedge-sdk new service <name> --resource <Owner> --backend <ent|gorm> \
  --module github.com/<org>/<name>
cd <name>
make test     # boots the server + a tenant-scoped CRUD round-trip — expect green
```

Confirm the generated `go.mod` requires the pinned `SDK_VERSION` and that `make tools`
installs the codegen plugins at that version (the Makefile derives it from `go.mod`).

## 4. Model the domain

Edit the generated proto for the owner's real fields, then add child resources and
**custom methods** if the domain needs them. Read the **"Add a custom method or second
resource"** how-to (`how-to/model-and-persist/custom-methods/`) for the wiring, and confirm
the relationship/projection annotations against the real `.proto` source (see "Where the
truth lives"). Custom RPCs each declare their own `(infoblox.authz.v1.rule)`. To expose them
alongside the generated CRUD, set the `Handler` option on the generated module — see the
gotchas below for the exact pattern. For a read-only rollup surface (e.g. a
`CartSummary`), use a multi-surface projection per the `codegen` "Multi-surface" reference;
surfaces are a subset of the owner's columns, so compute totals/counts in the custom RPC
response or the `FromEnt<R>Custom` hook, not as surface fields.

**Model fields the backend can store.** On both the ent and gorm backends, resource fields
must map to scalar columns. Common walls to know before you hit them:

- A `repeated string` field (e.g. `tags`) is **not** a column. Use `map<string,string>` for
  label/tag sets (stored as JSONB), or model the items as a child resource.
- A writable `google.protobuf.Timestamp` or nested message field is **not** directly storable
  as a writable column on either backend.
- An `OUTPUT_ONLY` field generates but the field-masked `Update` emits **no setter** for it,
  so a custom method cannot persist state through it (the write silently no-ops). Custom-method
  state that must be persisted — a status, a flag, a count — needs a **writable scalar** field
  (e.g. `bool`, `string`, or an enum), not an `OUTPUT_ONLY` timestamp.

The SDK's generate-time error messages are clear and actionable when a field cannot be mapped;
this note just warms you up so you plan the schema before hitting the walls blind. See the
"Model a resource" how-to on the docs site for the full field guide.

After each change: `make generate && make test`.

## 5. Run it and prove it works

Start the service and round-trip the real operations over HTTP (or gRPC) per the Quickstart
and the service's README — create the owner, exercise the custom methods, read it back, read
any projection, list. **Verify created resources come back with a non-empty `id`** (see the
Create gotcha below) and that a follow-up `Get` succeeds. Confirm the spine holds with no
hand-written code: tenant isolation (a cross-tenant read is denied/not-found), soft-delete,
etag preconditions, and fail-closed authz (an ungranted call is denied). Capture the
round-trip output for the user.

## 6. Hand off

Summarize for the user: the commands run, the resolved SDK version, what the service does,
how to run it, and the next steps from the docs (secret fields, search, deploy). Leave the
working service in place. **File any wall you hit** (a missing doc, an undocumented
convention, an SDK gap) as a devedge-sdk issue, or list it for the user to file.

## Gotchas to expect (verified on the cart build; check your SDK version)

These bit a previous run. They are cheap to avoid if you know them up front:

- **Adding a custom method or a second resource: use the generated module's `Handler` option.**
  The generated `<Service>Module` ships a first-class `<Svc>ServiceModuleOptions{ Handler: ... }`
  field. To expose custom methods, set that option — embed the generated `<Svc>CRUDHandler` in
  your own handler struct so the AIP-standard CRUD methods keep working, then implement your
  additional methods on the struct. The generated `module/module.go` contains a commented recipe
  showing the exact pattern. Do **not** hand-roll a parallel `servicekit.Module`; the option is
  the supported path.
- **`id` is auto-minted server-side — do not set it yourself.** The `id` field defaults to
  `SERVER_GENERATED` (UUID7). `Create` returns a populated `id` automatically; you do not need
  to call `uuid.NewString()`. If a caller-supplied id is ever required, declare the `USER_SETTABLE`
  id strategy in the proto annotation — that is the explicit, documented opt-in.
- **Over the HTTP gateway, `update_mask` is a query parameter, not a body field.** When an
  RPC maps `body: "<field>"`, only that field is read from the JSON body; the field mask goes
  on the URL: `PATCH /v1/<res>/<id>?update_mask=name,status`. Putting `update_mask` (or
  `updateMask`) in the body is silently ignored.
- **Over the HTTP gateway, a `body: "<resource>"` Create sends fields at the top level.** For
  a Create RPC whose gateway mapping is `body: "<resource>"`, the correct JSON body is the
  resource fields at the top level — `{"url":"…","title":"…"}`. Wrapping them under the field
  name — `{"bookmark":{"url":"…"}}` — is silently accepted (HTTP 200) but drops every user
  field, so the resource is created empty.

## Guardrails

- Pin the SDK version everywhere; never rely on `@latest`.
- Read the real `.proto`/`.go` source for exact API shapes; use fetched docs for prose only.
- Tenant and soft-delete scoping must come from the generated code — never hand-write a
  filterer or a tenant interceptor. If something forces you to, stop and tell the user.
- Prefer the documented path. If a documented step does not work, say so plainly and file it
  — do not paper over it with an undocumented workaround.

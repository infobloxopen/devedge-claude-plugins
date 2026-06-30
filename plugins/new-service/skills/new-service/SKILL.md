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
payments, …); you figure out the mechanics from the published docs and tooling.

**Authority for HOW to use the SDK is the published docs**, not guesses. The canonical
references — read them as you go, do not reinvent their contents:

- Getting started → Installation and Quickstart (prerequisites, the scaffold command).
- How-to → "Define a service" and "Add a custom method / second resource to a scaffolded
  service" (for any non-CRUD behavior).
- Reference → `servicekit` (the runtime the scaffold generates), `codegen` (multi-surface
  projections), `server`, `persistence`, `middleware`.

If the docs and the tools disagree, trust the tools and tell the user — that is a real
documentation bug worth filing.

## 1. Frame the domain with the user

From the user's one line, settle just enough to scaffold — ask only if genuinely unclear:

- The **service name** (a short noun, e.g. `cartd`) and Go **module path**.
- The **owner resource** (e.g. `Cart`) and its fields, including which need `unique`,
  `secret`, `soft-delete`, an `etag`, and tenant scoping (`account_id`).
- Any **child resources** (e.g. `CartItem` under `Cart`) and **custom, non-CRUD methods**
  the domain needs (e.g. `AddItem`, `Checkout`). Pure CRUD needs none of this.
- The **backend**: `ent` or `gorm` (default `ent` unless the user prefers GORM).

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
**custom methods** if the domain needs them. Custom RPCs each declare their own
`(infoblox.authz.v1.rule)` and are wired alongside the generated CRUD through the
`servicekit` module/host — **follow the "Add a custom method / second resource to a
scaffolded service" how-to exactly**; that page is the authority for the wiring the scaffold
expects (do not fall back to a bare `server.New` recipe). For a read-only rollup surface
(e.g. a `CartSummary`), use a multi-surface projection per the codegen "Multi-surface"
reference; surfaces are a subset of the owner's columns, so compute totals/counts in the
custom RPC response or the `FromEnt<R>Custom` hook, not as surface fields.

After each change: `make generate && make test`.

## 5. Run it and prove it works

Start the service and round-trip the real operations over HTTP (or gRPC) per the Quickstart
and the service's README — create the owner, exercise the custom methods, read it back, read
any projection, list. Confirm the spine holds with no hand-written code: tenant isolation
(a cross-tenant read is denied/not-found), soft-delete, etag preconditions, and fail-closed
authz (an ungranted call is denied). Capture the round-trip output for the user.

## 6. Hand off

Summarize for the user: the commands run, the resolved SDK version, what the service does,
how to run it, and the next steps from the docs (secret fields, search, deploy). Leave the
working service in place.

## Guardrails

- Pin the SDK version everywhere; never rely on `@latest`.
- Tenant and soft-delete scoping must come from the generated code — never hand-write a
  filterer or a tenant interceptor. If something forces you to, stop and tell the user.
- Prefer the documented path. If a documented step does not work, say so plainly and
  suggest filing it — do not paper over it with an undocumented workaround.

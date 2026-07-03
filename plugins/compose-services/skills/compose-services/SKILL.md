---
name: compose-services
description: >-
  Compose several devedge service modules into ONE host process — a k3s-style
  suite binary — and deploy it. Use this whenever the user wants to compose
  services into one binary, combine microservices into a suite, run several
  devedge modules in one process, build a control-plane binary from modules, or
  mentions "de compose" — for example "compose orders, inventory, and shipping
  into one control-plane binary", "combine my microservices into a suite", or
  "run these devedge modules in a single process". A devedge service is two
  artifacts — an importable module (owns the domain) and an executable host
  (owns the process); this skill drives the module → host → deploy flow.
argument-hint: <the member services/modules to compose, and a name for the suite binary>
---

# Compose devedge modules into one host binary

Take several devedge-sdk services and run them in **one** process — a suite binary — by
generating a new host that imports each service's **module** and runs them together via
`servicekit.Run`. This is static composition (imported, **not** Go plugins). The key idea:
a service is **two artifacts** — an importable **module** that owns domain behavior and exposes a
**composition seam** (`NewModule(db *gorm.DB) servicekit.Module` + `Models() []any`), and an
executable **host** that owns process behavior (`main`, env/flags, listeners, `os.Exit`). The host
opens one shared DB and builds every member through its `NewModule(db)` — `servicekit` is ORM-free,
so the host (not the module) owns the DB handle. The *same* module runs standalone or composed into a
suite; you change the host, not the module.

## Where the truth lives (read this first)

Do not guess flags or the composition-file shape — the CLI help is authoritative and always
matches the installed `de`:

- **`de compose <sub> --help`** — the exact flags, defaults, and steps for every subcommand
  (`init`, `add`, `remove`, `tidy`, `build`, `test`, `up`, `chart`). Read the one you're about
  to run before running it.
- **`servicekit` reference** on the devedge-sdk docs site,
  <https://infobloxopen.github.io/devedge-sdk/docs/> — the **Module / Descriptor / App** contract
  and `servicekit.Run`, plus the composition concept page. This is the source of truth for what a
  Module is and what a well-behaved module must (and must not) do.
- **The devedge portal**, <https://infobloxopen.github.io/devedge/> — where `de compose` fits in
  the workflow alongside `de new` and `de api publish`.

Generated **gorm** services expose the composition seam (`NewModule(db)/Models()` in
`module/compose.go`, emitted by `devedge-sdk new service` at **v0.52.0+**), so a service scaffolded on
a current SDK qualifies as a composition member with no changes. A service generated on an older SDK
must be regenerated to gain the seam. (ent composition is not yet supported — gorm only.)

## 1. Confirm each member is a composable module

Every member must be an **importable Go package that exposes the composition seam** —
`NewModule(db *gorm.DB) servicekit.Module` and `Models() []any` (in its `module/compose.go`) — so the
composed host can build it over one shared DB. Check each candidate:

- It was built with `devedge-sdk new service` at **v0.52.0+** (gorm backend) — a standalone service
  already qualifies; its `module` package exposes `NewModule`/`Models`. Older scaffolds must be
  regenerated on a current SDK to gain the seam.
- It owns **domain behavior only**. A module must **not** read env or flags, call `os.Exit`, or
  open its own listeners — the host owns all of that. If a service does any of these in code the
  module imports, that logic belongs in its `cmd/…/main.go` host, not the module.

Confirm the exact import path for each member (that is what `de compose add` takes).

## 2. Scaffold the composition

Create the `kind: Composition` file (default `composition.yaml`), which lists the member modules:

```bash
de compose init <suite-name>            # writes composition.yaml (override with -f/--file)
```

## 3. Add each member module

Add every member by its import path (optionally version-pinned `MODULE@VERSION`):

```bash
de compose add github.com/<org>/<svc>                       # a PUBLISHED member (by import path)
de compose add github.com/<org>/<svc> --path ../svc         # a LOCAL, unpublished member (writes a replace)
de compose add github.com/<org>/<svc>@v1.2.0 \
  --name orders --schema orders --config-prefix orders      # override name / DB schema / config namespace
```

Use **`--path <dir>`** to compose a **local, unpublished** module during development: it records a
`replace` so the composed binary builds without publishing the member first (the common case when you
are iterating on the members and the suite together). `--schema` sets the module's DB namespace
(defaults to `--name`); `--config-prefix` sets its config namespace (defaults to `--name`). Use
`de compose remove <name>` to drop a member.
Confirm flags with `de compose add --help`.

## 4. Tidy — catch conflicts BEFORE building

Validate the union of member descriptors and their versions **before** you generate anything:

```bash
de compose tidy        # unique IDs; no duplicate gRPC service / HTTP route prefix / permission
                       # names; coherent event graph; version compatibility
```

Modules linked into `de` (and its fixtures) are validated in-process; external modules `de`
doesn't link are reported as **unresolved** — that's expected, they still build statically in the
next step. Fix any reported descriptor conflict or version incompatibility here; it is far cheaper
than after `build`.

## 5. Build the static host

Generate the composed-binary sources — a `cmd/<name>/main.go` that imports the members and calls
`servicekit.Run`, a `go.mod` for the binary, and a `composition.lock` that pins the members + SDK
+ toolchain:

```bash
de compose build --module-path github.com/<org>/<suite-name>    # -o/--out to set the base output dir
```

No Go plugins are involved — the modules are compiled in. Commit `composition.lock`; it is the
reproducibility record for the suite.

## 6. Smoke-test, then build and test the Go

```bash
de compose test        # AssertComposition: validates the descriptor union, boots the composed
                       # host over it (the server's fail-closed completeness gate), shuts down
go build ./...         # in the generated cmd/<name> module
go test ./...
```

`de compose test` runs entirely in-process (no Docker) when there's no shared DB + migrations
configured; the real-DB path runs only when members declare migrations and a shared DB is set. It
**fails loud** (non-zero) when a member cannot be verified — it will not report a false pass for an
unresolved external module, so a green `de compose test` means the composed host actually built and
booted.

## 7. Run it and/or render deploy artifacts

- **Run / provision:** `de compose up` provisions the composition's shared database and registers
  the aggregated member routes through the edge (same resolve → provision → register sequencing as
  `de project up`). Add `--deploy` to also deploy the composed workload. The binary itself is what
  `de compose build` + `go build` produced.
- **Deploy artifacts:** `de compose chart` renders Helm artifacts. Topology is
  `single-binary` (default — one Deployment for the composed binary + one Ingress per member
  route), `multi-daemon` (a Deployment per member), or `hybrid` (composed binary except members
  marked `failurePolicy: dedicated-required` get their own Deployment). Set it via
  `spec.runtime.mode` or override with `--mode`; `-o/--out` sets the output dir.

## Gotchas to expect (confirm against your `de` via `--help`)

- **Descriptor conflicts across modules.** Two members that declare the same gRPC service name,
  HTTP route prefix, or permission name collide in the union. Module-qualify these names so they're
  unique; `de compose tidy` catches the collision before you build.
- **Version incompatibility across members.** Members built against incompatible SDK/codegen
  versions won't compose cleanly. `de compose tidy` validates version compatibility and
  `composition.lock` pins the resolved set — tidy first, build second.
- **DB isolation is module-namespace UNDER tenant.** Each module is namespaced (schema-preferred,
  `--schema`) beneath tenant (`account_id`) isolation; the framework tables (outbox / idempotency /
  migrations) are namespaced too, and the host runs each module's migrations. There are **no
  cross-module foreign keys** — modules share a database, not a schema.
- **Same-binary events go through the in-process dispatcher — not direct calls.** When composed,
  cross-module events are dispatched in-process (with bulkheads) rather than over outbox + CDC;
  they are still events, not direct function calls. Don't reach into another module's internals.
- **`failurePolicy` governs blast radius.** A member's `failurePolicy` (e.g. `fail-host` vs
  `degraded`, and `dedicated-required` for chart topology) decides whether its failure takes the
  host down or degrades just that member. Set it deliberately per member.
- **Modules must stay host-agnostic.** A member that reads env/flags, calls `os.Exit`, or opens its
  own listener breaks composition — the host owns process behavior. Keep that in `cmd/…/main.go`,
  not the imported module.

## Guardrails

- Read `de compose <sub> --help` for exact flags; never invent them. Read the `servicekit`
  reference for the Module/Descriptor contract.
- Always run `de compose tidy` before `de compose build` — resolve descriptor/version conflicts
  before generating the host.
- Keep **domain behavior in the module** and **process behavior in the host**; `main()` is a
  disposable host, and the same module must run standalone or composed unchanged.
- Static composition only — the members are **imported**, never loaded as Go plugins.
- If a documented step doesn't work, say so plainly and file it against `devedge-sdk` / `devedge` —
  do not paper over it with an undocumented workaround.

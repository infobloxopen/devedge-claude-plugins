---
name: model-domain
description: >-
  Model a devedge domain with DDD rigor. Use this whenever the user wants to add
  or relate domain objects, model a resource and its relationships, decide whether
  something should be an aggregate, reason about invariants and consistency
  boundaries, or expose read views/surfaces over a core model — for example "add a
  Subscription and relate it to Account", "should Order and OrderItem be one
  aggregate?", "what invariants belong on Cart", "how do I model this so shipped
  orders can't change", or "expose an admin view over the coupon model". The user
  brings the domain; this skill drives the modeling judgment — entities vs value
  objects, aggregate boundaries, references, and surfaces — onto real devedge
  mechanics. Distinct from new-service (bootstrap); composes with publish-api.
argument-hint: <the domain concept(s) to model and how they relate — e.g. "Order with line items; ship rule">
---

# Model a devedge domain (aggregates, invariants, surfaces)

Turn a domain into a devedge model with two clearly separated halves:

- **The write model** — resources clustered into **aggregates** that enforce **invariants**
  inside a transactional consistency boundary. This is where correctness lives.
- **The surfaces** — the read/API projections exposed *over* that core model (multiple API
  versions, read-only rollups, admin views, lookups). Several surfaces can sit over one
  backing model.

The framework already implements the Domain-Driven Design aggregate rules natively; your job
is the **judgment** — find the invariants, draw the smallest boundary, and decide what is an
aggregate vs. a cross-aggregate reference vs. a surface — then declare it with the real
annotations. Do not conflate "must be consistent together" (an aggregate) with "shown or
queried together" (a surface).

## Where the truth lives (read this first)

- **`concepts/aggregates.md`** on the devedge-sdk docs site
  (<https://infobloxopen.github.io/devedge-sdk/docs/>) — **authoritative**. devedge implements
  DDD aggregates directly: an aggregate root as the sole write entry point, members, the
  decision procedure, containment-vs-reference, the `Validate` invariant hook, the fail-closed
  boundary gate, and eventual cross-aggregate consistency via events. Read it before modeling.
- **The annotation source (GROUND TRUTH for exact shapes)** — after `go get`ing the SDK it is
  vendored at `$(go env GOMODCACHE)/github.com/infobloxopen/devedge-sdk@<ver>`; read the real
  proto:
  - `proto/infoblox/ddd/v1/ddd.proto` — `aggregate {root: true}`, `member {root: "<Root>"}`,
    `references {aggregate, foreign_key}`.
  - `proto/infoblox/field/v1/field.proto` — relationship options `has_one`, `has_many`
    (+ `position_field` for ordered), `belongs_to`, `many_to_many`.
  - `proto/infoblox/storage/v1/storage.proto` — the `model` option that binds several API
    surfaces to one backing model (the core-model-vs-surfaces primitive).
- **A worked example:** `testdata/iam` in the SDK — `Group` (root) owns `Membership` members
  (the "≥1 admin" invariant); `ApiKey` is its own aggregate that *references* `User` by ID;
  auth lookup is a read projection. Study it as the reference implementation.
- The `codegen` **Multi-surface** reference for read projections.

Read a paraphrase for prose, but confirm annotation shapes against the proto source — a
summarized annotation is not safe to build on.

## 1. Inventory the model you already have

Never add blind. Read the existing proto(s) and their annotations first: what resources exist,
which are aggregate **roots** (`ddd.v1.aggregate`), which are **members**, what
**references** already cross boundaries, which invariants are already encoded in `Validate`
hooks, and what surfaces (`storage.v1.model` projections) exist. New objects almost always
relate to this graph — you are extending it, not starting fresh.

## 2. Classify the new concept

- **Entity** — has a stable identity (an `id`) and a lifecycle; it is a resource. Most domain
  nouns are entities.
- **Value object** — no identity, defined only by its attributes, conceptually immutable (money,
  an address, a date range). Model it as **fields on an entity** (or a `map`/embedded), *not* as
  its own resource with an `id`. Giving a value object independent identity is a common
  over-modeling mistake.
- **A new aggregate** — an entity (plus any members) that owns an invariant. Decide this in §3.

## 3. Find the invariants and draw the aggregate boundary (the core decision)

Work the questions in order (from `concepts/aggregates.md`, with the DDD reasoning behind each):

1. **Is there an invariant spanning more than one resource that must hold within a single
   transaction?** A state rule, a count, a sum across a parent and children ("an item cannot be
   added once the order is SHIPPED", "a group keeps ≥1 admin"). If every resource is
   independently valid, you do **not** need an aggregate — plain CRUD resources are correct.
2. **What is the smallest set of resources that must be consistent in one transaction?** That
   set *is* the aggregate. Include **only** what the invariant requires. Small aggregates are a
   rule, not a preference — a big one loaded on every write kills concurrency and throughput.
3. **Root vs. member vs. its-own-aggregate.** The **root** is the only write entry point. A
   **member** is readable on its own but mutated only through the root, and has no life without
   it. If a thing has its own independent lifecycle and its own invariants, it is **its own
   aggregate** — link to it by **ID**, not by containment.
4. **Containment or reference?** *Containment* = the member dies with the root (cascade); keep a
   traversable `has_many`/`belongs_to` edge. A *reference* across aggregates is an **ID-only
   scalar FK with no traversable edge**, so code cannot walk or mutate across roots.

Two rules that override convenience:

- **`account_id` (tenant) is a partition, not an aggregate root.** It scopes queries; it does
  not enforce a cross-entity write invariant. Never model a tenant-wide "aggregate".
- **High-cardinality children are their own aggregate, referenced by ID.** An `Order` owning a
  bounded set of line items is a fine aggregate; a `Customer` "owning" all their orders is not —
  that is a reference + eventual consistency.

State the boundary decision back to the user in a sentence or two before you declare it.

## 4. Declare it in devedge

For a brand-new aggregate you can scaffold the wiring — `devedge-sdk new service <name>
--aggregate` generates a root that owns a member and wires the `TxRunner`,
`AggregateRepository`, and the transactional-outbox `Publisher`/`Dispatcher` in `main`; you then
model the specific resources, edges, and invariant. On an existing service, add the annotations
directly.

Map the decision onto the annotations (confirm shapes against `ddd.proto`/`field.proto`):

```protobuf
message Order {                                   // aggregate ROOT
  option (infoblox.ddd.v1.aggregate) = {root: true};
  string id = 1;
  string state = 2;
  repeated Item items = 3 [(infoblox.field.v1.opts) = {has_many: {foreign_key: "order_id"}}]; // containment
  string etag = 4 [(google.api.field_behavior) = OUTPUT_ONLY];   // the aggregate version
}
message Item {                                    // MEMBER — written through the root
  option (infoblox.ddd.v1.member) = {root: "Order"};
  string id = 1;
  string order_id = 2;
}
// Cross-aggregate link (ID only, no edge, never cascade):
//   User user = 5 [(infoblox.ddd.v1.references) = {aggregate: "User", foreign_key: "user_id"}];
```

- **Invariants live in the root's `Validate(ctx) error` hook** — a duck-typed method in an
  *owned, regen-safe* file (e.g. `order_behavior.go`). `Save` calls it before any persist; a
  violation rejects the whole `Save` with no partial write. Put the rule here, never in the
  client.
- **Load/Save the cluster as a unit** via the generated `AggregateRepository[Root, ID]`: `Load`
  eager-loads the containment edges; `Save` runs one `Atomically` transaction, tracks member
  add/remove/change, runs `Validate`, and bumps the root etag once (a stale etag →
  `ErrPreconditionFailed`).
- The generators emit cascade-on-delete for containment, an ID-only FK for `references`, and
  **member write-redirection** (a member's write RPCs become `Unimplemented`). At `Serve`,
  `AssertAggregateBoundaries` fails closed if a member registers a write method.

## 5. Choose the surfaces (the "vs surfaces" half)

Surfaces are **reads over the core model** — they carry no write authority and do not define
consistency. Decide them *separately* from the boundary:

- **Multiple API surfaces over one backing model** — set `(infoblox.storage.v1.model) = "Order"`
  on more than one message so `v1.Order`, `v2.Order`, and an admin view all project the same
  stored entity. A surface whose `model` differs from its message name is a projection, not a
  model of its own.
- **Read-only rollups / summaries** — a `CartSummary`-style projection is a subset of the
  owner's columns; compute totals/counts in the custom RPC response or a projection hook, **not**
  as a stored child. Needing a "total" is a read concern; it is not a reason to add a member.
- **Member read projections** — a `LookupBy<Field>Hash` or summary that shares a member's table
  is a read; it does not trip the boundary gate. (IAM resolves an API key by hashed secret via a
  projection, never by loading the owning aggregate.)
- **Public API surface** — the public API is decoupled from the internal model via apx; use the
  **publish-api** skill to expose and version it and to generate a typed client.

Litmus: *consistent together* → same aggregate; *shown/queried/versioned together* → a surface.

## 6. Consistency across aggregates

Each `Save` operates on exactly **one** root. There is no two-aggregate transaction by design.
Across aggregates, consistency is **eventual**: link by `references` (ID) and react with the
transactional-outbox domain-events seam (`events.Publisher` / `events.Dispatcher`; see the
`events` docs). If a rule seems to need two aggregates committed together, your boundary is
wrong — either it is really one aggregate, or the rule must tolerate eventual consistency.

## 7. Generate, then verify the model holds

`make generate`, then prove the modeling — not just that it compiles:

- The **boundary gate** passes at `Serve` (no member exposes a write method); removing a stray
  member write RPC (keeping `Get`/`List`) clears an `AssertAggregateBoundaries` failure.
- The **invariant** actually rejects: exercise the rule (add an item to a SHIPPED order) and
  confirm `Save` returns the mapped error with **no partial write**.
- The **etag precondition** works: a stale etag on the root → `ErrPreconditionFailed`.
- Reads through the surfaces return what they should; cross-aggregate reactions fire via events.

## Gotchas to expect

- **The #1 anti-pattern: an aggregate that eager-loads an unbounded `has_many` on every write.**
  If the child set is high-cardinality, it is its own aggregate referenced by ID — not a member.
- **Confusing a surface with an aggregate.** "We show these together / fetch them together" is a
  read model (a projection), not a consistency boundary. Do not add a child resource to display a
  rollup — compute it in the custom RPC or a projection. (Related: an `OUTPUT_ONLY` field emits no
  `Update` setter, and a surface is a *subset* of the owner's columns — so custom-method state
  that must persist needs a writable scalar on the root, not a projection field.)
- **Modeling a value object as a resource/aggregate.** No identity, no lifecycle → it is a field
  or embedded value, not an entity with an `id`.
- **Reaching past `Save` into the ent/GORM client directly** bypasses the boundary gate — the
  same risk class as bypassing the authz gate. All aggregate writes go through the root / `Save`.
- **`account_id` as an aggregate root.** It is the tenant partition; it never holds a
  cross-entity write invariant.
- **Wanting a two-aggregate transaction.** Impossible by design; it signals a wrong boundary or a
  rule that should be events-driven (eventual).

## Guardrails

- Read `concepts/aggregates.md` + the `ddd`/`field`/`storage` proto source before modeling. The
  framework encodes the DDD rules — apply them; don't reinvent or fight them.
- Model the **write model** (aggregates + invariants) and the **read surfaces** separately; never
  let a read convenience distort a consistency boundary.
- Keep aggregates small; one aggregate per transaction; cross-aggregate = `references` (ID) +
  events (eventual).
- Put invariants in the root's `Validate` hook (owned, regen-safe), enforced by `Save` — never in
  the client or a handler.
- Prefer the documented `ddd.v1` + `Validate` + `AggregateRepository` path. If the model forces a
  rule the framework cannot express, stop and say so — and file it against `devedge-sdk` rather
  than hand-wiring around the boundary.

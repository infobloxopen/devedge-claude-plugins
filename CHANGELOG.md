# Changelog

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

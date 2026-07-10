---
name: new-shell
description: >-
  Scaffold and run a single-spa SHELL that hosts your devedge micro-frontends and renders them behind a
  left side-nav menu. Use this whenever the user wants to host/compose their uFEs in a shell, "see my uFE
  render", create a devedge shell or root-config, add a nav menu, or asks for "de ufe shell" — for example
  "scaffold a shell for my uFEs", "host discovery and orders in one shell", "give me a shell with a nav
  menu", or "I ran de ufe new, now how do I see it render". It reads the kind:Shell roster `de ufe new`
  writes and generates a runnable shell with a left side-nav (one item per uFE, active-route highlight,
  signed-in user); each uFE mounts in the content area. `--preset-dir` is an extension point for overlaying
  a session provider, design system, and branded/grouped nav. Pairs with new-ufe (scaffold the uFEs first)
  and run-locally (the dev edge the shell routes through).
argument-hint: <optionally the shell.yaml roster to read and a name; or a --preset-dir preset to overlay>
---

# Scaffold a devedge shell (host your uFEs, with a nav menu)

`de ufe new` scaffolds a uFE but not a **shell** to mount it in — this closes that gap. `de ufe shell`
reads the `kind: Shell` roster `de ufe new` already wrote (`shell.yaml`) and generates a runnable
single-spa **shell**: a left side-nav that lists every uFE in the roster (title-cased label, active-route
highlight), the signed-in user in the header, and each uFE mounted into the content area. The result is a
self-contained "see it render" loop — no copying an example shell.

## 1. Prerequisites

- One or more uFEs already scaffolded + registered into `shell.yaml` (use the **new-ufe** skill; each
  `de ufe new` upserts itself into the roster).
- The devedge daemon/edge up (use **run-locally**: `sudo de start`; `de doctor` shows the edge checks
  PASS). A `read:packages` GitHub token for the open-core SDK on GitHub Packages: `export GITHUB_TOKEN=…`.

## 2. Scaffold the shell

```bash
de ufe shell                     # reads ./shell.yaml; writes <project>-shell/
# de ufe shell --shell notesapp-shell.yaml --name notesapp-shell
```

Generates the shell app: `index.html` (importmap + the left side-nav + a mount element per uFE),
`root-config.ts` (registers each roster uFE by hash route via the SDK's loader, active-nav highlight,
signed-in user), `package.json` (build+serve via esbuild + sirv on the roster's `shellUpstream` port),
`.npmrc` (GitHub Packages), `tsconfig`, `.gitignore`. Warns if the roster lists no uFEs.

## 3. Run it — and SEE IT RENDER

```bash
cd <project>-shell && pnpm install && pnpm start   # builds + serves the shell on its shellUpstream port
# in each uFE dir: pnpm install && pnpm start        # ng serve (binds 127.0.0.1 to match the edge)
de project up -f shell.yaml                         # route the shell + uFE bundles through the edge
```

Open `https://<host>.dev.test/#<route>` (the host from `shell.yaml`) in Chrome: the shell chrome + the
left side-nav render, and clicking a nav item mounts that uFE in the content area. **Don't stop until it
renders.** `*.dev.test` is mkcert-trusted and resolves through the devedge daemon.

## 4. Preset extension point (`--preset-dir`)

The shell is an **extension point**: `--preset-dir <path>` applies a preset overlay on top of the base
open shell, so a downstream can rebind the session provider, design system, and a branded/grouped nav
without forking — the same overlay seam `de ufe new --preset-dir` uses for uFEs.

```bash
de ufe shell --preset-dir <path-to-a-preset>   # overlay a session/design-system/nav preset
```

The open shell ships a flat "Applications" side-nav, generic OIDC (dev-session fallback), and Angular
Material; a preset can replace any of those. (The public open core ships no proprietary preset.)

## 5. Integrated mode — render your local uFE in a live shell

To develop your uFE locally but see it inside a **live hosted shell** (any shell that supports
`import-map-overrides`), no proxy:

```bash
de ufe override <ufe> --env https://<live-shell-url> [--namespace <ns>]
```

It serves your local bundle through the edge (`https://cdn.dev.test/<ufe>/main.js`, mkcert TLS + CORS) and
prints the exact `import-map-override` snippet to paste in the live shell's DevTools console. The live
shell cross-origin-fetches your local `main.js` and mounts it, inheriting the shell-owned session.

## Guardrails

- **Render is the gate** — a scaffolded shell isn't done until a uFE mounts and renders in the browser.
- **The shell owns the session; a uFE never authenticates** — it receives the read-only session as a prop.
- The uFE dev server must serve `main.js` with `Access-Control-Allow-Origin: *` and bind `127.0.0.1`
  (a `de ufe new` uFE does both) so the edge reaches it.
- Commit messages describe the change + intent only — never any AI/LLM attribution.

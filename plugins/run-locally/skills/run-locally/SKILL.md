---
name: run-locally
description: >-
  Bring a devedge service up on the local dev edge and reach it over its
  *.dev.test hostname. Use this whenever the user wants to run, start, expose,
  or serve a service locally with devedge; set up their local dev environment;
  install or start the devedge daemon; or debug why a service isn't reachable —
  for example "run my devedge service locally", "bring the service up on the
  edge", "start the devedge daemon", "set up my local dev environment", "expose
  my service locally", or "why can't I reach my service at *.dev.test". This
  skill drives install → daemon → register → reach → teardown, and diagnoses the
  common local-edge failures.
argument-hint: <service name/dir, or the host/port you want on the edge>
---

# Run a service on the local dev edge

Take a devedge service (or any local HTTP/gRPC workload) from "a process on a port" to
"reachable over `https://<host>.dev.test` with a trusted cert." The dev edge is a local
router the devedge daemon manages: it terminates TLS with a local CA and forwards each
`*.dev.test` host to a registered upstream. Your job is to get the daemon up, register the
service's routes, prove the round-trip, and diagnose it when the hostname does not resolve.

## Where the truth lives (read this first)

Do not guess flags or invent hostnames — the CLI help is authoritative and matches the
installed binary:

- **`de <cmd> --help`** for every command below — the exact flags, defaults, and what each
  command does. `de register --help` in particular documents path-prefix routing and
  host-collision behavior; read it before registering anything by hand.
- **Hosted docs (prose + rationale):** the devedge portal, <https://infobloxopen.github.io/devedge/>.
  Navigate from the home to **Getting Started / Installation**, the **CLI reference** (generated
  from the binary), and the **"Ship a full-stack feature"** tutorial. Use the portal for the
  documented flow and `de <cmd> --help` as the source of truth for flags.
- **A scaffolded service ships its own `devedge.yaml`** (from `de project init` / `de new service`).
  That file declares the routes; `de project up` reads it. Prefer it over ad-hoc registration.

## 1. First-time setup — install the daemon and configure the system

`de install` installs the devedge daemon and configures the host so the edge works: it sets up
DNS for `*.dev.test`, edits `/etc/hosts` as needed, and installs a local (mkcert-style) CA into
the system/browser trust stores so the edge's TLS certs are trusted. **This step needs elevated
privileges** (it touches `/etc/hosts` and the system trust store), so expect a `sudo` prompt.

```bash
de install     # first time on a machine; may prompt for sudo
```

Prerequisites: the `de` binary on `PATH`. If a service also runs in a local cluster, you need
`docker` + `k3d` + `kubectl` for the cluster path (step 3b). Confirm nothing else is claimed with
`de install --help` — as of this writing it takes only `-h`.

## 2. Start the daemon and verify the environment

```bash
de start       # start the devedge daemon
de status      # is the daemon up? (distinct from "is my route registered")
de doctor      # system health check: DNS, trust store, hosts file, daemon
```

Run `de doctor` whenever anything is off — it is the one command that tells you *which layer* is
broken (trust, DNS, or daemon) rather than making you guess. All three take only `-h`.

## 3a. Run the process and register its routes (the common case)

Start the service process, then put it on the edge. For a scaffolded devedge service, register
declaratively from `devedge.yaml`:

```bash
make run                     # (in the service dir) start the process the route points at
de project up                # register every route from ./devedge.yaml
de project up --watch        # OR: stay running and heartbeat the leases (Ctrl-C to release)
```

`de project up` reads `devedge.yaml` (override with `-f/--file`), and `--watch` keeps the leases
renewed for as long as the project is active. A scaffolded Go service routes at the app host under
its API-layout prefix — e.g. `app.dev.test/api/<domain>/v<major>/<resource>`, strip-prefixed so the
service keeps its own `/v1/...` paths.

For an **ad-hoc port** (no `devedge.yaml`), register a single route directly — flags per
`de register --help`:

```bash
de register api.myapp.dev.test http://127.0.0.1:3000            # HTTP upstream
de register app.dev.test http://127.0.0.1:8080 --path /api --strip-prefix   # path route on a shared host
de register postgres.myapp.dev.test 127.0.0.1:5432 --protocol tcp           # TCP (db/gRPC)
```

## 3b. For a workload in a local cluster (k3d)

If the service runs in Kubernetes rather than as a bare process, use the cluster path. `de cluster
create` makes a k3d cluster with ingress port-mapping and bootstraps it (mkcert CA, cert-manager
issuer, external-dns webhook); `de cluster watch` then auto-registers any `Ingress` annotated
`devedge.io/expose=true`:

```bash
de cluster create dev             # create + bootstrap a local k3d cluster
# (deploy your workload with an Ingress annotated devedge.io/expose=true)
de cluster watch dev              # auto-register/deregister ingress hosts as they appear
```

`de cluster bootstrap` re-runs the CA/issuer/webhook setup on an existing cluster; both refuse
non-local clusters unless `--force`.

## 4. Reach it and see the route

```bash
de ls                            # list active routes (add --json to script it)
de inspect app.dev.test          # details for one route (--path selects a path route)
curl -sS https://app.dev.test/api/<domain>/v1/<resource>   # round-trip over HTTPS, trusted cert
de ui                            # open the dashboard in a browser
```

The cert should validate with **no** `-k`; if curl complains about trust, that's a step-1 problem
(see Gotchas). Confirm the exact public path against the service's README / `devedge.yaml` and
`de inspect`.

## 5. Teardown

```bash
de project down                  # remove all routes for the project (reads devedge.yaml)
de project down --clean          # also destroy dependency data / a dedicated cluster
de stop                          # stop the daemon when you're done for the day
```

For an ad-hoc route, `de unregister <host>` removes all routes for the host (`--path` removes just
one). Leases from `de project up --watch` expire on their own once you Ctrl-C.

## Gotchas to expect (verify with `de doctor` / `--help` for your version)

- **"Daemon not running" vs. "route not registered" are different failures.** If the hostname
  doesn't resolve at all, check `de status` (daemon) then `de doctor` (DNS/trust). If it resolves
  but returns 502/404, the daemon is up but the route is missing or points at a dead upstream —
  check `de ls` / `de inspect` and confirm the process is actually listening on that port.
- **A cert-trust failure is quiet until `de install` sets up the CA.** Before `de install` runs,
  the edge presents a cert your system doesn't trust, so `curl` fails on verification (and browsers
  warn) even though routing works. Fix it by running `de install`, not by adding `-k`. `de doctor`
  reports the trust-store state.
- **Two services left on the default `app.dev.test` host collide.** `app.dev.test` is the default
  scaffolded host and the host's catch-all. Register two services there with no `--path` and the
  second shadows the first. Give one a distinct `--host` (at scaffold time or in `devedge.yaml`),
  or use path routing (`--path /api --strip-prefix`) — `de register --help` documents longest-prefix
  matching for a shared host.
- **`*.dev.test` DNS depends on step 1.** Resolution of `*.dev.test` is set up by `de install`; if
  a host won't resolve on a machine you didn't install on (or after an OS DNS reset), re-run
  `de install` and confirm with `de doctor`.
- **Leases expire.** Routes carry a lease TTL; without `de project up --watch` (or `de renew`) a
  route can lapse. If a route was there and vanished, re-run `de project up` or add `--watch`.

## Guardrails

- Read `de <cmd> --help` for exact flags before you run a command; never invent flags.
- Prefer the **declarative** `de project up` (from `devedge.yaml`) over hand-running `de register`.
  Reserve `de register` for ad-hoc ports with no project config.
- `de install` needs elevated privileges (hosts file + system trust store) — tell the user to
  expect a `sudo` prompt; don't try to work around trust with `curl -k`.
- When a route won't reach, distinguish the layers with `de status` (daemon) and `de doctor`
  (DNS/trust) before touching the route — don't re-register blindly.
- If a documented step doesn't work, say so plainly and file it against `devedge` — don't paper
  over it with an undocumented workaround.

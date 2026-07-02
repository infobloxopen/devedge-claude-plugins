# devedge plugins for Claude Code

Claude Code plugins for the [devedge](https://github.com/infobloxopen/devedge) developer
experience — skills for the common devedge operations, from bootstrapping a service on
[`devedge-sdk`](https://github.com/infobloxopen/devedge-sdk) to publishing its API, running it
on the local edge, scaffolding a micro-frontend, and composing services into one binary.

> These are **Claude Code** plugins (skills), not plugins for the devedge product itself.

## Install

```
/plugin marketplace add infobloxopen/devedge-claude-plugins
/plugin install new-service@devedge      # or publish-api@devedge, run-locally@devedge, …
```

Then just describe what you want to do — the matching skill triggers on intent. For example:

```
build a shopping cart microservice with devedge     # new-service
should Order and its line items be one aggregate?    # model-domain
publish the orders API and generate a typed client  # publish-api
run my service locally and reach it on the edge      # run-locally
scaffold a discovery uFE with devedge                # new-ufe
compose orders and inventory into one binary         # compose-services
```

Each skill drives its operation end to end — pinning versions, using the real `de`/`apx`
tooling, and following the published devedge docs and `--help` as ground truth at each step.

## Plugins

Each plugin is one skill for a common devedge operation. Install the ones you need
(`/plugin install <name>@devedge`), or all of them.

| Plugin | Skill | What it does |
|---|---|---|
| `new-service` | `new-service` | Bootstrap and build a microservice on devedge-sdk from a prompt. |
| `model-domain` | `model-domain` | Model the domain with DDD rigor — find invariants, draw aggregate boundaries, and expose read surfaces over the core model. |
| `publish-api` | `publish-api` | Publish a service's API as OpenAPI through apx and generate/consume a typed TS client (local hot-loop included). |
| `run-locally` | `run-locally` | Bring a service up on the local dev edge (`*.dev.test`), round-trip it, and diagnose why it isn't reachable. |
| `new-ufe` | `new-ufe` | Scaffold an Angular + single-spa micro-frontend wired to the devedge-ufe SDK and a `kind:Shell` roster. |
| `compose-services` | `compose-services` | Compose several service modules into one host process (a suite binary) and deploy it. |

## Updating

Third-party marketplaces do not auto-update. To pull the latest plugin versions:

```
/plugin marketplace update devedge
/plugin update
```

Plugins are versioned with semver in each plugin's `plugin.json` and released as git tags on
this repository.

## Layout

```
.
├── .claude-plugin/
│   └── marketplace.json          # the "devedge" marketplace catalog
└── plugins/
    └── <plugin>/                 # new-service, publish-api, run-locally, new-ufe, compose-services
        ├── .claude-plugin/
        │   └── plugin.json       # the plugin manifest (name, version, description)
        └── skills/
            └── <plugin>/
                └── SKILL.md      # the skill
```

## License

Apache-2.0.

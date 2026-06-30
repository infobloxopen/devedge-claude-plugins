# devedge plugins for Claude Code

Claude Code plugins for the [devedge](https://github.com/infobloxopen/devedge) developer
experience. Install once, then build microservices on
[`devedge-sdk`](https://github.com/infobloxopen/devedge-sdk) from a single prompt.

> These are **Claude Code** plugins (skills), not plugins for the devedge product itself.

## Install

```
/plugin marketplace add infobloxopen/devedge-claude-plugins
/plugin install new-service@devedge
```

After installing, in any directory (including an empty one) describe what you want to build:

```
build a shopping cart microservice with devedge
```

The `new-service` skill takes it from there: it pins the SDK version, scaffolds a
fail-closed gRPC/REST service, models your domain (owner resource, child resources, and any
custom methods), generates, builds, runs it, and round-trips the real operations — following
the published devedge-sdk docs at each step.

## Plugins

| Plugin | Skill | What it does |
|---|---|---|
| `new-service` | `new-service` | Bootstrap and build a microservice on devedge-sdk. |

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
    └── new-service/
        ├── .claude-plugin/
        │   └── plugin.json       # the new-service plugin manifest
        └── skills/
            └── new-service/
                └── SKILL.md      # the bootstrap skill
```

## License

Apache-2.0.

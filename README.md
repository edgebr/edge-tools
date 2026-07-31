# edge-tools

Internal **EDGE** team marketplace of Claude Code plugins for the design + development cycle.
This repository **is a Claude Code marketplace** — point Claude Code at it and install the plugins
you want.

```bash
/plugin marketplace add edgebr/edge-tools
```

## Plugins

| Plugin | What it does | Install |
|---|---|---|
| [`ds-helper`](plugins/ds-helper) | Design-system-bound design & development, integrated with Figma: build a screen (`design-builder`), turn a design into bound code (`figma-to-code`), verify code vs design (`figma-fidelity`). Design-system-agnostic via **profiles** — ships a Dell (DDS v3) profile. | `/plugin install ds-helper@edge-tools` |

> More plugins will live here alongside `ds-helper`, each installable independently.

## Structure

```
edge-tools/
├── .claude-plugin/marketplace.json   # marketplace manifest (lists the plugins)
├── plugins/
│   └── ds-helper/                    # a plugin (its own .claude-plugin/plugin.json)
└── README.md
```

Each plugin is self-contained under `plugins/<name>/` with its own manifest and README. Adding a
plugin = a new folder under `plugins/` + an entry in `.claude-plugin/marketplace.json`.

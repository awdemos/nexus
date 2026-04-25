# Nexus

> *Converge and conquer.*

Nexus is a high-performance, AI-native code editor forked from [Zed](https://github.com/zed-industries/zed). It preserves 100% upstream compatibility while converging your entire development stack — LLM routers, coding agents, and MCP servers — into a single, sovereign environment.

## Why Nexus?

Zed is an exceptional editor. Nexus takes that foundation and makes it the center of gravity for your own AI infrastructure:

- **Routage** — Your multi-armed bandit LLM router becomes the default inference backend
- **Pi** — Your personal coding agent speaks the Agent Client Protocol natively inside the agent panel
- **MCP Servers** — Your context servers (MCP-Server-App, Dagger, and more) are pre-wired out of the box
- **100% Upstream Compatible** — Rebase onto latest Zed in seconds, not hours

## Architecture

```
┌─────────────────────────────────────────┐
│              Nexus (UI)                 │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ │
│  │ Editor  │ │ Agent   │ │ Terminal │ │
│  │         │ │ Panel   │ │          │ │
│  └────┬────┘ └────┬────┘ └────┬─────┘ │
└───────┼───────────┼───────────┼───────┘
        │           │           │
        └───────────┴───────────┘
                    │
        ┌───────────┴───────────┐
        │  TensorZero / Merlin  │  ← LLM Gateway
        └───────────┬───────────┘
                    │
        ┌───────────┴───────────┐
        │      Routage          │  ← LLM Router (localhost:8080)
        │  (Multi-armed bandit) │
        └───────────────────────┘
```

## Build

Nexus builds exactly like Zed. If you can build Zed, you can build Nexus.

```bash
# macOS
script/bootstrap
cargo build --release

# Linux
script/bootstrap
cargo build --release
```

### Prerequisites

- Rust 1.80+
- Node.js 20+ (for bundled extensions)
- See [Zed's official docs](https://zed.dev/docs/development) for platform-specific requirements

## Staying Current with Upstream

Nexus uses a two-branch model to make upstream merges trivial:

- **`main`** — Fast-forward mirror of `zed-industries/zed`
- **`nexus`** — Your working branch with the rebrand + integrations

```bash
# One-command rebase
./script/sync-upstream
```

This fetches latest upstream, fast-forwards `main`, rebases `nexus` on top, and pushes both branches to origin. Conflicts are rare because Nexus only touches packaging metadata, config paths, and default settings.

## Integrations

### Routage (Default LLM Provider)

Routage is pre-configured as the default LLM provider in `assets/settings/default.json`:

```json
"language_models": {
  "openai_compatible": {
    "routage": {
      "api_url": "http://localhost:8080/v1",
      "available_models": [{ "name": "routage-router", "max_tokens": 128000 }]
    }
  }
}
```

Start Routage, then open the agent panel. All completions route through your bandit router.

### Pi (ACP Agent)

The Pi bridge lives in your parent workspace at `../pi-acp-bridge`. Build it:

```bash
cd ../pi-acp-bridge
cargo build --release
```

Then select **Pi** from the agent panel dropdown.

### MCP Context Servers

Edit `assets/settings/default.json` and update the `context_servers` block with your local paths:

```json
"context_servers": {
  "mcp-server-app": {
    "command": "python3",
    "args": ["/path/to/mcp-server-app/run_server.py"]
  },
  "dagger-mcp": {
    "command": "/path/to/dagger-mcp-server",
    "args": []
  }
}
```

## Product Suite

Nexus is part of a coherent sovereign computing stack:

| Project | Role |
|---------|------|
| **Nexus** | IDE — this repo |
| [RegicideOS](https://github.com/awdemos/RegicideOS) | AI-native Linux distribution |
| [Merlin](https://github.com/awdemos/merlin) / [Routage](https://github.com/awdemos/routage) | LLM routers |
| [Pi](https://github.com/awdemos/pi) | Coding agent |
| [Memento](https://github.com/awdemos/opencode-memento) | Coding memory/context |

## License

Nexus inherits Zed's licensing: GPL-3.0-or-later for the editor core. See individual crates for their specific licenses.

## Contributing

This is a personal fork. Issues and PRs are welcome, but the primary goal is upstream compatibility — keep changes surgical.

# NoFray Plugin Marketplace

The official public marketplace for the NoFray plugin for Codex and Claude Code.

It contains only the distributable plugin bundle. The NoFray application and
its users' workspace data are not included.

## Install

```sh
codex plugin marketplace add yvos/nofray-plugin-marketplace
codex plugin add nofray@nofray-marketplace
```

The plugin connects only to a NoFray MCP server running locally on the user's
Mac at `127.0.0.1`. See [the plugin instructions](AgentPlugin/README.md) for
the full setup, authentication, and supported-client guidance.

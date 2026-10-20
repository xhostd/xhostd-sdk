# xhostd SDK

This repository publishes the [xhostd](https://xhostd.com) plugin for coding
agents. xhostd is hosting for coding agents: your agent creates an app, pushes
its code, and deploys it to an HTTPS URL.

The plugin works in Claude Code, Codex, Cursor, and ChatGPT. It carries one
remote MCP server and the `xhostd` skill.

## Contents

- `plugins/xhostd/`: the plugin itself. Its
  [README](plugins/xhostd/README.md) covers installation in each client,
  sign-in, what the plugin sends and runs, privacy, and support.
- `.claude-plugin/marketplace.json`: the Claude Code marketplace.
- `.cursor-plugin/marketplace.json`: the Cursor marketplace.
- `.agents/plugins/marketplace.json`: the Codex marketplace.
- `server.json`: the entry in the
  [MCP registry](https://registry.modelcontextprotocol.io).

## Documentation

The [xhostd guides](https://docs.xhostd.com/guides) hold one complete recipe
for each shape of app. The plugin carries the same recipes under
`plugins/xhostd/skills/xhostd/references/`.

## License

MIT. See [LICENSE](LICENSE).

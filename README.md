# xhostd SDK

This repository publishes the [xhostd](https://xhostd.com) plugin for coding
agents. xhostd is hosting for coding agents: your agent creates a project, pushes
its code, and deploys it to an HTTPS URL.

The plugin works in Claude Code, Codex, Cursor, and ChatGPT. It carries one
remote MCP server and eight skills: the core `xhostd` skill and one skill for
each topic, such as git, data, and custom domains.

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
for each shape of project. Each guide's raw Markdown is at its address with
`.md` added, and the plugin's skills link to the recipes there.

## License

MIT. See [LICENSE](LICENSE).

# SaaS Custom Domains for coding agents

The official [SaaS Custom Domains](https://saascustomdomains.com) plugin for Claude Code, Codex, Cursor, VS Code and other coding agents. It gives your agent two things:

- **The `saas-custom-domains` skill**, which teaches the agent how to add custom domains to your app: the API, the DNS records your customers need, webhooks, and how to verify requests that come through the proxy.
- **The [MCP server](https://docs.saascustomdomains.com/mcp)** at `https://mcp.saascustomdomains.com/mcp`, so the agent can add your customers' domains, look up their DNS records, check why a domain isn't live and read traffic.

Setup guides for each agent: https://saascustomdomains.com/agent-setup

## Install

Where an agent supports plugins, the plugin installs the skill and connects the MCP server in one step. The MCP server asks you to sign in to your SaaS Custom Domains account the first time it's used.

### Claude Code

```
/plugin marketplace add saas-custom-domains/skills
/plugin install saas-custom-domains@saas-custom-domains
```

Then run `/mcp`, pick **saas-custom-domains** and choose **Authenticate** to sign in. To get updates later, run `/plugin marketplace update saas-custom-domains`.

### Codex

```sh
codex plugin marketplace add saas-custom-domains/skills
```

Then run `/plugins` in Codex, install **SaaS Custom Domains**, and start a new session.

### VS Code / GitHub Copilot

1. Enable `chat.plugins.enabled` in VS Code settings.
2. Open the Command Palette and run **Chat: Install Plugin From Source**.
3. Enter `https://github.com/saas-custom-domains/skills`.

### Cursor and other agents

Install the skill with the [`npx skills`](https://skills.sh) CLI:

```sh
npx skills add https://github.com/saas-custom-domains/skills
```

Then connect the MCP server using the steps for your client in the [MCP docs](https://docs.saascustomdomains.com/mcp).

### Copy the skill by hand

Copy `skills/saas-custom-domains` into your agent's skills directory:

| Agent | Skills directory |
| --- | --- |
| Claude Code | `~/.claude/skills/` |
| Cursor | `~/.cursor/skills/` |
| Codex | `~/.codex/skills/` |
| OpenCode | `~/.config/opencode/skills/` |

## What's inside

| Path | What it is |
| --- | --- |
| `skills/saas-custom-domains/SKILL.md` | The skill. The same file is published at [docs.saascustomdomains.com/.well-known/agent-skills](https://docs.saascustomdomains.com/.well-known/agent-skills/index.json). |
| `.mcp.json`, `mcp.json`, `mcp_config.json` | The MCP server, in the formats different agents read. |
| `.claude-plugin/`, `.codex-plugin/`, `.cursor-plugin/`, `.agents/`, `.antigravity-plugin/`, `plugin.json` | Plugin and marketplace manifests. |

## MCP server

The server is remote, so there's nothing to install. Connect by signing in with your SaaS Custom Domains account, or send an [API token](https://app.saascustomdomains.com/api_tokens) as `Authorization: Bearer <token>`. Setup for each client and the full tool list: https://docs.saascustomdomains.com/mcp

## More for agents

- [llms.txt](https://docs.saascustomdomains.com/llms.txt) and [llms-full.txt](https://docs.saascustomdomains.com/llms-full.txt): the API docs in markdown
- [openapi.json](https://docs.saascustomdomains.com/openapi.json): the OpenAPI spec
- [auth.md](https://docs.saascustomdomains.com/auth.md): how agents get API credentials
- [scd-mock](https://github.com/saas-custom-domains/scd-mock): a local mock of the API and proxy, for development and CI
- [Example apps](https://github.com/saas-custom-domains): runnable custom domains examples for 13 frameworks

## Contributing

The skill is maintained alongside the docs and copied here by a daily job, so fix it there: open an issue in this repo or email support@saascustomdomains.com.

## License

[MIT](LICENSE)

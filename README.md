# Anti-Detect Browser — Claude Code Skills

[Claude Code custom skills](https://docs.anthropic.com/en/docs/claude-code/skills) that teach Claude how to launch and manage anti-detect browsers with real-device fingerprints, both by writing code against the SDK and by letting an agent drive the browser directly via MCP.

## Skills in this repo

- **anti-detect-browser** — SDK, profiles, fingerprints, proxies, and the REST API. Use this to write custom scraping, multi-account, or automation scripts.
- **browser-mcp-agent** — MCP server mode. Use this to let an AI agent (Claude, GPT, etc.) launch and control the browser itself via tool calls, with no code to write.

## Install

```bash
# SDK / scripting skill
npx skills add https://github.com/antibrow/anti-detect-browser-skills --skill anti-detect-browser

# MCP agent-driven skill
npx skills add https://github.com/antibrow/anti-detect-browser-skills --skill browser-mcp-agent
```

Or add the repo URL in Claude Code settings under **Skills** and pick which skill(s) to install.

## What Claude learns

From **anti-detect-browser**:

- **SDK usage** — `AntiDetectBrowser` class, `launch()` options, `applyFingerprint()` for existing Playwright setups
- **Profile management** — persistent browser identities with cookies, storage, and fingerprint binding
- **Fingerprint filtering** — tags, browser version ranges, screen size constraints
- **Proxy integration** — per-browser proxy routing
- **Visual identification** — floating labels, window titles, theme colors for multi-window workflows
- **Live View** — real-time headless browser streaming to the dashboard
- **REST API** — all public `/api/v1/` endpoints for fingerprints and profiles

From **browser-mcp-agent**:

- **MCP server setup** — running `anti-detect-browser` as a tool server for AI agents
- **Available tools** — `launch_browser`, `navigate`, `click`/`fill`, `screenshot`, `get_content`, Live View controls, and more
- **Agent-driven workflows** — example task flows with no user-written code

## Repo structure

```
anti-detect-browser/
  SKILL.md          # SDK and REST API reference
browser-mcp-agent/
  SKILL.md          # MCP server mode reference
```

## Related

- npm package: [anti-detect-browser](https://www.npmjs.com/package/anti-detect-browser)
- Dashboard & docs: https://antibrow.com

---
name: browser-mcp-agent
description: Let an AI agent (Claude, GPT, etc.) directly launch, navigate, and interact with a real anti-detect browser through MCP tool calls, with no Playwright or SDK code to write. Use when the user wants an agent to drive a browser itself - navigate, click, fill forms, take screenshots, extract page content - while keeping a consistent per-profile device fingerprint across the session, or wants a computer-use / browser-use style setup backed by a real captured device fingerprint instead of a synthetic headless browser. Also use when the user mentions 'MCP browser', 'browser automation agent', 'browser-use MCP', 'let my agent browse the web', 'agent browser control', 'browserbase alternative', or 'steel browser alternative'. For writing custom scraping, multi-account, or automation code directly against the SDK or REST API, see anti-detect-browser.
---

# Browser MCP Agent

Run antibrow as an MCP server so an AI agent can launch and control a real, fingerprinted browser directly through tool calls - no Playwright code, no custom automation script. The agent navigates, clicks, fills forms, and reads pages itself.

- npm package: `anti-detect-browser`
- Dashboard: `https://antibrow.com`
- Full SDK / REST API reference: see the `anti-detect-browser` skill

## When to use

- **Agent-driven browsing** - the agent itself should navigate a site, log in, click through a flow, or extract content, without anyone writing automation code first
- **Computer-use / browser-use style setups** - the same idea as generic "agent controls a browser" tools, but backed by a real captured device fingerprint rather than a synthetic headless browser
- **Ad-hoc one-off tasks** - "go check my dashboard and tell me X" requests where writing a script would be overkill
- **Debugging agent browser actions** - watch what the agent is doing in real time via Live View while it works

## Setup

```json
{
  "mcpServers": {
    "anti-detect-browser": {
      "command": "npx",
      "args": ["anti-detect-browser", "--mcp"],
      "env": { "ANTI_DETECT_BROWSER_KEY": "your-api-key" }
    }
  }
}
```

Get your API key at `https://antibrow.com` (free tier includes 2 browser profiles).

## Available tools

| Tool | What it does |
|------|-------------|
| `launch_browser` | Start a new fingerprint browser session |
| `close_browser` | Close a running session |
| `navigate` | Go to a URL |
| `screenshot` | Capture the current screen |
| `click` / `fill` | Interact with page elements |
| `evaluate` | Run JavaScript on the page |
| `get_content` | Extract text from the page or a specific element |
| `start_live_view` | Stream the browser screen to the `https://antibrow.com` dashboard |
| `stop_live_view` | Stop live streaming |
| `list_sessions` | List all running browser instances |
| `list_profiles` | List all saved profiles |

## Example: agent-driven task

A typical agent-driven flow, with no code written by the user:

1. Agent calls `launch_browser` with a fingerprint tag (e.g. `Windows 10` + `Chrome`) and a profile name
2. Agent calls `navigate` to the target URL
3. Agent calls `get_content` or `screenshot` to read the page
4. Agent calls `click` / `fill` to interact, repeating navigate/read as needed
5. Agent calls `close_browser` when done - the profile's cookies and storage persist under the same profile name for next time

## Related Skills

- **anti-detect-browser** - full SDK and REST API reference for writing custom Playwright-based automation, scraping, and multi-account scripts directly
- **antibrow dashboard** (`https://antibrow.com`) - manage profiles, watch Live View sessions, get your API key

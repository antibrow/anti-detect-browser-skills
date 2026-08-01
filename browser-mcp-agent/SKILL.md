---
name: browser-mcp-agent
description: Let an AI agent (Claude, GPT, etc.) directly launch, navigate, and interact with a real anti-detect browser through MCP tool calls, with no Playwright or SDK code to write. Use when the user wants an agent to drive a browser itself - navigate, click, fill forms, take screenshots, extract page content - while keeping a consistent per-profile device fingerprint across the session, or wants a computer-use / browser-use style setup backed by a real captured device fingerprint instead of a synthetic headless browser. Also use when the user mentions 'MCP browser', 'browser automation agent', 'browser-use MCP', 'let my agent browse the web', 'agent browser control', 'browserbase alternative', or 'steel browser alternative'. Runs on Windows x64, macOS (Intel + Apple Silicon), and Linux x64 / arm64, from Node (npx) or Python. For writing custom scraping, multi-account, or automation code directly against the SDK or REST API, see anti-detect-browser.
---

# Browser MCP Agent

Run antibrow as an MCP server so an AI agent can launch and control a real, fingerprinted browser directly through tool calls - no Playwright code, no custom automation script. The agent navigates, clicks, fills forms, and reads pages itself.

- npm package: `anti-detect-browser` (Node >= 18) - ships the MCP server built in
- PyPI package: `antibrow` (Python 3.9 - 3.13) - `pip install "antibrow[mcp]"` for a stdio MCP server example
- Dashboard: `https://antibrow.com`
- Full SDK / REST API reference: see the `anti-detect-browser` skill

## Why this over a generic browser MCP

Generic "agent controls a browser" servers hand the agent a stock or patched headless Chromium. Every page the agent visits sees the tells: a `navigator` override that is not `[native code]`, a canvas hash that changes on every read, a worker thread disagreeing with the main thread, a headless build's own fingerprint. antibrow's spoofing happens **inside the Chromium kernel**, so the agent gets a browser whose Canvas, WebGL, WebGPU, audio, fonts, screen and timezone all agree - and whose TLS ClientHello and HTTP/2-3 behaviour are a genuine Chrome build's, because it is one. Sessions also **persist**: the agent logs in once under a profile name and stays logged in.

## Platform support

Windows 10/11 x64 · macOS 12+ (universal build, Apple Silicon + Intel) · Linux x64 and **arm64** (glibc) · Docker `linux/amd64` and `linux/arm64`. The correct kernel build is picked from the CPU automatically. Alpine/musl is not supported yet.

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

Get your API key at `https://antibrow.com` - the free key gives 1 concurrent browser and unlimited local profiles. The browser kernel downloads on first launch (~190 MB, ~320 MB for the macOS universal bundle) and is cached under `~/.anti-detect-browser/`.

### Python

If the agent stack is Python, `pip install "antibrow[mcp]"` and run the stdio MCP server from `python/examples/09_mcp_server.py` in `https://github.com/antibrow/antibrow`:

```json
{
  "mcpServers": {
    "antibrow": {
      "command": "python",
      "args": ["/abs/path/to/examples/09_mcp_server.py"],
      "env": { "ANTIBROW_API_KEY": "your-api-key" }
    }
  }
}
```

It exposes `launch_browser`, `navigate`, `click`, `fill`, `get_content`, `screenshot`, `evaluate` and `close_browser`. Both SDKs share one cache directory and one profile format, so a profile created from Node is drivable from Python with the identical fingerprint. The Node server is the fuller of the two - prefer it unless the deployment must be Python-only.

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
| `list_profiles` / `create_profile` | Manage persistent browser identities |
| `list_proxies` / `claim_proxy` | Managed residential proxies (paid plans) |
| `start_live_view` | Stream the browser screen to the `https://antibrow.com` dashboard |
| `stop_live_view` | Stop live streaming |
| `list_sessions` | List all running browser instances |

## Example: agent-driven task

A typical agent-driven flow, with no code written by the user:

1. Agent calls `launch_browser` with a fingerprint tag (e.g. `Windows 10` + `Chrome`) and a profile name
2. Agent calls `navigate` to the target URL
3. Agent calls `get_content` or `screenshot` to read the page
4. Agent calls `click` / `fill` to interact, repeating navigate/read as needed
5. Agent calls `close_browser` when done - the profile's cookies and storage persist under the same profile name for next time

## Operational notes

- **Concurrency is kernel-enforced.** The plan caps how many browsers run at once (free = 1) via cross-process file locks; an agent that forgets `close_browser` will block the next `launch_browser`. Have the agent close sessions it is done with.
- **Profiles are unlimited and free** - one per account/task is the right granularity, not one shared session.
- **Headless is not the stealthy option.** Real headless Chromium has its own fingerprint. On Windows the window is moved off-screen instead; on Linux/Docker run headful under Xvfb.
- **Timezone follows the proxy** when a proxy is set, so an agent browsing through a US exit does not report a local clock.

## Related Skills

- **anti-detect-browser** - full SDK and REST API reference for writing custom Playwright-based automation, scraping, and multi-account scripts directly
- **multi-account-isolation** - the checklist for keeping accounts from being linked when an agent operates several of them
- **antibrow dashboard** (`https://antibrow.com`) - manage profiles, watch Live View sessions, get your API key

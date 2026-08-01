# Anti-Detect Browser - Claude Code Skills

[Claude Code custom skills](https://docs.anthropic.com/en/docs/claude-code/skills) for running **multiple accounts from one machine without them getting linked or banned** - and for anything else that needs a browser which does not look automated. They teach Claude how to launch and manage [AntiBrow](https://antibrow.com) anti-detect browsers with real-device fingerprints, both by writing code against the SDK (JavaScript **or** Python) and by letting an agent drive the browser directly via MCP.

## Skills in this repo

- **multi-account-isolation** - the operational checklist for keeping accounts unlinked: per-account profile + proxy + timezone pairing, warm-up, verification, and the leaks that survive a perfect fingerprint (IP, cookies, payment, recovery contacts, behaviour). Start here if the question is "how do I stop my accounts being associated".
- **anti-detect-browser** - SDK (npm `anti-detect-browser` + PyPI `antibrow`), profiles, fingerprints, proxies, kernel updates, Docker, and the REST API. Use this to write custom scraping, multi-account, or automation scripts.
- **browser-mcp-agent** - MCP server mode. Use this to let an AI agent (Claude, GPT, etc.) launch and control the browser itself via tool calls, with no code to write.

## Install

### As a Claude Code plugin (recommended)

Installs both skills together and keeps them updatable via `/plugin update`:

```
/plugin marketplace add antibrow/anti-detect-browser-skills
/plugin install anti-detect-browser-skills@antibrow
```

### As individual skills

Works with Claude Code and any other agent the `skills` CLI supports:

```bash
# account-isolation checklist
npx skills add https://github.com/antibrow/anti-detect-browser-skills --skill multi-account-isolation

# SDK / scripting skill
npx skills add https://github.com/antibrow/anti-detect-browser-skills --skill anti-detect-browser

# MCP agent-driven skill
npx skills add https://github.com/antibrow/anti-detect-browser-skills --skill browser-mcp-agent

# both
npx skills add https://github.com/antibrow/anti-detect-browser-skills --all
```

Or add the repo URL in Claude Code settings under **Skills** and pick which skill(s) to install.

## What makes this different from a stealth plugin

Most stealth tooling patches JavaScript from the outside: override a getter, shim `navigator`, monkey-patch `toString`. Anti-bot vendors have been fingerprinting those patches for years - the patch *is* the tell.

AntiBrow ships a modified Chromium kernel. Canvas, WebGL, WebGPU, audio, fonts, `navigator`, screen, DOMRect and timezone are answered inside C++/Blink, so:

- there is **no injected script** to find and no property descriptor out of place;
- **Web Workers return exactly what the main thread does** (detectors re-read identity inside workers precisely because partial overrides only patch the main thread);
- **canvas and WebGL reads are deterministic per profile** - a browser that returns a new hash on every read is trivially flagged, and so is one whose OffscreenCanvas disagrees with its 2D canvas;
- **WebGL, WebGL2 and WebGPU name one GPU** - `adapter.info.vendor` matches the unmasked renderer family;
- the **TLS ClientHello and HTTP/2-3 behaviour are a genuine Chrome build's**, because it is one. Nothing running in JavaScript can reach that layer.

Each profile gets **one coherent persona sampled from one real machine** - 30+ categories, 500+ parameters - frozen at creation and replayed on every later launch. Independently randomized values contradict each other (an AMD renderer beside an Intel vendor string, a 1.0 DPR on a 1536x864 screen); a real device's don't.

Timezone, locale and WebRTC identity **follow the proxy**: the exit IP is resolved *through* the proxy before launch. Proxy credentials are answered inside the network stack (HTTP 407, SOCKS5 RFC 1929), so nothing shows up in `chrome://extensions`.

Want to check any of that yourself: [CreepJS](https://abrahamjuliot.github.io/creepjs/), [whoer.net](https://whoer.net), [browserleaks.com/canvas](https://browserleaks.com/canvas), [pixelscan.net](https://pixelscan.net), or `npx liarjs` ([liarjs.dev](https://liarjs.dev)) for ~40 open-source cross-layer consistency rules you can run in CI.

## Platforms

| Platform | Status |
|---|---|
| Windows 10/11 x64 | Supported |
| **macOS 12+ (Apple Silicon + Intel)** | Supported - universal build |
| Linux x64 (glibc) | Supported |
| **Linux arm64 (glibc)** | Supported - separate arm64 kernel, auto-selected from the CPU |
| Docker `linux/amd64` + `linux/arm64` | Supported - headful under Xvfb |
| Linux musl (Alpine) | Not yet |

## Two SDKs, one profile format

```bash
npm install anti-detect-browser playwright-core     # Node >= 18
pip install antibrow                                # Python 3.9 - 3.13
```

```typescript
const ab = new AntiDetectBrowser({ key: process.env.ANTI_DETECT_BROWSER_KEY })
const { page, browser } = await ab.launch({ profile: 'shopper-01' })
```

```python
from antibrow import launch
browser = launch(profile="shopper-01")   # same profile, same fingerprint
page = browser.new_page()
```

Both share `~/.anti-detect-browser/`, so a profile created from Node is launchable from Python (and from the desktop app) with the identical identity. `playwright install` is never needed - AntiBrow drives its own kernel.

## What Claude learns

From **multi-account-isolation**:

- **The linkage surface** - every layer a platform can use to tie two accounts together, and which ones a browser can and cannot isolate
- **One account, one of everything** - profile, fingerprint, proxy, timezone, identity data, all unshared
- **Proxy selection** - sticky vs rotating, residential vs datacenter, matching the account's claimed location
- **Warm-up and verification** - aging accounts, and checking the setup against CreepJS, whoer, browserleaks, pixelscan, liarjs
- **Troubleshooting order** - the cheap causes (reused profile, shared IP, timezone mismatch, shared recovery email) before blaming the fingerprint

From **anti-detect-browser**:

- **JS/TS SDK** - `AntiDetectBrowser`, `launch()` options, `applyFingerprint()` for existing Playwright setups, kernel update APIs
- **Python SDK** - `launch()` / `launch_async()` / `launch_persistent_context()` / `prepare_launch()`, the `Antibrow` handle, error types, `python -m antibrow` CLI, env vars, Docker
- **Framework integrations** - hand the CDP endpoint to browser-use, crawl4ai, Scrapling, Puppeteer, or plain Playwright
- **Profile management** - persistent identities with cookies, storage, and a frozen persona
- **Detection model** - the cross-layer consistency checks that actually decide whether a browser passes
- **Proxies** - native `http`/`https`/`socks5`/`relay` auth, geo-matched timezone and WebRTC
- **Visual identification** - floating labels, window titles, theme colors for multi-window workflows
- **Live View** - real-time headless browser streaming to the dashboard
- **Plans, concurrency, and licensing** - kernel-enforced concurrent-browser caps; MIT SDK vs closed-source kernel
- **REST API** - all public `/api/v1/` endpoints for fingerprints and profiles

From **browser-mcp-agent**:

- **MCP server setup** - `npx anti-detect-browser --mcp`, or a Python stdio server via `antibrow[mcp]`
- **Available tools** - `launch_browser`, `navigate`, `click`/`fill`, `screenshot`, `get_content`, profile and proxy tools, Live View controls
- **Agent-driven workflows** - example task flows with no user-written code, plus the operational gotchas (concurrency locks, headless, session hygiene)

## Repo structure

```
.claude-plugin/
  plugin.json       # plugin manifest (all three skills)
  marketplace.json  # lets this repo be added as a Claude Code marketplace
multi-account-isolation/
  SKILL.md          # account-linking checklist: profiles, proxies, timezone, warm-up
anti-detect-browser/
  SKILL.md          # SDK (JS + Python) and REST API reference
browser-mcp-agent/
  SKILL.md          # MCP server mode reference
```

## Related

- npm package: [anti-detect-browser](https://www.npmjs.com/package/anti-detect-browser)
- PyPI package: [antibrow](https://pypi.org/project/antibrow/)
- SDK source (MIT): [github.com/antibrow/antibrow](https://github.com/antibrow/antibrow)
- Dashboard & docs: https://antibrow.com

The SDKs are MIT. The browser kernel is a closed-source binary downloaded at runtime under its own license - usable for your own work at any company size, but not redistributable; see `BINARY-LICENSE.md` in the SDK repo.

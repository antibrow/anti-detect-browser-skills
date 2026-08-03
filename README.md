# Anti-Detect Browser - Claude Code Skills

[Agent Skills](https://agentskills.io) for driving a browser that presents **one coherent real device** instead of a headless build: persistent isolated profiles, kernel-level fingerprints, and a per-profile proxy whose exit IP sets timezone and WebRTC. They teach an agent how to launch and manage [AntiBrow](https://antibrow.com) browsers, both by writing code against the SDK (JavaScript **or** Python) and by letting the agent drive the browser directly via MCP.

Typical uses: QA and bot-detection testing against your own site, checking your own ads and pricing from another region, scraping public data, giving an agent a browser that stays logged in, and keeping identities you own or are authorized to operate in genuinely separate profiles. See [Acceptable use](#acceptable-use).

## Skills in this repo

- **multi-account-isolation** - verifying that profiles are actually isolated rather than assuming it: ten concrete checks (timezone vs exit IP, WebRTC, canvas stability across relaunches, duplicate personas or addresses across a fleet), which detection suites to run, what the runtime does with your credentials, and which layers browser isolation cannot cover at all. Start here if the question is "is my setup actually holding?".
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
# isolation self-check
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
npm install anti-detect-browser@2.2.0 playwright-core   # Node >= 18; pin the version
pip install antibrow==0.3.0                             # Python 3.9 - 3.13
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

- **The configuration invariant** - one identity, one profile, one persona, one egress, one timezone; what a shared cell breaks
- **Ten concrete checks** - timezone vs exit IP, WebRTC candidates, canvas stability across relaunches, worker vs main thread, one GPU across three interfaces, no duplicate personas or addresses across the fleet
- **Which suites to run** - CreepJS, whoer, browserleaks WebRTC, pixelscan, and `npx liarjs` for the ~40 rules you can run unattended in CI
- **Reading a failure** - the cheap causes (reused profile name, duplicate address, clock/address disagreement, a persona that regenerated) before suspecting the fingerprint
- **Runtime transparency** - where cookies, personas, proxy credentials and the API key actually go, and how to verify it yourself
- **What isolation cannot cover** - everything outside the browser, stated plainly

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

- **MCP server setup** - `npx -y anti-detect-browser@2.2.0 --mcp` (pinned) with `${VAR}` key expansion, or a Python stdio server via `antibrow[mcp]`
- **Treating page content as untrusted input** - the indirect-prompt-injection rules for an agent that both reads pages and picks the next tool call
- **Available tools** - `launch_browser`, `navigate`, `click`/`fill`, `screenshot`, `get_content`, profile and proxy tools, Live View controls
- **Agent-driven workflows** - example task flows with no user-written code, plus the operational gotchas (concurrency locks, headless, session hygiene)

## Repo structure

```
.claude-plugin/
  plugin.json       # plugin manifest (all three skills)
  marketplace.json  # lets this repo be added as a Claude Code marketplace
multi-account-isolation/
  SKILL.md          # isolation self-check: the ten assertions and the suites that verify them
anti-detect-browser/
  SKILL.md          # SDK (JS + Python) and REST API reference
browser-mcp-agent/
  SKILL.md          # MCP server mode reference
```

## Security and supply chain

- **Secrets come from the environment.** No sample in these skills contains a literal API key or a proxy password. In MCP configs the key is a `${ANTI_DETECT_BROWSER_KEY}` reference, not a value, because `.mcp.json` gets committed.
- **Pin the version.** Bare `npx anti-detect-browser` resolves `latest` at every start. Pin it, commit a lockfile, use `npm ci`, and verify a release before adopting it: `npm view anti-detect-browser@2.2.0 dist.integrity`. The npm package declares no install scripts; its dependencies are `ws`, `socks`, `yauzl`, `adm-zip`, `@modelcontextprotocol/sdk`.
- **Two artifacts land on the machine**: the MIT SDK from npm/PyPI, and a closed-source Chromium kernel downloaded once from AntiBrow's CDN into `~/.anti-detect-browser/`. Prefetch both at image-build time if the runtime must not fetch anything. Installed kernels are never swapped underneath a running profile.
- **License checks are online-only.** The kernel verifies a short-lived server-signed token at startup, roughly once a day in practice. There is no offline mode; air-gapped deployments are not supported.
- **Profile directories hold live cookies and session tokens.** Treat `~/.anti-detect-browser/` as credential material - keep it out of images, shared backups, and issue attachments. `browser.plan.redacted_args()` gives a secrets-masked command line for bug reports.
- **Page content is untrusted input.** Text, screenshots and `evaluate()` results are third-party data, never instruction - especially in MCP mode, where the agent also picks the next action. Each skill spells out the rules.

## Acceptable use

**Intended:** automating your own accounts and systems; running client accounts with the holder's authorization; collecting publicly available data; verifying your own ads, pricing and geo-gated content; testing your own anti-fraud and bot-detection stack; giving an agent a browser for work you would do yourself.

**Out of scope, and not supported:** accessing any system without authorization; credential stuffing or logging into accounts that are not yours; account takeover; bulk creation of fake accounts, reviews or engagement; circumventing authentication, payment or authorization controls; scraping personal data in violation of applicable law; working around a platform's enforcement decision.

Complying with the terms of the sites being automated, and with applicable law, is the operator's responsibility. Report abuse or a security issue via the contact at https://antibrow.com.

## Related

- npm package: [anti-detect-browser](https://www.npmjs.com/package/anti-detect-browser)
- PyPI package: [antibrow](https://pypi.org/project/antibrow/)
- SDK source (MIT): [github.com/antibrow/antibrow](https://github.com/antibrow/antibrow)
- Dashboard & docs: https://antibrow.com

The SDKs are MIT. The browser kernel is a closed-source binary downloaded at runtime under its own license - usable for your own work at any company size, but not redistributable; see `BINARY-LICENSE.md` in the SDK repo.

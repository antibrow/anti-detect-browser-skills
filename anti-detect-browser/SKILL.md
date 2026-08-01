---
name: anti-detect-browser
description: Launch and manage anti-detect browsers with kernel-level real-device fingerprints for multi-account operations, browser automation, web scraping, and AI agent workflows, from JavaScript/TypeScript (npm 'anti-detect-browser') or Python (PyPI 'antibrow'). Use when the user needs to run multiple isolated browser sessions with distinct fingerprints and persistent profiles (cookies, storage, login state), automate web scraping or data collection, verify ads and content across regions or device types, manage multiple social media or e-commerce accounts from one machine, run cross-browser or cross-device QA testing, or drive a stealth browser from browser-use, crawl4ai, Scrapling, Puppeteer, or Playwright. Also use when the user mentions 'antibrow', 'anti-detect browser', 'fingerprint browser', 'multi-account browser', 'web scraping browser', 'Playwright fingerprint', 'Python anti-detect browser', 'CreepJS', 'whoer', 'pixelscan', or 'residential proxy browser'. Runs on Windows x64, macOS (Intel + Apple Silicon), and Linux x64 / arm64. For an AI agent to drive the browser itself via MCP tool calls with no code to write, see browser-mcp-agent.
---

# Anti-Detect Browser SDK

Launch Chromium instances with real-device fingerprints via standard Playwright APIs. Every profile carries one coherent, real-device identity that is frozen at creation and replayed byte-for-byte on every later launch.

- npm package: `anti-detect-browser` (Node >= 18)
- PyPI package: `antibrow` (Python 3.9 - 3.13)
- Dashboard: `https://antibrow.com`
- REST API base: `https://antibrow.com/api/v1/`
- Documentation: `https://antibrow.com/docs`

## Why antibrow

- **Spoofing lives in the engine, not in a script.** A custom Chromium kernel answers Canvas, WebGL, WebGPU, audio, fonts, `navigator`, screen, DOMRect and timezone inside C++/Blink. There is no injected script to find, no property descriptor out of place, and worker contexts return exactly what the main thread does.
- **Real TLS and HTTP layer.** It *is* Chromium, so the ClientHello, cipher order and HTTP/2-3 behaviour are a genuine Chrome build's - the network half that a patched headless browser can never fake coherently.
- **One coherent persona per profile.** 30+ categories and 500+ parameters sampled from the same real machine. Independently randomized values contradict each other (an AMD renderer next to an Intel vendor string, a 1.0 DPR on a 1536x864 screen); these do not.
- **Timezone and geo follow the proxy.** The exit IP is resolved *through* the proxy before launch, then written into the fingerprint along with the WebRTC identity.
- **Proxy auth handled in the network stack.** HTTP/HTTPS 407 and SOCKS5 RFC 1929 are answered by the kernel, so nothing appears in `chrome://extensions` - a classic anti-detect tell avoided.
- **Unlimited local profiles, free.** A profile is a directory; name one and it exists. Plans cap *concurrent* browsers, not identities.
- **Drop-in Playwright API** in both JS and Python - existing scripts change only their launch line.
- **Runs as an MCP server** so AI agents drive it directly via tool calls.

## Platform support

| Platform | Status | Notes |
|---|---|---|
| Windows 10/11 x64 | Supported | Headful, or headless via off-screen window |
| macOS 12+ Apple Silicon + Intel | Supported | Universal build (arm64 + x64 in one bundle) |
| Linux x64 (glibc) | Supported | Headless needs Xvfb; container flags applied automatically |
| Linux arm64 (glibc) | Supported | Separate arm64 kernel, picked automatically from the CPU |
| Docker `linux/amd64` + `linux/arm64` | Supported | Run headful under Xvfb |
| Linux musl (Alpine) | Not yet | No kernel build |

The browser kernel is downloaded and cached once per version (~190 MB on Windows/Linux, ~320 MB for the macOS universal bundle). Real headless Chromium has its own detectable fingerprint, which is why headless mode moves the window off-screen on Windows and renders to a virtual display on Linux rather than using `--headless=new`.

## When to use

- **Multi-account management** - Run dozens of social media, e-commerce, or ad accounts on the same machine, each with its own isolated profile, fingerprint, cookies, and storage.
- **Web scraping & data collection** - Rotate fingerprints and proxies across scraping sessions so each session presents a consistent, independent device profile.
- **Ad verification & competitive intelligence** - View ads and content as different user profiles across regions and device types.
- **Social media automation** - Manage multiple accounts with persistent profiles that survive browser restarts.
- **E-commerce operations** - Operate multiple seller/buyer accounts with fully isolated browser environments.
- **QA & cross-environment testing** - Test how your site behaves under different browser fingerprints, screen sizes, and device configurations.

## Quick start

```bash
npm install anti-detect-browser playwright-core
```

```typescript
import { AntiDetectBrowser } from 'anti-detect-browser'

// Get your API key at https://antibrow.com - store it in an env var, never hardcode it
const ab = new AntiDetectBrowser({ key: process.env.ANTI_DETECT_BROWSER_KEY })

const { browser, page } = await ab.launch({
  fingerprint: { tags: ['Windows 10', 'Chrome'] },
  profile: 'my-account-01',
  proxy: process.env.PROXY_URL, // e.g. 'http://user:pass@host:port' - load from config, don't hardcode
})

// Standard Playwright API from here - zero learning curve
await page.goto('https://example.com')
await browser.close()
```

## What detection actually tests

Modern anti-bot systems do not compare one value against a blocklist. They **cross-check signals that must agree on a real device**, then score the contradictions. This is why JS-patching stealth plugins fail and an engine-level implementation does not - the list below is the standard consistency battery (see `npx liarjs` / `https://liarjs.dev` for an open implementation of ~40 such rules):

| Cross-check | What it exposes |
|---|---|
| `Function.prototype.toString`, own-instance props vs prototype getters | The *patch itself*. Any `navigator` override done from JS leaves a non-`[native code]` function or a rewritten descriptor. Kernel-level spoofing leaves neither. |
| Web Worker ↔ main thread | UA, `languages`, `hardwareConcurrency`, timezone, GPU and canvas re-read inside a worker. Partial overrides only patch the main thread. |
| Canvas read stability, and OffscreenCanvas ↔ 2D canvas | Per-call noise (a different hash every read) and half-hooked draw paths. Real hardware is deterministic. |
| WebGL ↔ WebGL2 ↔ WebGPU | Three interfaces must name one GPU. `adapter.info.vendor`/`architecture` has to match the unmasked WebGL renderer family. |
| UA string ↔ UA-CH `fullVersionList` ↔ `Sec-CH-UA` header | Version drift between the string, the client hints and the wire. |
| `navigator.platform` ↔ `Sec-CH-UA-Platform` ↔ font set | A "Windows" UA with no Segoe UI, or CJK fonts leaking on a non-CJK locale. |
| IP timezone ↔ `Intl` zone ↔ `Date.getTimezoneOffset()` ↔ DST rule | The single most common leak: proxy in Los Angeles, browser clock in Shanghai. |
| WebRTC ICE candidates ↔ connection IP, mDNS obfuscation | Real IP leaking past the proxy. |
| `DynamicsCompressor` defaults vs spec constants, H.264 codec support, plugin/mimeType shape vs the Chrome major | Values a script-level shim forgets to keep in sync with the version it claims. |
| DPR / `colorDepth` / `availHeight` realism, touch vs pointer media queries | Screen geometry that no shipped device has. |
| TLS ClientHello (length, extension order) + HTTP/2-3 behaviour vs the claimed Chrome build | The network half. Nothing running in JavaScript can reach it. |

antibrow answers each of these in the kernel from **one persona sampled from one real machine**, so the values are consistent by construction rather than by patch. Verify it yourself against [CreepJS](https://abrahamjuliot.github.io/creepjs/), [whoer.net](https://whoer.net), [browserleaks.com/canvas](https://browserleaks.com/canvas), [pixelscan.net](https://pixelscan.net), or `npx liarjs` in CI.

## Core concepts

### Profiles - persistent browser identities

A profile saves cookies, localStorage, and session data across launches. Same profile name = same stored state next time.

```typescript
// First launch - fresh session
const { page } = await ab.launch({ profile: 'shop-01' })
await page.goto('https://shop.example.com/login')
// ... login ...
await browser.close()

// Later - session restored, already logged in
const { page: p2 } = await ab.launch({ profile: 'shop-01' })
await p2.goto('https://shop.example.com/dashboard') // no login needed
```

### Fingerprints - real device data, frozen per profile

A new profile draws a real fingerprint collected from an actual device - 30+ categories (Canvas, WebGL, WebGPU, Audio, Fonts, WebRTC, etc.) with 500+ individual parameters - and then **freezes it**. The persona is written once to `persona.json` and never regenerated, so the same profile reports the same UA, GPU, screen, seeds and font set on every launch. Determinism matters as much as the values: a browser that returns a *new* canvas hash on every call is trivially flagged.

```typescript
// Windows Chrome, version 130+
await ab.launch({
  fingerprint: { tags: ['Windows 10', 'Chrome'], minBrowserVersion: 130 },
})

// Mac Safari
await ab.launch({
  fingerprint: { tags: ['Apple Mac', 'Safari'] },
})

// Mobile Android
await ab.launch({
  fingerprint: { tags: ['Android', 'Mobile', 'Chrome'] },
})
```

Available filter tags: `Microsoft Windows`, `Apple Mac`, `Android`, `Linux`, `iPad`, `iPhone`, `Edge`, `Chrome`, `Safari`, `Firefox`, `Desktop`, `Mobile`, `Windows 7`, `Windows 8`, `Windows 10`

### Visual identification - tell windows apart at a glance

When running many browsers simultaneously, each window gets a floating label, title prefix, and unique theme color.

```typescript
await ab.launch({
  profile: 'twitter-main',
  label: '@myhandle',       // floating label + window title
  color: '#e74c3c',         // unique window border color
})
```

### Proxy integration

Route each browser through a different proxy for geo-targeting or IP rotation. Proxy URL format: `socks5://user:pass@host:port` - load the actual value from an env var or secrets store, don't hardcode it.

```typescript
await ab.launch({
  proxy: process.env.US_PROXY_URL,
  fingerprint: { tags: ['Windows 10', 'Chrome'] },
  profile: 'us-account',
})
```

### Live View - watch headless browsers in real time

Monitor headless sessions from the `https://antibrow.com` dashboard. Useful for debugging AI agent actions or letting team members observe.

```typescript
const { liveView } = await ab.launch({
  headless: true,
  liveView: true,
})

console.log('Watch live:', liveView.viewUrl)
// Share this URL - anyone with access can see the browser screen
```

## Inject into existing Playwright setup

Already have Playwright scripts? Add fingerprints without changing your workflow.

```typescript
import { chromium } from 'playwright'
import { applyFingerprint } from 'anti-detect-browser'

const browser = await chromium.launch()
const context = await browser.newContext()

await applyFingerprint(context, {
  key: process.env.ANTI_DETECT_BROWSER_KEY,
  fingerprint: { tags: ['Windows 10', 'Chrome'] },
  profile: 'my-profile',
})

const page = await context.newPage()
await page.goto('https://example.com')
```

## Python SDK - `antibrow` on PyPI

Same product, same kernel, same on-disk profile format. A profile created from Node is launchable from Python with the identical fingerprint, because both SDKs share `~/.anti-detect-browser/`.

```bash
pip install antibrow
python -m antibrow install    # download the kernel (one-time; first launch does it too)
python -m antibrow login      # store the API key in ~/.antibrow/license.key
```

`playwright install` is **not** needed - antibrow drives its own kernel. The `playwright` pip package is still required for its client library.

```python
from antibrow import launch

# Named profile: same fingerprint, cookies and storage every time.
browser = launch(profile="shopper-01")

page = browser.new_page()
page.goto("https://whoer.net")
print(page.title())

browser.close()
```

Context manager, headless, proxy with geo-matched timezone:

```python
with launch(
    profile="scraper-eu",
    headless=True,
    proxy="socks5://user:pass@gate.example.com:1080",
    geoip=True,                  # timezone + WebRTC follow the proxy exit
    label="acct@shop.com",       # address-bar tag, tells windows apart
) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
    print(browser.timezone, browser.public_ip)   # America/Los_Angeles 203.0.113.7
```

Async twin, for agents and concurrent crawls:

```python
import asyncio
from antibrow import launch_async

async def main():
    browser = await launch_async(profile="agent-01")
    page = await browser.new_page()
    await page.goto("https://example.com")
    await browser.close()

asyncio.run(main())
```

### Key `launch()` options

| Option | Default | What it does |
|---|---|---|
| `profile` | `"default"` | Same name → same identity, cookies, storage. Unlimited and free. |
| `headless` | `False` | Off-screen window on Windows; use Xvfb on Linux; no effect on macOS yet. |
| `proxy` | `None` | `http://` / `https://` / `socks5://` / `relay://` URL, or Playwright's dict form. |
| `geoip` | `True` | Resolve the exit IP *through* the proxy and match timezone + WebRTC to it. |
| `timezone` | `None` | Force an IANA zone, overriding the geo lookup. |
| `profile_dir` | `None` | Exact directory, bypassing `cache_dir`/`profile` - handy for CI volumes. |
| `kernel_version` | newest | Kernel for a **new** profile; existing profiles keep the version frozen in their persona. |
| `proxy_auth` | `"native"` | Credentials answered in the network stack, with no extension loaded. |
| `update_kernel` | `False` | Check for a newer kernel build and install it before launching. |
| `on_progress` | `None` | Receives progress lines during download and startup. |

### The handle

Attribute lookups fall through to the Playwright `BrowserContext`, so it behaves like one:

```python
browser.new_page(); browser.pages; browser.add_cookies([...])   # delegated to the context
browser.context, browser.browser        # raw Playwright objects
browser.cdp_url, browser.cdp_endpoint   # hand these to any CDP-speaking framework
browser.persona                         # frozen identity: UA, GPU, screen, seeds
browser.timezone, browser.public_ip, browser.kernel_version, browser.pid
browser.plan.redacted_args()            # command line with secrets masked, safe for bug reports
```

Other entry points: `launch_async()` (asyncio), `launch_persistent_context()` (a literal Playwright `BrowserContext`), `prepare_launch()` (resolve executable, args, persona and timezone without starting a process).

Errors all derive from `AntibrowError` - catch `ConcurrencyLimitError` (plan's simultaneous-browser cap, enforced by the kernel via cross-process locks) and `LicenseError` (missing or rejected key) specifically.

### Framework integrations

Every integration is the same move: antibrow starts the browser, you hand its **CDP endpoint** to whatever drives it.

```python
# browser-use
session = await launch_async(profile="agent-01", proxy="http://user:pass@gate:8080")
agent = Agent(task="...", llm=ChatOpenAI(model="gpt-4.1-mini"),
              browser=Browser(cdp_url=session.cdp_url))

# crawl4ai
config = BrowserConfig(cdp_url=session.cdp_url, headless=False)

# Scrapling
page = DynamicFetcher.fetch("https://example.com", cdp_url=browser.cdp_endpoint)

# Puppeteer (any language) - it is plain CDP
# puppeteer.connect({ browserURL: browser.cdp_url })
```

Selenium is not supported: it cannot attach to a CDP-only endpoint without a matching chromedriver.

### CLI and environment

```bash
python -m antibrow install [--version 150.0.7871.182] [--force]
python -m antibrow info      # kernels, profiles, license, cache dir - run this first when debugging
python -m antibrow login [--key ab_live_...]
python -m antibrow version
```

`ANTIBROW_API_KEY` (also accepts the Node SDK's `ANTI_DETECT_BROWSER_KEY`), `ANTIBROW_LICENSE_TOKEN`, `ANTIBROW_CACHE_DIR`, `ANTIBROW_SERVER`.

### Docker

The Linux kernel runs **headful under Xvfb** - real headless Chromium has its own fingerprint, so the image renders to a virtual display. The same image works on `linux/amd64` and `linux/arm64`; the matching kernel build is chosen from the container's CPU.

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
      xvfb libnss3 libatk1.0-0 libatk-bridge2.0-0 libcups2 libdrm2 \
      libxkbcommon0 libxcomposite1 libxdamage1 libxfixes3 libxrandr2 \
      libgbm1 libasound2 libpango-1.0-0 libcairo2 fonts-liberation ca-certificates \
    && rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir antibrow
COPY script.py .
CMD ["xvfb-run", "-a", "python", "script.py"]
```

```bash
docker run --rm -e ANTIBROW_API_KEY=$ANTIBROW_API_KEY \
  -v antibrow-cache:/root/.anti-detect-browser my-scraper
```

Mount the cache volume so the kernel and profiles survive between runs.

## Keeping the browser kernel up to date

Installed kernels are cached and **never swapped under you**.

```typescript
if (await ab.hasKernelUpdate()) {
  const updated = await ab.updateKernel()      // → ['150.0.7871.182']
}
await ab.launch({ profile: 'shopper-01', updateKernelBeforeLaunch: true })  // default false
```

Python: `python -m antibrow install --force`, or `launch(update_kernel=True)`.

`launch()` checks once per process in the background and prints a one-line notice if a newer build exists. Offline machines skip the check silently - updates never block a launch.

## Plans and concurrency

Local profiles are unlimited on every plan, including free. What scales is how many browsers run **at the same time**, enforced by the kernel with cross-process file locks - spawning more Node or Python processes does not get around it.

| Plan | Local profiles | Concurrent browsers | Cloud sync | Managed proxies |
|---|:--:|:--:|:--:|:--:|
| Free | unlimited | 1 | – | – |
| Basic | unlimited | 5 | yes | yes |
| Pro | unlimited | 20 | yes | yes |
| Team | unlimited | 100 | yes | yes |

Exceeding the cap raises an error rather than hanging. Cloud profile sync and Live View are implemented in the Node SDK and the desktop app; the Python package is local-only for now.

## Licensing

The SDKs (npm + PyPI) are **MIT**. The browser kernel is a **closed-source binary** downloaded from AntiBrow's CDN onto the end user's machine at runtime - usable for your own work including commercial work at any company size, but not redistributable, resellable or embeddable; exposing it to third-party customers needs a separate OEM/SaaS license. Listing these packages as a dependency is **not** redistribution. `BINARY-LICENSE.md` in `https://github.com/antibrow/antibrow` is the authoritative text.

An API key is required: the kernel verifies a short-lived, server-signed license token at startup, and that check is compiled into the binary - there is no offline mode. The token is cached, so a tight relaunch loop hits the network roughly once a day.

## MCP server mode - for AI agents

`anti-detect-browser` can also run as an MCP server so an agent drives the browser directly via tool calls, without writing any of the SDK code below. Setup, the full tool list, and example agent-driven flows live in the **`browser-mcp-agent`** skill.

## Workflow examples

### Multi-account social media

```typescript
const accounts = [
  { profile: 'twitter-1', label: '@brand_main', color: '#1DA1F2' },
  { profile: 'twitter-2', label: '@support', color: '#FF6B35' },
  { profile: 'twitter-3', label: '@personal', color: '#6C5CE7' },
]

for (const acct of accounts) {
  const { page } = await ab.launch({
    fingerprint: { tags: ['Windows 10', 'Chrome'] },
    proxy: getNextProxy(),
    ...acct,
  })
  await page.goto('https://twitter.com')
}
```

### Scraping with fingerprint rotation

```typescript
for (const url of urlsToScrape) {
  const { browser, page } = await ab.launch({
    fingerprint: { tags: ['Desktop', 'Chrome'], minBrowserVersion: 125 },
    proxy: rotateProxy(),
  })
  await page.goto(url)
  const data = await page.evaluate(() => document.body.innerText)
  saveData(url, data)
  await browser.close()
}
```

### Headless monitoring with live view

```typescript
const { page, liveView } = await ab.launch({
  headless: true,
  liveView: true,
  profile: 'price-monitor',
  fingerprint: { tags: ['Windows 10', 'Chrome'] },
})

// Share the live view URL with your team
console.log('Dashboard:', liveView.viewUrl)

while (true) {
  await page.goto('https://shop.example.com/product/123')
  const price = await page.textContent('.price')
  if (parseFloat(price) < targetPrice) notify(price)
  await page.waitForTimeout(60_000)
}
```

## REST API

Base URL: `https://antibrow.com/api/v1/` - all endpoints require `Authorization: Bearer <api-key>` header.

### Fingerprints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/fingerprints/fetch` | Fetch a fingerprint matching filter criteria. Returns `{ dataUrl }` - download the presigned URL for full fingerprint data. |
| `GET` | `/fingerprints/versions` | List available browser versions |

Query parameters for `/fingerprints/fetch`: `tags`, `id`, `minBrowserVersion`, `maxBrowserVersion`, `minWidth`, `maxWidth`, `minHeight`, `maxHeight`

### Profiles

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/profiles` | List all profiles |
| `POST` | `/profiles` | Create a new profile (server assigns a random fingerprint). Returns profile info including `dataUrl` for immediate fingerprint data download. |
| `GET` | `/profiles/:name` | Get profile details with `dataUrl` for fingerprint data download |
| `DELETE` | `/profiles/:name` | Delete a profile |

**POST `/profiles` request body:**
```json
{ "name": "my-profile", "tags": ["Windows 10", "Chrome"] }
```

**POST `/profiles` response (201):**
```json
{
  "name": "my-profile",
  "tags": ["Windows 10", "Chrome"],
  "ua": "Mozilla/5.0 ...",
  "browserVersion": 131,
  "width": 1920,
  "height": 1080,
  "createdAt": "2025-01-01T00:00:00.000Z",
  "dataUrl": "https://cdn.example.com/fingerprints/..."
}
```

The `dataUrl` is a presigned download URL valid for a limited time - download it directly and promptly, no additional API call needed.

## Get started

1. Sign up at `https://antibrow.com` - the free key gives 1 concurrent browser and unlimited local profiles
2. Get your API key from the dashboard
3. `npm install anti-detect-browser playwright-core`, or `pip install antibrow`
4. Launch your first anti-detect browser - the kernel downloads on first run

Full documentation: `https://antibrow.com/docs` · SDK reference: `https://antibrow.com/docs/sdk` · Source: `https://github.com/antibrow/antibrow`

## Acceptable use

Automating systems without authorization, credential stuffing and bulk account-creation abuse are out of scope. Scraping public data, testing your own anti-fraud stack, and managing your own accounts are the intended uses. Complying with the terms of the sites being automated is the operator's responsibility.

## Related Skills

- **browser-mcp-agent** - run as an MCP server so an AI agent drives the browser itself via tool calls, no SDK code required

---
name: anti-detect-browser
description: Launch and manage anti-detect browsers with real, per-profile device fingerprints for multi-account operations, browser automation, web scraping, and AI agent workflows. Use when the user needs to run multiple isolated browser sessions with distinct fingerprints and persistent profiles (cookies, storage, login state), automate web scraping or data collection, verify ads and content across regions or device types, manage multiple social media or e-commerce accounts from one machine, or run cross-browser or cross-device QA testing. Also use when the user mentions 'antibrow', 'anti-detect browser', 'fingerprint browser', 'multi-account browser', 'web scraping browser', 'Playwright fingerprint', or 'residential proxy browser'. For an AI agent to drive the browser itself via MCP tool calls with no code to write, see browser-mcp-agent.
---

# Anti-Detect Browser SDK

Launch Chromium instances with real-device fingerprints via standard Playwright APIs. Each browser launch gets a real, consistent device fingerprint that stays stable across sessions and is distinct per profile.

- npm package: `anti-detect-browser`
- Dashboard: `https://antibrow.com`
- REST API base: `https://antibrow.com/api/v1/`
- Documentation: `https://antibrow.com/docs`

## Why antibrow

- Fingerprints are captured from real devices, not synthetically generated - 30+ categories, 500+ parameters
- Local profiles persist indefinitely at no extra cost, unlike cloud browser services billed per concurrent session
- Drop-in Playwright API - existing scripts work with `applyFingerprint()`, no new SDK to learn
- Runs as an MCP server so AI agents drive it directly via tool calls

## When to use

- **Multi-account management** - Run dozens of social media, e-commerce, or ad accounts on the same machine, each with its own isolated profile, fingerprint, cookies, and storage.
- **Web scraping & data collection** - Rotate fingerprints and proxies across scraping sessions so each session presents a consistent, independent device profile.
- **Ad verification & competitive intelligence** - View ads and content as different user profiles across regions and device types.
- **Social media automation** - Manage multiple accounts with persistent profiles that survive browser restarts.
- **E-commerce operations** - Operate multiple seller/buyer accounts with fully isolated browser environments.
- **QA & cross-environment testing** - Test how your site behaves under different browser fingerprints, screen sizes, and device configurations.

## Quick start

```bash
npm install anti-detect-browser
```

```typescript
import { AntiDetectBrowser } from 'anti-detect-browser'

// Get your API key at https://antibrow.com — store it in an env var, never hardcode it
const ab = new AntiDetectBrowser({ key: process.env.ANTIBROW_API_KEY })

const { browser, page } = await ab.launch({
  fingerprint: { tags: ['Windows 10', 'Chrome'] },
  profile: 'my-account-01',
  proxy: process.env.PROXY_URL, // e.g. 'http://user:pass@host:port' — load from config, don't hardcode
})

// Standard Playwright API from here — zero learning curve
await page.goto('https://example.com')
await browser.close()
```

## Core concepts

### Profiles — persistent browser identities

A profile saves cookies, localStorage, and session data across launches. Same profile name = same stored state next time.

```typescript
// First launch — fresh session
const { page } = await ab.launch({ profile: 'shop-01' })
await page.goto('https://shop.example.com/login')
// ... login ...
await browser.close()

// Later — session restored, already logged in
const { page: p2 } = await ab.launch({ profile: 'shop-01' })
await p2.goto('https://shop.example.com/dashboard') // no login needed
```

### Fingerprints — real device data from the cloud

Each launch fetches a real fingerprint collected from actual devices. Over 30 categories (Canvas, WebGL, Audio, Fonts, WebRTC, WebGPU, etc.) with 500+ individual parameters.

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

### Visual identification — tell windows apart at a glance

When running many browsers simultaneously, each window gets a floating label, title prefix, and unique theme color.

```typescript
await ab.launch({
  profile: 'twitter-main',
  label: '@myhandle',       // floating label + window title
  color: '#e74c3c',         // unique window border color
})
```

### Proxy integration

Route each browser through a different proxy for geo-targeting or IP rotation. Proxy URL format: `socks5://user:pass@host:port` — load the actual value from an env var or secrets store, don't hardcode it.

```typescript
await ab.launch({
  proxy: process.env.US_PROXY_URL,
  fingerprint: { tags: ['Windows 10', 'Chrome'] },
  profile: 'us-account',
})
```

### Live View — watch headless browsers in real time

Monitor headless sessions from the `https://antibrow.com` dashboard. Useful for debugging AI agent actions or letting team members observe.

```typescript
const { liveView } = await ab.launch({
  headless: true,
  liveView: true,
})

console.log('Watch live:', liveView.viewUrl)
// Share this URL — anyone with access can see the browser screen
```

## Inject into existing Playwright setup

Already have Playwright scripts? Add fingerprints without changing your workflow.

```typescript
import { chromium } from 'playwright'
import { applyFingerprint } from 'anti-detect-browser'

const browser = await chromium.launch()
const context = await browser.newContext()

await applyFingerprint(context, {
  key: process.env.ANTIBROW_API_KEY,
  fingerprint: { tags: ['Windows 10', 'Chrome'] },
  profile: 'my-profile',
})

const page = await context.newPage()
await page.goto('https://example.com')
```

## MCP server mode — for AI agents

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

Base URL: `https://antibrow.com/api/v1/` — all endpoints require `Authorization: Bearer <api-key>` header.

### Fingerprints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/fingerprints/fetch` | Fetch a fingerprint matching filter criteria. Returns `{ dataUrl }` — download the presigned URL for full fingerprint data. |
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

The `dataUrl` is a presigned download URL valid for a limited time — download it directly and promptly, no additional API call needed.

## Get started

1. Sign up at `https://antibrow.com` (free tier: 2 browser profiles)
2. Get your API key from the dashboard
3. `npm install anti-detect-browser`
4. Launch your first anti-detect browser

Full documentation: `https://antibrow.com/docs`

## Related Skills

- **browser-mcp-agent** - run as an MCP server so an AI agent drives the browser itself via tool calls, no SDK code required

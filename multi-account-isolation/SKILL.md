---
name: multi-account-isolation
description: Avoid account linking and bans when running multiple accounts from one machine - the full isolation checklist, not just browser fingerprints. Use when the user asks how to keep accounts from being associated with each other, why accounts got banned or restricted after being used on the same computer, how to run many social media / marketplace / ad accounts safely, how to pair a proxy with each account, or what links accounts beyond the browser (IP, cookies, timezone, payment details, recovery email and phone, behaviour). Also use when the user mentions 'account association', 'avoid account linking', 'accounts got linked', 'multi-account setup', 'account ban prevention', 'shadowban', '账号关联', '防关联', '防封号', '多账号运营', or 'multiple accounts same device'. Covers per-account profile + proxy + timezone pairing, account warm-up, verification, and the leaks that survive a perfect fingerprint. Implementation is the anti-detect-browser skill.
---

# Multi-Account Isolation

Keeping several of your own accounts from being tied together by a platform is not one problem, it is a stack of them. A perfect browser fingerprint fixes exactly one layer. Accounts far more often get linked by an IP, a shared cookie, a mismatched clock, or the same recovery phone number than by a canvas hash.

This skill is the checklist. For the code that implements the browser layer, see the **anti-detect-browser** skill.

## The model: platforms link on many layers, and only need one

Linkage systems build a graph. Every account is a node; every shared signal is an edge. One strong edge (same payment card) or several weak ones (same /24 subnet + same screen resolution + same active hours) is enough to merge two nodes into "one operator".

So the goal is not "beat the fingerprint test". It is **leave no edge**. Getting nine layers right and the tenth wrong still merges the nodes.

## Linkage surface

| Layer | What links the accounts | How to isolate |
|---|---|---|
| **Browser fingerprint** | Canvas, WebGL/WebGPU, audio, fonts, screen, UA, `navigator` all identical across accounts | One antibrow profile per account - each draws its own real-device persona, frozen at creation |
| **Cookies / storage** | A third-party or leftover first-party cookie seen under two logins; `localStorage` IDs; service-worker caches | Separate profile per account. Never "log out and log in as the other account" in one profile |
| **IP address** | Same exit IP, or same subnet, or an IP already burned by a banned account | One proxy per account, sticky (not rotating) for logged-in sessions. Residential or mobile, not datacenter |
| **Timezone / locale / geo** | Browser clock in one country, exit IP in another - the single most common tell | `geoip=True` (default): the exit IP is resolved *through* the proxy and timezone + WebRTC are written to match |
| **WebRTC** | Real IP leaking past the proxy in ICE candidates | Handled in the kernel when the proxy is set; verify at browserleaks or whoer |
| **Account metadata** | Same recovery email, same phone, same birthday, same name spelling, same profile photo file | Distinct per account. This is the edge people most often forget - no browser setting touches it |
| **Payment** | Same card, same PayPal, same payout bank account, same billing address | Distinct per account where the platform allows it; this is usually the hardest-to-hide edge |
| **Behaviour** | All accounts active in the same 20-minute window, identical posting cadence, same follow targets, copy-pasted content | Stagger schedules and vary content. Nothing technical fixes this |
| **Referral graph** | Accounts following, liking, or messaging each other early on | Avoid cross-interaction, especially before accounts are aged |

The bottom four rows are **not** solved by an anti-detect browser. If accounts keep getting linked with the browser layer correct, look there first.

## The rule: one account, one of everything

```
account  →  profile  →  fingerprint  →  proxy  →  timezone  →  identity data
   1     :     1      :       1        :    1    :     1      :        1
```

Any shared cell in that row is a potential edge. Profiles are unlimited and free on every antibrow plan, so there is never a reason to reuse one.

## Setup

Pair each account with its own profile and its own sticky proxy. The persona is drawn once and frozen, so the account sees the same device on every later launch - which is what a real user looks like.

```typescript
import { AntiDetectBrowser } from 'anti-detect-browser'

const ab = new AntiDetectBrowser({ key: process.env.ANTI_DETECT_BROWSER_KEY })

const accounts = [
  { profile: 'shop-us-01', proxy: process.env.PROXY_US_1, tags: ['Windows 10', 'Chrome'] },
  { profile: 'shop-us-02', proxy: process.env.PROXY_US_2, tags: ['Apple Mac', 'Safari'] },
  { profile: 'shop-de-01', proxy: process.env.PROXY_DE_1, tags: ['Windows 10', 'Edge'] },
]

for (const a of accounts) {
  const { browser, page } = await ab.launch({
    profile: a.profile,               // isolated cookies, storage, login state
    proxy: a.proxy,                   // sticky, one per account
    fingerprint: { tags: a.tags },    // frozen at first launch, replayed after
    label: a.profile,                 // floating label so windows are tellable apart
  })
  // ... work this account, then close it ...
  await browser.close()
}
```

Python, same profile format and same on-disk identity:

```python
from antibrow import launch

with launch(
    profile="shop-us-01",
    proxy="socks5://user:pass@us-sticky.example.com:1080",
    geoip=True,            # timezone + WebRTC follow the proxy exit
    label="shop-us-01",
) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
    print(browser.timezone, browser.public_ip)
```

### Proxy selection

- **Sticky, not rotating.** A logged-in session whose IP changes mid-flight looks like a hijacked session. Rotation belongs in scraping, not in account operation.
- **Residential or mobile** for consumer platforms. Datacenter ranges are widely tagged.
- **Match the account's claimed location** to the proxy, and keep it stable. An account that has always been in Ohio and suddenly appears in Vietnam is a stronger signal than any fingerprint.
- **Never share one exit across accounts**, and do not reuse the exit of an account that was already banned.

### Fingerprint variety

Do not give every account `['Windows 10', 'Chrome']`. Real populations are mixed. Vary the tags across the fleet - `Apple Mac`, `Android`, `Edge`, `Mobile` - and let each account keep its own draw forever.

## Warm-up

New accounts that immediately do the thing you made them for are the easiest to catch. Age each account with ordinary behaviour before it does anything of value: browse, read, follow a few unrelated things, come back the next day. Days, not minutes. Warm-up is a behaviour problem, so no browser feature substitutes for it.

## Verify before you trust the setup

Run a fresh profile against a detection suite **through its own proxy** and confirm the layers agree:

- [CreepJS](https://abrahamjuliot.github.io/creepjs/) - cross-layer consistency, worker vs main thread
- [whoer.net](https://whoer.net) - IP, timezone and locale agreement at a glance
- [browserleaks.com/webrtc](https://browserleaks.com/webrtc) - real IP leaking past the proxy
- [pixelscan.net](https://pixelscan.net) - IP-to-fingerprint coherence
- `npx liarjs` ([liarjs.dev](https://liarjs.dev)) - ~40 open-source consistency rules, runnable in CI

Check specifically: does the reported timezone match the exit IP's country, does the WebRTC candidate show only the proxy, and does the canvas hash stay **the same** across two launches of the same profile (a changing hash is itself a flag).

## Troubleshooting: accounts still getting linked

Work down in this order - the cheap layers first, because they are the ones usually at fault:

1. **Two accounts in one profile?** Check that no profile name was reused. `list_profiles` or `~/.anti-detect-browser/`.
2. **Shared or recycled IP?** Confirm each account's `public_ip` is distinct and that none belonged to a banned account.
3. **Timezone mismatch?** Print `browser.timezone` and `browser.public_ip` and confirm they agree.
4. **Shared account metadata?** Recovery email, phone, payout account, billing address - the most common real cause.
5. **Behavioural overlap?** Same active hours, same content, accounts interacting with each other.
6. **Only then** suspect the fingerprint - and verify it with the tools above rather than assuming.

## What this cannot do

- It does not make a banned account come back.
- It does not hide a shared payment instrument or a shared payout account from a platform that checks them.
- It does not fix content or behaviour that a platform would penalise anyway.
- It does not defeat identity verification - a document check is not a fingerprint problem.

## Acceptable use

Managing your own accounts across platforms, running client accounts with authorization, testing your own anti-fraud stack, and scraping public data are the intended uses. Bulk account-creation abuse, credential stuffing, evading a ban you were given for a policy violation, and automating systems you have no authorization to access are out of scope. Complying with the terms of the platforms being used is the operator's responsibility.

## Related Skills

- **anti-detect-browser** - the SDK, profiles, fingerprints, proxies, and REST API that implement the browser layer
- **browser-mcp-agent** - MCP server mode, for letting an AI agent drive an isolated profile itself

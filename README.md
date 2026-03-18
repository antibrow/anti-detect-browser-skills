# Anti-Detect Browser — Claude Code Skill

A [Claude Code custom skill](https://docs.anthropic.com/en/docs/claude-code/skills) that teaches Claude how to launch and manage anti-detect browsers with real-device fingerprints.

Once installed, Claude can help you write multi-account automation scripts, configure fingerprint browsers, call the REST API, set up MCP server mode, and more — all using the `anti-detect-browser` npm package.

## Install

```bash
claude skill install --from https://github.com/antibrow/anti-detect-browser-skills
```
or
```bash
npx skills add https://github.com/antibrow/anti-detect-browser-skills --skill anti-detect-browser
```

Or add the repo URL in Claude Code settings under **Skills**.

## What Claude learns

After installation, Claude gains context about:

- **SDK usage** — `AntiDetectBrowser` class, `launch()` options, `applyFingerprint()` for existing Playwright setups
- **Profile management** — persistent browser identities with cookies, storage, and fingerprint binding
- **Fingerprint filtering** — tags, browser version ranges, screen size constraints
- **Proxy integration** — per-browser proxy routing
- **Visual identification** — floating labels, window titles, theme colors for multi-window workflows
- **Live View** — real-time headless browser streaming to the dashboard
- **MCP server mode** — running as a tool server for AI agents (Claude, GPT, etc.)
- **REST API** — all public `/api/v1/` endpoints for fingerprints and profiles

## Repo structure

```
anti-detect-browser/
  SKILL.md    # Skill definition with trigger rules and full SDK reference
```

## Related

- npm package: [anti-detect-browser](https://www.npmjs.com/package/anti-detect-browser)
- Dashboard & docs: https://antibrow.com

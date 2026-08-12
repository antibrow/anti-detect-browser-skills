# Android profiles

Loaded on demand from the `anti-detect-browser` skill. A profile created with `deviceType: 'android'` (`device_type="android"` in Python) presents a real phone's identity while running on the Windows, macOS or Linux machine you already have.

This is a **desktop-hosted simulation**, not a device farm and not a remote phone. Nothing about the host changes; what changes is the identity the page sees, answered in the kernel rather than by a script.

## What an Android profile reports

| Surface | Value |
|---|---|
| UA + client hints | `Mobile Safari` UA, `Sec-CH-UA-Mobile: ?1`, `Sec-CH-UA-Platform: "Android"`, real `model` and `platformVersion`, `formFactors: ["Mobile"]`, empty `architecture` / `bitness` |
| `navigator` | `platform: "Linux armv81"`, mobile `maxTouchPoints`, mobile core and memory counts, no plugins or mime types, `pdfViewerEnabled: false` |
| Touch and layout | `ontouchstart`, `window.orientation`, portrait `screen.orientation`, `(pointer: coarse)` / `(hover: none)`, window sized to the device screen so `innerWidth === screen.width` on a viewport-meta page |
| GPU | mobile unmasked vendor and renderer, plus the ETC/ASTC compressed-texture extensions a phone GPU actually exposes |
| Screen, audio, fonts, connection | taken from the same device as everything above |

Three real phones ship **inside the package**, so a free-plan profile can be Android with no network round-trip. The package picks a whole device row, never an individual field - that is the only way the screen, the GPU report and the client hints stay consistent with each other.

## The two constraints

- **The device type is fixed when the profile is created.** It lives in the persona, so passing `deviceType` to a profile that already exists does nothing. Create a new profile to switch device classes; this is the same freeze that keeps a desktop profile's canvas hash stable.
- **Android needs kernel `151` or newer** - the first build carrying mobile support. Capability is decided by the Chrome major alone; the build stamp is not consulted, so a qualifying kernel is never refused because a manifest row omitted its build or carried an older date. The SDK picks the newest qualifying kernel for a new Android profile, installs it, and fails with an explicit message rather than launching a desktop kernel behind a phone's fingerprint.

```typescript
import { androidCapableKernels, kernelSupportsAndroid, resolveAndroidKernel } from 'anti-detect-browser'

kernelSupportsAndroid('151')     // → true
await androidCapableKernels()    // → the kernel versions that qualify
await resolveAndroidKernel()     // → the one a new Android profile would get
```

Python: `kernel_supports_android()`, `android_capable_kernels()`, `resolve_android_kernel()`.

An explicit `kernelVersion` is honoured when it qualifies and ignored in favour of the newest qualifying build when it does not. Existing profiles keep the kernel frozen into their identity either way. Because the floor is a Chrome major rather than a pinned build, kernels published later become available through the manifest with no SDK release needed.

## Launching one

```typescript
const { page } = await ab.launch({
  profile: 'phone-01',
  deviceType: 'android',        // 'desktop' (default) | 'android'
})
```

```python
browser = launch(profile="phone-01", device_type="android")
```

`headless: true` on an Android profile keeps the persona's screen size while the window is hidden, so `innerWidth` does not start contradicting the spoofed `screen.width`.

## Drawing from the fingerprint library instead

```typescript
await ab.launch({ profile: 'phone-02', deviceType: 'android', realFingerprint: true })
```

`realFingerprint` takes the identity from the captured-device library on the server instead of the bundled table, so each profile is a different real machine - Android or desktop. It is a paid-plan option: a free key is rejected by the server rather than quietly downgraded to a generated persona. Like `deviceType`, it applies only when the profile is first created.

## Over MCP

`launch_browser` and `create_profile` both accept `deviceType` and `realFingerprint`, so an agent can ask for a phone profile in the same call that starts it. See the `browser-mcp-agent` skill.

## What it does not do

- It does not run Android, ARM binaries, or an app. The page sees a phone; the process is desktop Chromium with a mobile persona.
- It does not change how you drive the page. The persona reports touch support; the input still comes from your script over the same Playwright/CDP connection as a desktop profile, so a flow that needs real gestures needs you to send them.
- It does not make a mobile-only app or a device-attestation check pass. Attestation is signed by hardware this profile does not have.

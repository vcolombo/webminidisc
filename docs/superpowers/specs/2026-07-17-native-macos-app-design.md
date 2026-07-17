# Native macOS App — Design Spec

**Date:** 2026-07-17
**Status:** Approved architecture; revised after external (Codex) review

## Goal

Ship Web MiniDisc Pro as a native macOS application: a small, notarized `.app`
distributed via DMG (not the Mac App Store). Reuse the existing React/Vite app
essentially unchanged.

## Scope (revised)

**v1 = NetMD only.** NetMD is a USB vendor-class device — libusb/`nusb` claims
it cleanly on macOS. This is the shippable target.

**Full HiMD is out of v1 — gated investigation.** Full HiMD enumerates as USB
Mass Storage; on macOS the kernel driver owns that interface. The chosen `nusb`
backend cannot detach a kernel driver on macOS (`detach_kernel_driver` /
`detach_and_claim_interface` are Linux-only, no-ops elsewhere), and unmounting
the volume is not documented to release the interface claim. There is **no
demonstrated macOS strategy** for taking exclusive access. Full HiMD support is
therefore a separate spike (see "HiMD gate" below), not a milestone of v1. HiMD
read-only/filesystem features that do not require raw SCSI control may still be
in reach — to be confirmed by the spike, not assumed.

## Constraints & decisions

- **Not the Mac App Store.** The repo is GPL-2.0 (authored by `asivery`; this is
  a fork). GPLv2 §6 is incompatible with the App Store's usage/DRM terms (the
  known VLC/FSF case), and we are not the copyright holder for the app or its
  GPL dependencies. Dropping the App Store removes this legal blocker.
  Distribution is via notarized DMG only.
- **Native, not Electron.** Small native shell via **Tauri v2** (system WKWebView
  + Rust). No bundled Chromium. Requires Tauri **≥ 2.1.0** for the
  `app.security.headers` support relied on below.
- **WebUSB is the linchpin.** WKWebView has no WebUSB. NetMD access currently
  flows through `navigator.usb`. We replace `navigator.usb` with a native USB
  bridge rather than editing the device libraries.

## Guiding principle: polyfill, don't fork

Install a drop-in `navigator.usb` replacement at startup, **only** when running
under Tauri. The web build keeps using real WebUSB; the Tauri build uses the
shim. Existing device-library call sites (`src/services/interfaces/netmd.ts`,
`netmd-js`) stay untouched.

**Tauri detection is not automatic.** `window.__TAURI__` only exists if
`app.withGlobalTauri: true` is set; otherwise the shim must import the Tauri
JS API (`@tauri-apps/api`) directly and detect Tauri via that import. Decision:
import the API directly (no reliance on the injected global).

```
Existing React/Vite app (device-library call sites UNCHANGED)
  netmd-js  ->  navigator.usb
                    |
        navigator.usb = NativeUSB shim   (installed at boot under Tauri)
                    |
src/native-usb/  (new, TS)
  - polyfill.ts   installs navigator.usb, registers hotplug
  - device.ts     NativeUSBDevice (mimics WebUSB USBDevice)
  - picker.tsx    replaces requestDevice() chooser
      each method -> Tauri IPC
                    |
src-tauri/  (new, Rust)
  nusb-backed commands + raw/streamed binary IPC; hotplug -> JS event
```

## Prerequisite: version-exact WebUSB API audit (do before coding)

The review could not confirm polyfill coverage because `node_modules` isn't
present. **Before implementation**, extract the pinned tarballs (`netmd-js`
4.4.1, `himd-js` 0.2.12, `netmd-exploits`, `networkwm-js` per
`package-lock.json`) and build a call-site matrix of every `navigator.usb` /
`USBDevice` member they touch, with exact semantics for:

- control transfer **`recipient`** field (device/interface/endpoint) — not just
  `requestType`/`request`/`value`/`index`.
- `USBDevice.reset()` — a real reset re-enumerates the device and **invalidates
  the handle**; the shim + Rust state must model re-open, not silent reuse.
- interface claim / release, `selectConfiguration`, alternate settings.
- descriptor reads (`configuration.interfaces[].alternate.endpoints[]`).
- status/stall handling (`clearHalt`).
- disconnect **device identity** — how the app correlates an `ondisconnect`
  event to the open device.

The command table below is the starting point; the audit is authoritative and
may add members.

## Component: Rust USB backend (`src-tauri`)

`nusb`-backed. State keyed by an integer handle. **State model must account for
`nusb` semantics**, not just a flat map:

- exclusive interface claim (claim/release lifecycle, one owner).
- endpoint queue lifecycle (in-flight transfers on unplug).
- handle invalidation on `reset()` and on unplug (subsequent calls reject, not
  hang or use-after-free).

Command surface (starting point; finalized by the API audit):

| Command | Args | Returns |
|---|---|---|
| `usb_list_devices` | — | `[{deviceId, vendorId, productId, serial, manufacturer, product}]` |
| `usb_open` | `deviceId` | `handle` |
| `usb_close` | `handle` | — |
| `usb_select_configuration` | `handle, configValue` | — |
| `usb_claim_interface` | `handle, iface` | — |
| `usb_release_interface` | `handle, iface` | — |
| `usb_control_in` | `handle, {recipient,requestType,request,value,index}, length` | `bytes` |
| `usb_control_out` | `handle, setup, data` | `bytesWritten` |
| `usb_bulk_in` | `handle, endpoint, length, timeoutMs` | `bytes` |
| `usb_bulk_out` | `handle, endpoint, data, timeoutMs` | `bytesWritten` |
| `usb_clear_halt` | `handle, direction, endpoint` | — |
| `usb_reset` | `handle` | — (handle invalidated; caller re-opens) |
| `usb_descriptors` | `handle` | config/interface/endpoint tree |

## Component: binary IPC (corrected)

**Prior draft was wrong.** Plain Tauri command args/returns are JSON-serialized;
Tauri explicitly warns large returns are slow. `Vec<u8>` return does **not**
give a raw path. Use the real binary mechanisms:

- **Bulk read → JS:** return `tauri::ipc::Response::new(bytes)` (raw response
  body, no JSON).
- **Bulk write ← JS:** send the payload as a raw request body via
  `tauri::ipc::Request` / `InvokeBody::Raw` (`ArrayBuffer` from JS), not a JSON
  number array. Matters directly — track upload streams `ArrayBuffer` data
  (`src/services/interfaces/himd.ts:531`, and the NetMD transfer path).
- **Streaming large transfers:** `Channel` for chunked progress on multi-MB
  reads/writes.

Shim mirrors WebUSB return shapes exactly:
- `controlTransferIn` / `transferIn` → `{ data: DataView, status: 'ok' }`
- `controlTransferOut` / `transferOut` → `{ bytesWritten, status: 'ok' }`

**Benchmark on real NetMD hardware** before locking the design — chatty control
polling + multi-MB bulk are the two perf-sensitive paths.

## Component: device picker & hotplug

`requestDevice(filters)` replacement:

1. `usb_list_devices`, filter by caller's `{vendorId, productId}` filters.
2. **0 matches** → reject `NotFoundError` (matches WebUSB on cancel).
3. **1 match** → return it directly, no prompt.
4. **2+ matches** → `picker.tsx` modal (mounted once at boot) lists product +
   serial; user picks.

Permission model: none. Non-sandboxed hardened-runtime app → `nusb` sees
vendor-class devices with no entitlement. `getDevices()` returns the current
live matching list.

Hotplug: Rust `nusb` hotplug watch → emits Tauri event `usb-disconnect`
carrying a stable device identity (per the API audit). Shim builds a
WebUSB-shaped `{ device }` event → `navigator.usb.ondisconnect`
(`src/index.tsx:46`). Connect events not wired (app doesn't auto-connect).

## Component: native-bridge feature parity (new — was under-stated)

A `navigator.usb` shim alone is **not** full parity. The app already has an
Electron-oriented native bridge (`window.native.*` — `src/bridge-types.d.ts:52`,
consumed in `src/services/interface-service-manager.ts:30`). Each capability
gets an explicit disposition for the Tauri build:

| Capability (source) | v1 disposition |
|---|---|
| `window.native.interface` (NetMD native path) | **Redesign** — served by the `navigator.usb` shim; keep the web WebUSB path for the browser build. |
| `nwInterface` (NetworkWM) | Confirm during audit; likely web-path reuse. |
| `himdFullInterface` (Full HiMD native) | **Dropped from v1** — blocked by HiMD gate. |
| `unrestrictedFetchJSON` (Shazam CORS bypass, `src/redux/actions.ts:1102`) | **Redesign** — route via a Tauri HTTP command (no browser CORS); or disable Song Recognition in v1 if deferred. |
| Local ATRAC encoder bridge | Confirm need; web/WASM encoders already exist in-app. |
| Native dialogs | Optional; Tauri dialog plugin if used. |

Anything not explicitly retained/redesigned is **dropped from v1** and noted, so
scope is a choice, not an accident.

## Component: web-stack integration (WASM, workers, headers)

App loads FFmpeg workers (`src/utils.ts:540`), ATRAC workers
(`src/services/audio/atrac3os-worker.ts:151`, `atracdenc-worker.ts:43`), v86,
`.wasm`. `SharedArrayBuffer` needs cross-origin isolation.

1. **Prod headers:** Tauri ≥2.1.0 `app.security.headers` → COOP `same-origin` +
   COEP `require-corp` on protocol responses (documented to enable SAB).
2. **Dev headers (gap):** the Vite dev server needs the same COOP/COEP headers
   set separately (`vite.config.ts` has none today) or dev is broken while prod
   works.
3. **COEP fallout:** every embedded cross-origin resource must be CORP/CORS-
   compatible — verify each remote feature endpoint, or it's blocked.
4. **Acceptance gate (not assumption):** a milestone test asserting
   `crossOriginIsolated === true`, `SharedArrayBuffer` present, and **each**
   worker/WASM path runs — in **both** dev and prod header configs.
5. **Worker/WASM paths** resolve against the Tauri protocol origin; confirm Vite
   `base` and worker URLs.

## Component: CSP

- Must include Tauri's own required `connect-src ipc: http://ipc.localhost`.
- **Remote NetMD uses a user-supplied `serverAddress`**
  (`src/services/interface-service-manager.ts:82`) → a static build-time
  `connect-src` allowlist can't cover it. Either relax `connect-src` for the
  remote-NetMD feature, make CSP configurable at runtime, or gate the feature.
  Decide during implementation; do not ship a static allowlist that silently
  breaks Remote NetMD.
- External ATRAC encoder + Shazam hosts added per the parity decisions above.

## Component: signing, notarization, distribution

1. **Bundle:** `tauri build` → `.app` + `.dmg`.
2. **Codesign:** Developer ID Application cert; hardened runtime on.
3. **USB under hardened runtime:** non-sandboxed → `nusb` USB access works with
   **no** entitlement (`com.apple.security.device.usb` is sandbox-only). Real
   blocker is kernel-driver ownership (HiMD), not hardened runtime.
4. **Privacy usage strings (gap):** the app uses `getUserMedia` for
   microphone/line-in recording (`src/services/browserintegration/
   mediarecorder.ts:46`). Info.plist **must** include
   `NSMicrophoneUsageDescription` or macOS terminates the app on capture access.
5. **Notarize + staple:** Tauri reads `APPLE_ID`/`APPLE_PASSWORD`/
   `APPLE_TEAM_ID` (or `APPLE_API_KEY`); submit + staple.
6. **Account prereqs (not code):** Apple Developer Program, Developer ID cert.
   If absent, build unsigned/ad-hoc `.app` (right-click-open), defer
   notarization — don't block dev on paperwork.
7. **Arch:** universal (arm64 + x86_64).

### Release checklist

- [ ] Hardened runtime enabled, no stray sandbox entitlements.
- [ ] `NSMicrophoneUsageDescription` present.
- [ ] Notarization submitted, ticket stapled, verified (`spctl`/`stapler`).
- [ ] Universal binary confirmed on Intel + Apple Silicon.

## Known risks

1. **HiMD kernel-driver ownership (v1 blocker → out of scope).** No demonstrated
   `nusb`/macOS strategy for exclusive access to the mass-storage interface. See
   HiMD gate. NetMD unaffected.
2. **Binary IPC perf.** Multi-MB bulk over IPC — validated by the corrected
   `Response`/raw-request/`Channel` design + real-hardware benchmark.
3. **Chatty NetMD control polling latency.** Each poll is an IPC round-trip;
   measure on real hardware early.
4. **Transfer timeouts / mid-op disconnect.** Bulk reads block; per-call
   timeouts + clean rejection so unplug/reset fails the promise, not hangs.

## HiMD gate (separate spike, not v1)

Before promising any Full HiMD support on macOS, prove exclusive access to a
kernel-driver-owned mass-storage interface — e.g. an IOKit-level approach
outside `nusb`, a different backend, or an authorized unmount+claim sequence.
If unproven, Full HiMD stays unsupported on native macOS and users keep using
the Chromium web app for it. This is a research task with a binary outcome, not
a build milestone.

## Milestones (revised, de-risk order)

1. **Tauri shell** wraps the current build, no USB. Acceptance gate: FFmpeg / v86
   / all workers / WASM run in WKWebView with `crossOriginIsolated`, dev **and**
   prod headers. Retires the web-stack risk.
2. **WebUSB API audit** — extract tarballs, produce the call-site matrix.
3. **NetMD USB bridge + shim** — one real NetMD device round-trips (list, open,
   read TOC, list tracks, transfer a track) over the corrected binary IPC; benchmarked.
4. **Native-bridge parity** — Shazam fetch redesign (or defer), Remote NetMD CSP
   decision, hotplug/picker.
5. **Sign / notarize / DMG** (universal) + release checklist.
6. **HiMD gate** — separate spike; only then decide native HiMD scope.

## Out of scope (deliberate)

Full HiMD on native macOS (pending gate); auto-update; Windows/Linux Tauri
builds (bridge is cross-platform, near-free later); Mac App Store; in-app
licensing.

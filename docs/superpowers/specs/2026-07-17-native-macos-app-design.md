# Native macOS App — Design Spec

**Date:** 2026-07-17
**Status:** Approved architecture; revised after external (Codex) review

## Goal

Ship Web MiniDisc Pro as a native macOS application: a small, notarized `.app`
distributed via DMG (not the Mac App Store). Reuse the existing React/Vite app
essentially unchanged.

## Scope (revised)

**v1 = NetMD + Full HiMD.** Both are in scope.

- **NetMD** is a USB vendor-class device — `nusb` claims it cleanly on macOS.
  Straightforward.
- **Full HiMD** enumerates as USB Mass Storage (class `0x08`); the macOS kernel
  driver owns that interface and auto-mounts the volume. `nusb` cannot detach a
  kernel driver on macOS. **But the ecosystem already solves this without
  needing to detach:** the `netmd-exploits` `HiMDUSBClassOverride` exploit
  firmware-patches the device's USB interface descriptor to **vendor class
  `0xFF`** (and VID/PID `"ASVR"`), so it re-enumerates as a non-mass-storage
  device that no kernel driver claims — on every OS, macOS included. This is
  exactly how the Chromium web app does Full HiMD on macOS today. See "HiMD USB
  access" below. Fallbacks exist if the bootstrap step misbehaves on macOS.

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

## Component: HiMD USB access (class-override bootstrap)

Full HiMD works by turning the device into a vendor-class device before doing
raw I/O. The exploit is pure JS in `netmd-exploits`, driven **through the same
`navigator.usb` shim** — no extra native surface beyond what the shim already
exposes, provided the shim supports the two things below.

Flow (matches the browser's proven macOS path):

1. **Bootstrap (the one macOS unknown).** Device is still mass-storage; the
   kernel driver owns the bulk interface. `netmd-exploits` enters HiMD factory
   mode and applies `HiMDUSBClassOverride` — patching the USB descriptor to
   class `0xFF`, VID/PID `"ASVR"`. These factory commands ride **EP0 control
   transfers**, which do **not** require claiming the owned bulk interface —
   which is why WebUSB (and macOS) permit them despite the class-08 block.
   **Requirement:** the shim/backend must support **device-level control
   transfers** independent of interface claim.
2. **Re-enumeration.** After the patch the device drops and reappears as an
   `ASVR` vendor-class device. **Requirement:** the shim must survive this —
   `ondisconnect` fires for the old device, hotplug surfaces the new `ASVR`
   device, and the flow re-opens it. himd-js/webminidisc already orchestrate
   this dance in the browser; the shim must not swallow it.
3. **Direct SCSI.** The `ASVR` device is vendor class → `nusb` claims it
   cleanly (identical to NetMD from here) → himd-js Direct mode issues SCSI over
   bulk. Full read/write/DRM works.

**The single de-risking question:** can `nusb` issue the factory EP0 control
transfers on macOS while the device is still mass-storage (kernel driver owning
the bulk interface)? Chrome proves it's possible in principle; `nusb` must be
confirmed on real hardware early (Milestone 3a).

**Fallbacks, in order, if the `nusb` EP0 bootstrap fails on macOS:**

1. **`rusb` (libusb) backend.** libusb added macOS kernel-driver detach via
   whole-device capture (libusb PR #911). Swap the Rust backend from `nusb` to
   `rusb` for HiMD (or entirely) and use device capture to reach EP0 / claim.
2. **Restricted (volume) mode.** himd-js supports pointing at the mounted
   `/Volumes/<disc>` filesystem — read-only/filesystem features, no raw USB.
   Feature floor if full DRM transfer can't be bootstrapped.

Decision: build on `nusb` + EP0 bootstrap first (Milestone 3a proves it); keep
`rusb` capture as the designed-in fallback, restricted mode as the floor.

## Component: native-bridge feature parity (new — was under-stated)

A `navigator.usb` shim alone is **not** full parity. The app already has an
Electron-oriented native bridge (`window.native.*` — `src/bridge-types.d.ts:52`,
consumed in `src/services/interface-service-manager.ts:30`). Each capability
gets an explicit disposition for the Tauri build:

| Capability (source) | v1 disposition |
|---|---|
| `window.native.interface` (NetMD native path) | **Redesign** — served by the `navigator.usb` shim; keep the web WebUSB path for the browser build. |
| `nwInterface` (NetworkWM) | Confirm during audit; likely web-path reuse. |
| `himdFullInterface` (Full HiMD native) | **In v1** — served by the `navigator.usb` shim + class-override bootstrap (see "HiMD USB access"). |
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
   **no** entitlement (`com.apple.security.device.usb` is sandbox-only). HiMD's
   kernel-driver ownership is handled by the class-override bootstrap (see "HiMD
   USB access"), not by entitlements. If the `rusb`/libusb capture fallback is
   used, verify device capture also needs no special entitlement outside the
   sandbox.
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

1. **HiMD EP0 control bootstrap on macOS (single biggest unknown).** Can `nusb`
   issue the factory EP0 control transfers while the mass-storage kernel driver
   owns the bulk interface? Proven possible in Chrome; must be confirmed on real
   hardware at Milestone 3a. Fallbacks designed in: `rusb`/libusb device
   capture, then restricted (volume) mode. See "HiMD USB access."
2. **Re-enumeration handling.** After class override the device disconnects and
   reappears as `ASVR`; the shim must survive the drop/reappear cycle, not treat
   it as a fatal disconnect.
3. **Binary IPC perf.** Multi-MB bulk over IPC — validated by the corrected
   `Response`/raw-request/`Channel` design + real-hardware benchmark.
4. **Chatty NetMD control polling latency.** Each poll is an IPC round-trip;
   measure on real hardware early.
5. **Transfer timeouts / mid-op disconnect.** Bulk reads block; per-call
   timeouts + clean rejection so unplug/reset fails the promise, not hangs.

## Milestones (revised, de-risk order)

1. **Tauri shell** wraps the current build, no USB. Acceptance gate: FFmpeg / v86
   / all workers / WASM run in WKWebView with `crossOriginIsolated`, dev **and**
   prod headers. Retires the web-stack risk.
2. **WebUSB API audit** — extract tarballs, produce the call-site matrix
   (include device-level control transfers needed for the HiMD bootstrap).
3. **NetMD USB bridge + shim** — one real NetMD device round-trips (list, open,
   read TOC, list tracks, transfer a track) over the corrected binary IPC;
   benchmarked.
   - **3a. HiMD bootstrap spike (do early — highest-risk item).** On real HiMD
     hardware: `nusb` EP0 control transfers → apply `HiMDUSBClassOverride` →
     survive re-enumeration → claim `ASVR` device. If it fails, switch to the
     `rusb`/libusb capture fallback before building further HiMD on `nusb`.
4. **HiMD full path** — Direct SCSI read/write/DRM through the shim on the
   `ASVR` device; restricted (volume) mode wired as the read-only floor.
5. **Native-bridge parity** — Shazam fetch redesign (or defer), Remote NetMD CSP
   decision, hotplug/picker.
6. **Sign / notarize / DMG** (universal) + release checklist.

## Out of scope (deliberate)

Auto-update; Windows/Linux Tauri builds (bridge is cross-platform, near-free
later); Mac App Store; in-app licensing. (Full HiMD is now **in** v1.)

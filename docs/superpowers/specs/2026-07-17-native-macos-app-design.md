# Native macOS App — Design Spec

**Date:** 2026-07-17
**Status:** Approved, pre-implementation

## Goal

Ship Web MiniDisc Pro as a native macOS application: a small, notarized `.app`
distributed via DMG (not the Mac App Store). Reuse the existing React/Vite app
essentially unchanged.

## Constraints & decisions

- **Not the Mac App Store.** The repo is GPL-2.0 (authored by `asivery`; this is
  a fork). GPLv2 §6 is incompatible with the App Store's usage/DRM terms (the
  known VLC/FSF case), and we are not the copyright holder for the app or its
  GPL dependencies (`netmd-js`, `himd-js`, `netmd-exploits`, `networkwm-js`).
  Dropping the App Store removes this legal blocker. Distribution is via
  notarized DMG only.
- **Native, not Electron.** Target a small native shell via **Tauri v2**
  (system WKWebView + Rust). No bundled Chromium.
- **WebUSB is the linchpin.** WKWebView has no WebUSB. NetMD/HiMD access
  currently flows through `navigator.usb`. We replace `navigator.usb` with a
  native USB bridge rather than editing the device libraries.

## Guiding principle: polyfill, don't fork

Install a drop-in `navigator.usb` replacement at startup, **only** when running
under Tauri (`window.__TAURI__` present). The web build keeps using real WebUSB;
the Tauri build uses the shim. Existing app code (`src/services/interfaces/
netmd.ts`, `himd.ts`, `netmd-js`, `himd-js`) stays untouched. Smallest diff, and
the web app keeps working.

```
Existing React/Vite app (UNCHANGED)
  netmd-js / himd-js  ->  navigator.usb
                              |
        navigator.usb = NativeUSB shim   (installed at boot iff window.__TAURI__)
                              |
src/native-usb/  (new, TS)
  - polyfill.ts   installs navigator.usb, registers hotplug
  - device.ts     NativeUSBDevice (mimics WebUSB USBDevice)
  - picker.tsx    replaces requestDevice() chooser
      each method -> invoke('usb_*')
                              |
                    Tauri IPC (commands + events)
                              |
src-tauri/  (new, Rust)
  nusb-backed commands: list/open/close/claim/control/bulk; hotplug -> JS event
```

Three independently testable units:

- **Rust USB backend** — raw libusb ops via `nusb`, exposed as Tauri commands.
  Knows nothing about MiniDisc.
- **JS shim** — faithfully mimics the WebUSB `USBDevice` interface the libraries
  consume; translates to IPC. Knows nothing about the UI.
- **Boot polyfill** — one guarded line: if Tauri, set `navigator.usb` = shim and
  register the hotplug event.

## Repo layout

- `src-tauri/` — new Rust project; produces the `.app`/`.dmg`. Its
  `beforeBuildCommand`/`beforeDevCommand` call the existing npm build/dev
  scripts; `dist/` is reused as the frontend dist.
- `src/native-usb/` — new: polyfill, `NativeUSBDevice`, device picker.
- Existing Vite pipeline reused. Likely a one-line `base` tweak for Tauri.

## Component: Rust USB backend (`src-tauri`)

`nusb`-backed Tauri commands. State = `Mutex<HashMap<u32, DeviceHandle>>` in
Tauri managed state; `usb_open` returns an integer handle.

| Command | Args | Returns |
|---|---|---|
| `usb_list_devices` | — | `[{deviceId, vendorId, productId, serial, manufacturer, product}]` |
| `usb_open` | `deviceId` | `handle: u32` |
| `usb_close` | `handle` | — |
| `usb_select_configuration` | `handle, configValue` | — |
| `usb_claim_interface` | `handle, iface` | — |
| `usb_release_interface` | `handle, iface` | — |
| `usb_control_in` | `handle, {requestType,request,value,index}, length` | `bytes` |
| `usb_control_out` | `handle, setup, data` | `bytesWritten` |
| `usb_bulk_in` | `handle, endpoint, length, timeoutMs` | `bytes` |
| `usb_bulk_out` | `handle, endpoint, data, timeoutMs` | `bytesWritten` |
| `usb_clear_halt` | `handle, direction, endpoint` | — |
| `usb_reset` | `handle` | — |
| `usb_descriptors` | `handle` | config/interface/endpoint tree |

## Component: JS WebUSB shim (`src/native-usb`)

Mirror WebUSB return shapes **exactly** or netmd-js breaks:

- `controlTransferIn` -> `{ data: DataView, status: 'ok' }`
- `transferIn` -> `{ data: DataView, status: 'ok' }`
- `transferOut` / `controlTransferOut` -> `{ bytesWritten, status: 'ok' }`
- `device.configuration.interfaces[].alternate.endpoints[]` populated from
  `usb_descriptors` (netmd-js reads endpoint numbers/directions).

**Binary path.** Track download/upload = multi-MB bulk. Use **Tauri v2** raw
byte IPC (`Vec<u8>` / `&[u8]`, not base64/JSON arrays). Shim wraps in
`DataView`/`Uint8Array`.

## Component: device picker & hotplug

`requestDevice(filters)` replacement flow:

1. Shim calls `usb_list_devices`, filters by caller's `{vendorId, productId}`
   filters (netmd-js/himd-js pass known NetMD/HiMD VID/PID lists).
2. **0 matches** -> reject with `NotFoundError` (matches WebUSB on cancel);
   existing "no device" UI shows.
3. **1 match** -> return it directly, no prompt.
4. **2+ matches** -> small React modal (`picker.tsx`, mounted once at boot)
   lists product + serial; user picks; returns chosen `NativeUSBDevice`.

Permission model: none. Non-sandboxed hardened-runtime app -> libusb sees
devices directly; no per-site grant. `getDevices()` returns the current live
matching list.

Hotplug: Rust registers `nusb` hotplug watch -> emits Tauri event
`usb-disconnect` with `deviceId`. Shim listens, builds a WebUSB-shaped
`{ device }` event, invokes `navigator.usb.ondisconnect` (consumed at
`src/index.tsx:46`). Connect events not wired — the app doesn't auto-connect
(YAGNI).

## Component: web-stack integration

The app loads ffmpeg (`@ffmpeg/ffmpeg` 0.6.1), `atracdenc.js`, `v86` (x86 emu
for ATRAC encode), multiple Web Workers, `.wasm`, `assembler.wasm`.

1. **Cross-origin isolation.** `SharedArrayBuffer` (v86, possibly ffmpeg worker)
   needs COOP `same-origin` + COEP `require-corp`. Set via `app.security.headers`
   in `tauri.conf.json`. **Verify need first** at prototype time — old ffmpeg
   0.6.1 may run asm.js without SAB. Add headers only if a worker throws.
2. **COEP fallout.** With COEP on, sub-resources (workers, wasm, images) need
   `Cross-Origin-Resource-Policy`. Bundled Tauri-protocol assets are same-origin
   (fine); verify remote `fetch` features don't break.
3. **Worker + WASM paths.** Workers load from `/worker.min.js`,
   `/recorderWorker.js`, `public/*`. Confirm Vite `base` and worker URLs resolve
   against the Tauri protocol origin (likely a one-line `base` tweak).
4. **Remote features.** Remote NetMD, external ATRAC encoder, Shazam API =
   outbound HTTP/WS. Tauri CSP blocks by default -> allow needed hosts in
   `tauri.conf.json` CSP `connect-src`, enumerated from the services layer at
   build time.

## Component: signing, notarization, distribution

1. **Bundle.** `tauri build` -> `.app` + `.dmg`.
2. **Codesign.** Developer ID Application cert; **hardened runtime** on.
3. **USB under hardened runtime.** Non-sandboxed + hardened runtime -> libusb/
   IOKit USB access works with **no special entitlement**
   (`com.apple.security.device.usb` is sandbox-only). No USB prompt for
   vendor-class NetMD.
4. **Notarize + staple.** Tauri reads `APPLE_ID`/`APPLE_PASSWORD`/`APPLE_TEAM_ID`
   (or `APPLE_API_KEY`) env vars; submit to notary; staple ticket.
5. **Account prereqs (not code):** Apple Developer Program, Developer ID cert.
   If absent, build an unsigned/ad-hoc `.app` (right-click-open) and defer
   notarization — do not block dev on paperwork.
6. **Arch.** Build `universal` (arm64 + x86_64).

## Known risks

1. **HiMD mass-storage interface stealing (highest device risk).** NetMD is
   vendor-class -> libusb claims cleanly. **HiMD enumerates as USB Mass Storage
   -> macOS auto-mounts and the kernel driver owns the interface.** libusb must
   detach it / the volume must be unmounted first. May need an unmount step or a
   documented "eject first" instruction. NetMD unaffected.
2. **Chatty protocol latency.** NetMD polls status via many small control
   transfers; each is now an IPC round-trip. Expected fine; measure on a real
   device early.
3. **Transfer timeouts / mid-op disconnect.** Bulk reads block until data. Need
   per-call timeouts + clean error propagation so an unplug rejects the promise
   rather than hanging.

## Milestones (de-risk order)

1. **Tauri shell** wraps the *current* build, no USB. Prove ffmpeg/v86/workers/
   headers render and run in WKWebView. Retires the web-stack risk first.
2. **USB bridge + shim** — one real NetMD device round-trips (list, open, read
   TOC, list tracks).
3. **HiMD + hotplug + picker** edge cases (incl. mass-storage unmount).
4. **Sign / notarize / DMG** (universal).

## Out of scope (deliberate)

Auto-update; Windows/Linux Tauri builds (bridge is cross-platform, so
near-free later); Mac App Store; in-app licensing.

# Native macOS Shell (Plan A) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a running native macOS app (Tauri v2 + system WKWebView) that loads the existing Web MiniDisc Pro frontend unchanged, with cross-origin isolation working so all WASM/worker features run — and produce the WebUSB API audit that unblocks the USB bridge (Plan B).

**Architecture:** Tauri v2 wraps the existing Vite build with zero frontend changes. Tauri's `beforeDevCommand`/`beforeBuildCommand` drive the existing npm scripts; the built `dist/` is served over Tauri's asset protocol. Cross-origin isolation headers (COOP/COEP) are set in both the Vite dev server and Tauri's production asset responses so `SharedArrayBuffer`-dependent features (v86, ffmpeg, atracdenc) work. This plan adds **no USB code** — that is Plan B.

**Tech Stack:** Tauri v2 (Rust), existing React 18 + Vite 6 frontend, `@tauri-apps/cli`, macOS WKWebView.

## Global Constraints

- Tauri version floor: **≥ 2.1.0** (required for `app.security.headers`). Copy verbatim into `Cargo.toml`/`package.json` dependency ranges.
- Distribution target: **not** the Mac App Store — no App Sandbox, no `com.apple.security.device.usb` entitlement.
- Frontend device-library call sites stay **unchanged** in this plan. No edits to `src/services/interfaces/*.ts`, `netmd-js`, or `himd-js`.
- Tauri detection in later plans imports `@tauri-apps/api` directly rather than relying on the injected `window.__TAURI__` global; this plan does not set `withGlobalTauri`.
- The app is GPL-2.0. No AI attribution in commits.
- Vite dev server runs on port **5173** (Vite default); Tauri `devUrl` must match.

**Prerequisites (developer machine, one-time — not build steps):**
- macOS with Xcode Command Line Tools (`xcode-select --install`).
- Rust toolchain via `rustup` (`curl https://sh.rustup.rs -sSf | sh`).
- Node + npm as already used by the repo.

---

### Task 1: Scaffold Tauri v2 shell wired to the existing Vite build

**Files:**
- Create: `src-tauri/` (generated: `Cargo.toml`, `tauri.conf.json`, `src/main.rs`, `src/lib.rs`, `build.rs`, `capabilities/default.json`, `icons/`)
- Modify: `package.json` (add `@tauri-apps/cli` dev dependency + `tauri` script)
- Modify: `.gitignore` (ignore `src-tauri/target`)

**Interfaces:**
- Consumes: existing npm scripts `dev` and `build`; existing `dist/` output directory.
- Produces: a runnable `npm run tauri dev` that opens a native window loading the app; `npm run tauri build` producing an `.app`/`.dmg`. Later plans add Rust commands to `src-tauri/src/lib.rs`.

- [ ] **Step 1: Install the Tauri CLI as a dev dependency**

Run:
```bash
npm install -D @tauri-apps/cli@^2.1.0
```
Expected: `@tauri-apps/cli` appears in `package.json` devDependencies; no errors.

- [ ] **Step 2: Add the `tauri` npm script**

Edit `package.json` `scripts`, add:
```json
"tauri": "tauri"
```
(Place alongside the existing `"start"`, `"dev"`, `"build"` scripts.)

- [ ] **Step 3: Scaffold `src-tauri/` non-interactively**

Run:
```bash
npm run tauri init -- --ci \
  --app-name "Web MiniDisc Pro" \
  --window-title "Web MiniDisc Pro" \
  --frontend-dist ../dist \
  --dev-url http://localhost:5173 \
  --before-dev-command "npm run dev" \
  --before-build-command "npm run build"
```
Expected: a new `src-tauri/` directory with `Cargo.toml`, `tauri.conf.json`, `src/main.rs`, `src/lib.rs`, `build.rs`, `icons/`.

- [ ] **Step 4: Set the bundle identifier and macOS bundle targets**

Edit `src-tauri/tauri.conf.json`. Set the identifier and bundle block:
```json
{
  "identifier": "wiki.minidisc.webminidisc",
  "bundle": {
    "active": true,
    "targets": ["app", "dmg"],
    "category": "Music",
    "icon": ["icons/32x32.png", "icons/128x128.png", "icons/icon.icns"]
  }
}
```
(Merge into the generated file; keep the generated `build`, `app`, and `$schema` keys.)

- [ ] **Step 5: Ignore the Rust build output**

Add to `.gitignore`:
```
src-tauri/target
```

- [ ] **Step 6: Run the dev shell and verify the app renders**

Run:
```bash
npm run tauri dev
```
Expected: Rust compiles (first build is slow), a native window titled "Web MiniDisc Pro" opens, and the Web MiniDisc Pro welcome UI renders inside it. The device features will not work yet (no USB) — that is expected. Close the window to stop.

Note: this is a manual verification step. There is no automated test — Tauri scaffolding produces framework code, and rendering can only be confirmed by launching the shell.

- [ ] **Step 7: Commit**

```bash
git add package.json package-lock.json .gitignore src-tauri
git commit -m "Scaffold Tauri v2 shell wrapping the existing Vite build"
```

---

### Task 2: Cross-origin isolation (dev + production headers)

**Files:**
- Modify: `src-tauri/tauri.conf.json` (`app.security.headers`)
- Modify: `vite.config.ts:14-92` (add `server.headers` + `strictPort`)

**Interfaces:**
- Consumes: the Tauri shell from Task 1.
- Produces: `window.crossOriginIsolated === true` in both the Vite dev server (used by `tauri dev`) and the Tauri production build. This is the precondition for `SharedArrayBuffer`, which v86/ffmpeg/atracdenc workers rely on.

- [ ] **Step 1: Add production isolation headers to Tauri**

Edit `src-tauri/tauri.conf.json`, in `app.security`:
```json
"security": {
  "csp": null,
  "headers": {
    "Cross-Origin-Opener-Policy": "same-origin",
    "Cross-Origin-Embedder-Policy": "require-corp"
  }
}
```
(`csp: null` for now — CSP is designed in Plan D. Keep any generated `security` sub-keys.)

- [ ] **Step 2: Add dev-server isolation headers to Vite**

Edit `vite.config.ts`. Inside the `defineConfig({ ... })` object (sibling to `base`, `plugins`, `build`), add:
```ts
server: {
  port: 5173,
  strictPort: true,
  headers: {
    'Cross-Origin-Opener-Policy': 'same-origin',
    'Cross-Origin-Embedder-Policy': 'require-corp',
  },
},
```
`strictPort` guarantees the port matches Tauri's `devUrl`; without it Vite may pick another port and the shell loads a blank page.

- [ ] **Step 3: Verify isolation in dev**

Run `npm run tauri dev`. When the window opens, open the WebView devtools (right-click → Inspect Element, or `Cmd+Option+I`), and in the console run:
```js
crossOriginIsolated
```
Expected: `true`. If `false`, the headers are not being applied — recheck Steps 1–2.

- [ ] **Step 4: Verify isolation in a production build**

Run:
```bash
npm run tauri build
```
Then launch the built app:
```bash
open "src-tauri/target/release/bundle/macos/Web MiniDisc Pro.app"
```
Open devtools in the running app (devtools are available in release only if enabled; if not, temporarily run `npm run tauri build -- --debug` which keeps devtools) and confirm `crossOriginIsolated === true` in the console.

Expected: `true`, proving the Tauri `headers` config applies to the production asset protocol.

- [ ] **Step 5: Commit**

```bash
git add src-tauri/tauri.conf.json vite.config.ts
git commit -m "Enable cross-origin isolation in dev and production for SharedArrayBuffer"
```

---

### Task 3: WKWebView web-stack acceptance gate (WASM, workers, v86)

**Files:**
- Modify (only if remediation is needed): `src-tauri/tauri.conf.json`, `index.html`, or worker-loading sites surfaced during verification.
- Create: `docs/superpowers/plans/notes/plan-a-webstack-acceptance.md` (record the result)

**Interfaces:**
- Consumes: the isolated shell from Task 2.
- Produces: a documented confirmation that every WASM/worker feature runs under WKWebView, or a recorded remediation. This retires the largest non-USB risk before Plan B starts.

- [ ] **Step 1: Exercise ffmpeg (audio decode/convert)**

In the running `npm run tauri dev` shell: add a common audio file (e.g. an MP3 or WAV) to the app via the drop zone / file picker. Confirm the app decodes it and shows a track ready to transfer (no console errors referencing ffmpeg, wasm, or COEP).

Expected: the track appears with correct duration. Any `Cross-Origin-Embedder-Policy` / `SharedArrayBuffer is not defined` / worker load error is a failure — record it in Step 5.

- [ ] **Step 2: Exercise the atracdenc / v86 LP encoders**

In the app settings, select an LP encoding mode (LP2 or LP4), which routes through the atracdenc worker and/or the v86 x86 emulator (`public/atrac3vm`). Start encoding a short track.

Expected: encoding progresses and completes. Watch the console for worker instantiation errors, `wasm` compile/instantiate failures, or `SharedArrayBuffer` errors.

- [ ] **Step 3: Exercise the high-quality client-side encoder path**

If the app exposes the client-side ATRAC3 encoder (per README "high-quality client-side encoder"), run one encode through it. Confirm no worker/wasm/COEP errors.

- [ ] **Step 4: If any COEP breakage surfaced, remediate**

COEP `require-corp` blocks any sub-resource lacking a Cross-Origin-Resource-Policy. Bundled assets served by Tauri are same-origin (normally fine). Concrete remediations, applied only to the specific failing resource:
- A worker/script that fails to load cross-origin: ensure it is loaded from the app origin (relative URL under `public/`), not an absolute external URL.
- A `wasm`/asset fetched from a remote origin: either bundle it under `public/` so it is same-origin, or (only if it must stay remote) mark the fetch/element `crossorigin` and confirm the remote sends CORP/CORS.
- If a needed resource genuinely cannot send CORP, evaluate `Cross-Origin-Embedder-Policy: credentialless` instead of `require-corp` in both `vite.config.ts` and `tauri.conf.json`, and re-verify `crossOriginIsolated` is still `true`.

Re-run Steps 1–3 after any change.

- [ ] **Step 5: Record the acceptance result**

Create `docs/superpowers/plans/notes/plan-a-webstack-acceptance.md` with: which features were exercised, pass/fail for each, macOS version, Apple-Silicon vs Intel, and any remediation applied. This is the evidence that Milestone 1's acceptance gate passed.

- [ ] **Step 6: Commit**

```bash
git add docs/superpowers/plans/notes/plan-a-webstack-acceptance.md src-tauri tauri.conf.json vite.config.ts index.html 2>/dev/null; git add -A
git commit -m "Verify WASM/worker/v86 stack runs under WKWebView; record acceptance"
```

---

### Task 4: WebUSB / USBDevice API audit artifact

**Files:**
- Create: `docs/superpowers/specs/webusb-api-matrix.md`

**Interfaces:**
- Consumes: the pinned dependency versions in `package-lock.json` (`netmd-js` 4.4.1, `himd-js` 0.2.12, `netmd-exploits`, `networkwm-js`).
- Produces: the authoritative call-site matrix of every `navigator.usb` / `USBDevice` member the frontend and its device libraries touch, with exact semantics. **Plan B's Rust command surface and JS shim are derived from this document**, so it must exist before Plan B starts.

- [ ] **Step 1: Populate `node_modules` at the pinned versions**

Run:
```bash
npm ci
```
Expected: installs exactly the `package-lock.json` versions, including `netmd-js`, `himd-js`, `netmd-exploits`, `networkwm-js` under `node_modules/`.

- [ ] **Step 2: Enumerate `navigator.usb` usage across app + libraries**

Run:
```bash
grep -rnoE "navigator\.usb\.[a-zA-Z]+|\.(controlTransferIn|controlTransferOut|transferIn|transferOut|claimInterface|releaseInterface|selectConfiguration|selectAlternateInterface|clearHalt|reset|open|close)\b|requestDevice|getDevices|ondisconnect" \
  src node_modules/netmd-js node_modules/himd-js node_modules/netmd-exploits node_modules/networkwm-js 2>/dev/null \
  | sort | uniq -c | sort -rn
```
Expected: a frequency list of every USB method invoked. Save the raw output to paste into the matrix.

- [ ] **Step 3: Extract control-transfer setup usage (recipient/requestType)**

Run:
```bash
grep -rnE "controlTransfer(In|Out)\s*\(" -A 6 src node_modules/netmd-js node_modules/himd-js node_modules/netmd-exploits 2>/dev/null | grep -iE "recipient|requestType|request:|value|index|'device'|'interface'|'endpoint'|'vendor'|'standard'|'class'"
```
Expected: the exact `{ requestType, recipient, request, value, index }` shapes passed — needed so the Rust `usb_control_*` commands include the `recipient` field the WebUSB spec requires.

- [ ] **Step 4: Confirm descriptor and device-identity reads**

Run:
```bash
grep -rnE "\.(configuration|configurations|interfaces|alternate|endpoints|endpointNumber|direction|vendorId|productId|serialNumber|deviceVersionMajor)\b" node_modules/netmd-js node_modules/himd-js 2>/dev/null | head -60
```
Expected: the `USBDevice` descriptor properties the libraries read, so the shim's `usb_descriptors` return shape covers them.

- [ ] **Step 5: Confirm the HiMD bootstrap USB surface (Plan C dependency)**

Run:
```bash
grep -rnE "controlTransfer|transferIn|transferOut|claimInterface|factoryIface|USBCodeExecution|open\(|reset\(" node_modules/netmd-exploits/dist 2>/dev/null | head -60
```
Expected: identifies whether the `HiMDUSBClassOverride` path uses EP0 control transfers vs. a claimed bulk interface — the single de-risking fact for Plan C's Milestone 3a.

- [ ] **Step 6: Write the matrix document**

Create `docs/superpowers/specs/webusb-api-matrix.md` with a table, one row per distinct USB member found:

| Member | Called by (file) | Args / setup fields | WebUSB return shape | Native/shim notes |
|---|---|---|---|---|

Fill every row from Steps 2–5 output. For `controlTransferIn/Out` rows, record the `recipient`. For `reset`, note handle-invalidation/re-enumeration. For `ondisconnect`, note how the app correlates the event to the open device. Add a short "HiMD bootstrap" subsection stating, from Step 5, whether the class-override commands ride EP0 control transfers.

Expected: no member found in Steps 2–5 is missing a row.

- [ ] **Step 7: Commit**

```bash
git add docs/superpowers/specs/webusb-api-matrix.md
git commit -m "Add WebUSB/USBDevice API audit matrix for the native USB bridge"
```

---

## Later plans (written after this one lands)

- **Plan B — NetMD USB bridge** (spec Milestone 3): Rust `nusb` backend commands + `NativeUSBDevice` shim + `navigator.usb` polyfill + device picker + hotplug. Derived from Task 4's matrix. Requires real NetMD hardware. The shim is unit-tested against a mocked Tauri `invoke`; end-to-end is manual hardware verification (round-trip a track).
- **Plan C — HiMD** (spec Milestones 3a–4): device-level EP0 control transfers, `HiMDUSBClassOverride` bootstrap, re-enumeration handling, Direct SCSI path, restricted (volume) mode floor; `rusb`/libusb capture fallback. Requires real HiMD hardware.
- **Plan D — Parity + ship** (spec Milestones 5–6): `unrestrictedFetchJSON` → Tauri HTTP command (Shazam), Remote NetMD runtime CSP, `NSMicrophoneUsageDescription`, Developer ID signing + notarization + universal DMG + release checklist.

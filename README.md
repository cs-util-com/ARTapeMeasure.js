# Pocket Tape AR 

---

## 1. Product Overview

**Pocket Tape AR** is a browser-based, two-point distance-measurement tool that runs on any WebXR-capable mobile browser. Users tap twice on real-world surfaces to place endpoints; the app draws a coloured 3-D line, shows the distance, and adds the entry to a session list. It aims at DIY hobbyists **and** professionals by providing a single, high-precision interface that is still lightweight and installable as a PWA.

---

## 2. Core Use-flow

| Step                                   | AR Scene                                                                   | HUD / 2-D UI                                                                                                                           | Notes                                                |
| -------------------------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| **Launch**                             | WebXR session starts in portrait mode (or “AR not supported” overlay).     | Top-center banner empty. FAB (➕ List) in bottom-right.                                                                                 | Camera permission requested only if not yet granted. |
| **Place 1st point**                    | White sphere marker appears.                                               | Banner shows live distance = 0.00 m / 0.00 ft.                                                                                         | Haptic tick.                                         |
| **Aim**                                | Reticle follows hit-test. Live provisional line connects marker → reticle. | Banner updates every frame (fixed precision).                                                                                          | Banner only—no in-scene label yet.                   |
| **Place 2nd point**                    | • Second sphere marker                                                     |                                                                                                                                        |                                                      |
| • Coloured 0.03 m line                 |                                                                            |                                                                                                                                        |                                                      |
| • Mid-point label billboard (3-D text) | Banner freezes to final value, then clears on next measurement.            | Haptic tick.                                                                                                                           |                                                      |
| **Session list**                       | –                                                                          | Tap FAB → modal sheet slides up. Each row: coloured dot, value + units, copy ❐, delete ✖, unit-toggle switch (Metric/Imperial) at top. | Modal closes when done.                              |

---

## 3. Functional Requirements

| Area               | Requirement                                                                                                                                                                                                                   |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Measurement        | • Only straight-line, two-point measurements.<br>• Unlimited concurrent lines.<br>• Internal unit = metres (Float64).                                                                                                         |
| Precision & Format | • Metric: metres to **2 decimals** (≥ 1 m) or centimetres (cm) under 1 m.<br>• Imperial: feet + decimal inches (e.g., 4 ft 7.25 in).                                                                                          |
| Units              | • Conversion occurs at display layer only.<br>• Default system guessed from `navigator.language` but **not stored**.<br>• User can toggle in list modal.                                                                      |
| Clipboard          | • Copies **number + units** (e.g., `2.54 m`).                                                                                                                                                                                |
| Session List       | • Always starts empty on reload.<br>• Delete prompts “Delete this measurement?” (modal) before removal from list **and** scene.                                                                                               |
| Clear-all          | • No bulk clear; users reload page or delete items individually.                                                                                                                       |
| Editing            | • Measurements are read-only once placed.                                                                                                                                                                                    |
| Colouring          | • Each new measurement gets a random high-contrast colour (H ≠ last ± 30 °).<br>• Mid-point label & list dot share colour.                                                                                                   |
| Feedback           | • Haptic tick on each placement; no audio.                                                                                                                                            |
| Tracking Loss      | • Yellow warning banner under top banner: “Move phone to regain tracking.”<br>• UI remains interactive; taps disabled until tracking resumes.                                                                                |
| Unsupported Browser| • Full-screen overlay with message + device suggestions; no fallback mode.                                                                                                             |

---

## 4. Non-Functional Requirements

| Category                | Specification                                                                                                                                                                                 |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Performance**         | Target ≥ 30 fps on mid-range devices with 20+ lines. Use frustum culling & unlit materials.                                                                                                   |
| **Orientation**         | `screen.orientation.lock('portrait')`.                                                                                                                                                        |
| **Accessibility**       | WCAG-AA contrast for banners & modals. No additional colour-blind accommodations in v1.                                                                                                       |
| **Privacy**             | No analytics, cookies, or external calls (except CDN modules). Camera feed stays on-device.                                                                                                   |
| **Security**            | Must run over HTTPS; service-worker restricted to origin scope.                                                                                                                               |
| **Supported Platforms** | Any browser that returns `navigator.xr.isSessionSupported('immersive-ar') == true` (e.g., Chrome ≥ 94 Android, iOS Safari ≥ 15, Samsung Internet, Vision Pro Safari). Others receive overlay. |

---

## 5. Technical Architecture

### 5.1 Front-end Stack

| Layer          | Choice                                                                                                                                                       | Rationale                           |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------- |
| 3-D / AR       | **Three.js 0.165** ES-modules + WebXR API                                                                                                                    | Familiar ecosystem, matches sample. |
| UI             | **Vanilla JS + Tailwind CSS CDN**                                                                                                                            | Zero build step, quick to load.     |
| PWA            | Minimal `manifest.json` (name, short\_name, icons, start\_url, display = standalone, orientation = portrait, theme\_color #000000).                          |                                     |
| Service-Worker | Hand-written `sw.js` using Cache API: precache **index.html**, `main.js`, `styles.css`, icon set; runtime network-first for CDN modules with cache-fallback. |                                     |

---

## 6. Algorithms & Key Methods

| Concern           | Approach                                                                                                                                                                                                                                 |
|-------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Hit-testing       | `session.requestReferenceSpace('viewer')` → `requestHitTestSource`. Use first result each frame; reticle shows `visible = result.length > 0`.                                                                                           |
| Distance calc     | `distance = p1.distanceTo(p2)` (metres).<br>• Metric: ≥ 1 m → metres (toFixed 2); < 1 m → centimetres (Math.round).<br>• Imperial: totalInches = metres×39.3701; feet = Math.floor(totalInches/12); inchesDec = (totalInches % 12).toFixed(2). |
| Colour generator  | Keep HSL saturation = 80%, lightness = 55%.<br>`nextHue = (prevHue + 137) % 360` ensures contrast.                                                                                                |

---

## 7. Error-Handling Strategy

| Scenario                 | UI Response                                                                                   |
| ------------------------ | --------------------------------------------------------------------------------------------- |
| Camera permission denied | Replace AR canvas with overlay: “Camera access is required. Enable it in browser settings.”   |
| WebXR session init fails | Same overlay with error code.                                                                 |
| Unhandled JS error       | Console .log only (no remote logging).                                                        |
| CDN file missing         | Service-worker returns cached copy; if absent, overlay: “Connection lost—reload when online.” |

---

## 8. Testing Plan

| Layer                  | Tooling & Scope                                                                                                                                                                                                                                          |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Unit Tests**         | **Jest** (Node) for: distance calculation, metric↔imperial conversion, colour generator non-collision, random-hue cycle.                                                                                                                                 |
| **Manual Exploratory** | Checklist for: placement, list copy/delete, unit toggle, haptic feedback, loss-tracking banner, PWA install/open offline. Test on at least: Pixel 8 (Android 15 Chrome), iPhone 14 (iOS 17 Safari), Galaxy S22 (Samsung Internet), Vision Pro simulator. |
| **CI**                 | GitHub Actions: `npm ci` → `npm test`. Failures block merge.                                                                                                                                                                                             |

---

## 9. Build & Deployment

| Stage          | Detail                                                                                                                                                          |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Build**      | None. All scripts/styles loaded via `<script type="module">` / Tailwind CDN. Icons generated once (512 × 512 source → smaller sizes via real-favicongenerator). |
| **Deployment** | Manual: push `main` branch; GitHub Pages serves root `/pocket-tape-ar/`.                                                                                        |
| **Versioning** | Git tags `vMAJOR.MINOR.PATCH`.                                                                                                                                  |
| **License**    | **GPL v3**. Include `LICENSE` file.                                                                                                                             |

---

## 10. Assets to Deliver

1. `index.html`, `main.js`, `ar.js`, `ui.js`, `styles.css`, `manifest.json`, `sw.js`.
2. 512×512 app icon – classic yellow tape-measure on white background (PNG + SVG).
3. README with quick-start, supported devices, and GPL notice.
4. `__tests__/` directory with Jest specs.

---

## 11. Future-Ready Extensions (out of scope v1)

* Multi-language UI via i18next.
* Per-measurement edit mode.
* High-contrast / colour-blind safe palette toggle.
* Advanced calibration workflow.
* Automated end-to-end Playwright suite + device farm.


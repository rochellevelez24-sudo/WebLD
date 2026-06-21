# WebLD Explainer

**Web Laptop & Device API**

This document is a [WICG-style explainer](https://w3ctag.github.io/explainers) for WebLD. It is intended to accompany `README.md` (the API surface reference) with the *why* behind the design — motivating use cases, alternatives considered, and the privacy reasoning a standards body would expect before incubation.

---

## Authors

- WebLD Contributors (see repository `CONTRIBUTORS.md`)

## Participate

- GitHub Issues: `github.com/webld/spec/issues`
- W3C Community Group: *proposed, pre-chartering*

---

## 1. Introduction

Browsers today expose rich access to *virtual* device capabilities — GPU contexts, audio graphs, sensors for orientation and ambient light — but remain almost entirely blind to the **operating envelope** of the physical machine running them: thermal headroom, power state, available storage, signal quality, and basic device ergonomics.

This blindness causes a recurring class of real-world failure: a web app behaves identically on a plugged-in desktop with full airflow and on a thermally-constrained laptop running on battery in a hot room, right up until the OS intervenes — frame rates collapse, fans spin audibly, or the OS throttles the CPU mid-session — with the page having had no signal to adapt ahead of time.

Native applications do not have this problem. They query system frameworks (e.g. OS power and thermal management APIs) and adapt continuously. WebLD proposes a narrow, privacy-preserving equivalent for the web platform: **coarse, abstracted, permission-gated telemetry**, not raw hardware access.

WebLD is explicitly *not* an attempt to give the web parity with native hardware APIs. It is an attempt to give the web just enough situational awareness to degrade gracefully instead of failing abruptly.

---

## 2. Goals

- Provide web applications with **coarse, real-time signals** about thermal pressure, battery state, storage headroom, network signal quality, device chassis/ergonomic posture, and connected peripheral *classes*.
- Make every signal **event-driven** where possible, so applications can react without polling loops that themselves waste power.
- Ensure the API is **permission-gated per namespace**, consistent with the existing Permissions API model used by Geolocation, Notifications, and Camera/Microphone.
- Guarantee the API is **structurally incapable of fingerprinting** — not merely policy-discouraged from it.
- Provide a **user-facing Kill-Switch** that overrides all grants, at the OS/browser level, independent of per-origin permission state.
- Design a surface area small and uniform enough to be implementable consistently across engines.

## 3. Non-Goals

- WebLD does **not** expose precise sensor values (exact temperatures, exact battery percentages, exact coordinates, MAC addresses, or device serials).
- WebLD does **not** replace or extend the existing Geolocation API; `webld.geo` is a distinct, intentionally coarser, locale-hinting mechanism and is not a precision positioning system.
- WebLD does **not** provide control/actuation capabilities (e.g. it cannot throttle the CPU, change power plans, or disable peripherals). It is read-only telemetry; all adaptation logic lives in application code.
- WebLD does **not** aim to support forensic, diagnostic, or enterprise device-management use cases. Those belong to native tooling or enterprise-managed browser extensions, not the open web platform.
- WebLD is not a replacement for the existing (non-standard) Battery Status API; where overlap exists, WebLD's `battery` namespace is intentionally coarser for privacy reasons.

---

## 4. User Research / Problem Validation

Motivating signals for this proposal (qualitative, drawn from common developer-reported pain points rather than a single study):

- WebGL/WebGPU-heavy applications (CAD tools, 3D configurators, browser games) have no standard way to detect thermal throttling and pre-emptively reduce render load, leading to visible stutter that users misattribute to "bad code" rather than device conditions.
- Progressive Web Apps with offline-first storage (IndexedDB-heavy apps, local-first editors) have no signal for impending storage exhaustion and frequently fail writes silently or with cryptic quota errors.
- Video conferencing and streaming web apps adapt to *measured* network throughput after degradation has already occurred, rather than having a coarse leading indicator of signal quality.
- Adaptive/responsive layout logic today relies entirely on viewport dimensions and has no notion of physical device posture (e.g., a convertible laptop folded into tablet mode), forcing developers to infer posture indirectly and unreliably.

---

## 5. Use Cases

1. **Creative/CAD tools** reduce shadow quality, anti-aliasing, and draw calls automatically as `temp` transitions from `nominal` → `elevated` → `critical`.
2. **Offline-first editors** pause non-essential local writes and warn users before `storage` reports `low: true`.
3. **Video conferencing apps** pre-emptively downgrade resolution as `signal.band` weakens, rather than reacting to a frozen frame.
4. **Convertible-aware web apps** adjust input affordances (e.g., switching from a mouse-optimized to touch-optimized layout) based on `chassis.postureBand`.
5. **Battery-conscious background apps** (e.g., long-running dashboards, collaborative whiteboards) pause expensive polling or animation loops when `battery.chargeBand` is `critical` and `isCharging` is `false`.
6. **Accessibility-aware peripheral detection** lets an application recognize that an external `input` device class is connected and adapt focus/keyboard-navigation affordances accordingly, without learning *which* device it is.

---

## 6. Proposed API (Summary)

Full type definitions live in `README.md`. Summary surface:

```js
navigator.webld.battery     // EventTarget — chargeBand, isCharging
navigator.webld.chassis     // EventTarget — form, postureBand
navigator.webld.temp        // EventTarget — thermal level
navigator.webld.geo         // Promise-based — coarse region hint
navigator.webld.signal      // EventTarget — connection band/class
navigator.webld.storage     // EventTarget — available band, low flag
navigator.webld.peripheral  // EventTarget — connected device classes
```

Every event-driven namespace follows the same subscription pattern:

```js
const status = await navigator.permissions.request({ name: 'webld-temp' });
if (status.state === 'granted') {
  navigator.webld.temp.addEventListener('change', handleThermalChange);
}
```

This mirrors the existing Permissions API request/observe pattern intentionally, to minimize the learning curve for developers already familiar with Geolocation or Notifications permission flows.

---

## 7. Key Scenarios (Worked Examples)

### Scenario A: Thermal-aware WebGL throttling

A browser-based 3D configurator subscribes to `temp`. On `elevated`, it lowers shadow map resolution; on `critical` or `throttling`, it halves the render pixel ratio and disables post-processing. When the level returns to `nominal`, quality is restored. The user experiences a temporary, intentional quality step-down instead of an uncontrolled frame-rate collapse.

### Scenario B: Storage-safe local-first writes

A note-taking PWA subscribes to `storage`. On `low: true`, it stops buffering new attachments locally, surfaces a non-blocking banner prompting cloud sync or cleanup, and resumes normal behavior once `low` clears.

### Scenario C: Signal-aware adaptive streaming

A video call client subscribes to `signal`. On a transition to the `weak` band, it preemptively renegotiates to a lower-bitrate codec profile *before* packet loss becomes audible/visible, rather than reacting after quality has already degraded.

---

## 8. Detailed Design Discussion

### 8.1 Why abstraction bands instead of raw values?

Raw values (exact temperature in °C, exact battery mAh, exact signal dBm) are individually low-risk but combine into high-entropy fingerprints when read at sufficient frequency and precision — particularly battery drain curves and thermal curves, which are well-documented fingerprinting vectors in prior web platform research. WebLD avoids this entirely by never allowing raw values to leave the implementation layer. This is a stronger guarantee than rate-limiting alone, because it removes the *information*, not just the *frequency* of access.

### 8.2 Why per-namespace permissions instead of one umbrella permission?

A single `webld` permission would create an over-broad grant problem identical to early mobile OS permission models (e.g., one grant covering camera *and* microphone *and* location). Per-namespace permissions let a video app request only `signal` without implicitly gaining access to `chassis` or `peripheral`, keeping the principle of least privilege intact and making permission prompts legible to users.

### 8.3 Why a Kill-Switch independent of per-origin permissions?

Per-origin permissions answer "should *this site* have access" but not "should *any* site have access, right now, regardless of prior consent." The Kill-Switch exists for contexts where users want a hard guarantee — e.g., during screen-sharing, presentations, or sensitive work — without auditing or revoking every individual origin grant. It also serves as a backstop against compromised or malicious origins that obtained consent legitimately but are now behaving adversarially.

### 8.4 Why is `geo` polling-based while other namespaces are event-driven?

Coarse region hints change infrequently relative to thermal/signal/battery state, and a `query()` model avoids holding an open subscription for data that rarely updates — reducing unnecessary background permission indicators in browser chrome.

---

## 9. Considered Alternatives

| Alternative | Why Rejected |
|---|---|
| Extend the existing Battery Status API to cover all namespaces | That API already shipped with known fingerprinting concerns in some engines; WebLD intentionally breaks from it with coarser bands rather than inheriting its precision model. |
| Single umbrella `navigator.webld.permission` grant | Violates least-privilege; produces vague permission prompts ("this site wants device access") that erode user trust. |
| Expose raw sensor values behind a strict rate limiter only | Rate-limiting reduces fingerprinting *speed* but not fingerprinting *feasibility* over a longer session; abstraction is a stronger guarantee than throttling alone. |
| Native-only access via installed PWA capability gating | Excludes the open web entirely; contradicts the goal of universal, capability-based progressive enhancement. |
| Make `geo` full-precision and rely on existing Geolocation permission semantics | Out of scope — duplicates an existing, separately-specified API; WebLD's `geo` is intentionally a distinct, coarser primitive. |

---

## 10. Stakeholder Feedback

*This section is a placeholder pending formal outreach to browser vendors, the W3C Privacy Interest Group (PING), and the W3C TAG. Feedback will be logged here as incubation proceeds.*

- **Browser vendors:** Not yet solicited.
- **W3C PING:** Not yet solicited — review of the abstraction/bucketing model will be a priority request given the fingerprinting-adjacent surface area.
- **W3C TAG:** Not yet solicited.

---

## 11. References & Acknowledgements

- W3C TAG Explainer guidelines: `w3ctag.github.io/explainers`
- W3C Permissions API
- Prior art: Battery Status API, Network Information API, Device Posture API (chassis/posture precedent)
- `README.md` (this repository) — full API and TypeScript reference
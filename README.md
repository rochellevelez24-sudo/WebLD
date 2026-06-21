# WebLD — Web Laptop & Device API

**Status:** Draft Community Group Report
**Version:** 0.1.0-proposal
**License:** Apache License 2.0

---

## 1. Executive Summary

Modern browsers are functionally **hardware-blind**. A web application can render a cinematic WebGL scene, run a multi-threaded WASM workload, or stream a 4K video call — yet it has no idea if the host device is thermally throttling, running on 4% battery, or tethered to a congested Wi-Fi signal. This blindness produces degraded user experiences by default: dropped frames, silent battery drain, and unresponsive UIs, with no recourse for the developer to adapt gracefully.

**WebLD (Web Laptop & Device API)** closes this gap. It is a proposed browser API that exposes a narrow, permission-gated, event-driven telemetry layer between the web sandbox and the physical state of the host device — covering **Thermals, Battery, Ergonomics, Storage, and Signal**.

The mission is not to give the web *access* to hardware. It is to give the web **awareness** of hardware state, so applications can make intelligent, real-time adaptations — throttling render pipelines before thermal shutdown, deferring writes before storage exhaustion, or downgrading stream quality before a signal drop causes a stall.

WebLD is designed from the ground up to be standards-track: minimal surface area, strict abstraction boundaries, and privacy guarantees that are structural rather than policy-based.

---

## 2. Security & Privacy Manifesto

WebLD's design philosophy rests on four non-negotiable pillars. These are architectural constraints, not best-practice suggestions — violating them is treated as a specification bug.

### 2.1 Abstraction
WebLD never exposes raw hardware data. Every value returned by the API is a **normalized, bucketed, or categorical abstraction** of the underlying sensor. Raw register reads, vendor-specific telemetry, and device-unique signal shapes are deliberately discarded at the implementation layer, before they ever reach script context.

| Raw Hardware Signal | WebLD Abstraction |
|---|---|
| CPU package temp in millidegrees | `nominal \| elevated \| critical` |
| Exact battery mAh / cycle count | Percentage band + charging boolean |
| Wi-Fi BSSID / MAC address | Signal strength band only |
| Disk model / serial number | Available storage band (GB, rounded) |

### 2.2 Security-Blindness
WebLD is architecturally incapable of producing a stable, cross-origin, or cross-session identifier. No namespace returns a value — or combination of values — that is unique enough to contribute to **device fingerprinting**. This is enforced via:
- **Quantization**: continuous values are rounded into coarse buckets.
- **Rate-limiting**: telemetry updates are throttled to prevent reconstructing high-resolution signal curves.
- **No persistent identifiers**: hardware IDs, serials, MAC addresses, and chassis identifiers are never serialized to script.

### 2.3 User-Intent
Every namespace is **opt-in and permission-gated** through the standard Permissions API model. No telemetry stream activates without an explicit, namespace-scoped grant. Permissions are:
- Scoped per-namespace (granting `battery` does not imply `signal`).
- Revocable at runtime without a page reload.
- Subject to a visible, persistent indicator in the browser chrome whenever an active subscription is live.

### 2.4 The Kill-Switch
WebLD mandates a hard, OS-level **Kill-Switch**: a single user-accessible control (browser settings + OS privacy panel integration) that immediately and irrevocably terminates *all* active WebLD subscriptions across *all* origins, with no API to detect or circumvent its state. The Kill-Switch is the privacy backstop beneath the permission model — even a malicious or compromised origin with prior consent cannot keep a stream alive once it is engaged.

---

## 3. The API Specification

WebLD is exposed as a single read-only entry point: `navigator.webld`. It is composed of seven independently permissioned namespaces.

| Namespace | Domain | Description | Permission Key | Update Model |
|---|---|---|---|---|
| `battery` | Power | Charge band, charging state, estimated time bracket | `webld-battery` | Event-driven (`change`) |
| `chassis` | Ergonomics | Form factor (laptop/tablet/convertible), lid angle band, posture state | `webld-chassis` | Event-driven (`change`) |
| `temp` | Thermals | Thermal pressure level (`nominal`/`elevated`/`critical`/`throttling`) | `webld-temp` | Event-driven (`change`) |
| `geo` | Coarse Location | Region-level locale hinting (no GPS coordinates) | `webld-geo` | Polled (`query()`) |
| `signal` | Connectivity | Network signal band, connection type class | `webld-signal` | Event-driven (`change`) |
| `storage` | Disk | Available storage band, low-storage boolean | `webld-storage` | Event-driven (`change`) |
| `peripheral` | I/O | Connected peripheral *class* (no device IDs): input, audio, display | `webld-peripheral` | Event-driven (`connect`/`disconnect`) |

> **Note:** `geo` in WebLD is intentionally distinct from the existing Geolocation API. It returns coarse, locale-level hints only (e.g. for regional content/regulatory adaptation) and is not a substitute for precision geolocation.

---

## 4. Code Usage Examples

### 4.1 Throttling a WebGL Render Loop on Thermal Pressure

```js
async function initRenderPipeline() {
  const status = await navigator.permissions.request({ name: 'webld-temp' });
  if (status.state !== 'granted') return startDefaultPipeline();

  navigator.webld.temp.addEventListener('change', (event) => {
    switch (event.level) {
      case 'critical':
      case 'throttling':
        renderer.setPixelRatio(0.75);
        renderer.shadowMap.enabled = false;
        break;
      case 'elevated':
        renderer.setPixelRatio(1.0);
        break;
      case 'nominal':
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.shadowMap.enabled = true;
        break;
    }
  });
}
```

### 4.2 Adapting Stream Quality on Signal Change

```js
async function watchSignalForStreaming() {
  const status = await navigator.permissions.request({ name: 'webld-signal' });
  if (status.state !== 'granted') return;

  navigator.webld.signal.addEventListener('change', (event) => {
    if (event.band === 'weak') {
      player.setQuality('480p');
    } else if (event.band === 'strong') {
      player.setQuality('1080p');
    }
  });
}
```

### 4.3 Deferring Writes on Low Storage

```js
navigator.webld.storage.addEventListener('change', (event) => {
  if (event.low) {
    autosave.pause();
    ui.notify('Local storage is low — autosave paused.');
  } else {
    autosave.resume();
  }
});
```

---

## 5. Implementation Interface (TypeScript)

```ts
// WebLD Type Definitions — extends the global Navigator interface

type ThermalLevel = 'nominal' | 'elevated' | 'critical' | 'throttling';
type SignalBand = 'none' | 'weak' | 'moderate' | 'strong';
type ChassisForm = 'laptop' | 'tablet' | 'convertible' | 'desktop';
type PeripheralClass = 'input' | 'audio' | 'display';

interface WebLDBattery extends EventTarget {
  readonly chargeBand: 'critical' | 'low' | 'medium' | 'high' | 'full';
  readonly isCharging: boolean;
  onchange: ((this: WebLDBattery, ev: Event) => any) | null;
}

interface WebLDChassis extends EventTarget {
  readonly form: ChassisForm;
  readonly postureBand: 'closed' | 'tent' | 'flat' | 'open';
  onchange: ((this: WebLDChassis, ev: Event) => any) | null;
}

interface WebLDTemp extends EventTarget {
  readonly level: ThermalLevel;
  onchange: ((this: WebLDTemp, ev: Event) => any) | null;
}

interface WebLDGeo {
  query(): Promise<{ regionHint: string }>;
}

interface WebLDSignal extends EventTarget {
  readonly band: SignalBand;
  readonly connectionClass: 'wired' | 'wifi' | 'cellular' | 'unknown';
  onchange: ((this: WebLDSignal, ev: Event) => any) | null;
}

interface WebLDStorage extends EventTarget {
  readonly availableBandGB: number;
  readonly low: boolean;
  onchange: ((this: WebLDStorage, ev: Event) => any) | null;
}

interface WebLDPeripheral extends EventTarget {
  readonly connected: PeripheralClass[];
  onconnect: ((this: WebLDPeripheral, ev: Event) => any) | null;
  ondisconnect: ((this: WebLDPeripheral, ev: Event) => any) | null;
}

interface WebLD {
  readonly battery: WebLDBattery;
  readonly chassis: WebLDChassis;
  readonly temp: WebLDTemp;
  readonly geo: WebLDGeo;
  readonly signal: WebLDSignal;
  readonly storage: WebLDStorage;
  readonly peripheral: WebLDPeripheral;
}

interface Navigator {
  readonly webld: WebLD;
}
```

---

## 6. Licensing

This specification, its reference implementation, and all accompanying documentation are released under the **Apache License 2.0**.

```
Copyright [YEAR] WebLD Contributors

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

---

## 7. Contributing & Standards Track

WebLD is published as a draft intended for incubation within a W3C Community Group. Contributions, counter-proposals, and threat-model critiques are welcome via Issues and Pull Requests. All substantive changes to the Security & Privacy Manifesto require consensus discussion — this section is the spec's foundation, not its appendix.

> *WebLD: give the web a pulse on the machine it runs on — without ever letting it read the machine's fingerprints.* 
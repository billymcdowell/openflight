# Plan: Rust Rewrite for Smallest Phone-Monitored Device

> Status: **planning only** — no implementation in this document.
> Audience: OpenFlight maintainers who may be new to Rust.
> Goal: Run the launch monitor on the smallest practical device; phone is the display via browser.

---

## What you are trying to build (plain English)

Today OpenFlight runs on a Raspberry Pi (often Pi 5 + optional touchscreen). The Pi talks to radar hardware, does heavy math (FFT, spin, carry), and serves a React web UI.

You want:

1. **Smaller box** — less power, less cost, less bulk than a Pi 5 + screen.
2. **Phone as the monitor** — open a URL on your phone; see shots live.
3. **Same behavior** — ball speed, club speed, spin, carry, angles (if hardware present).
4. **Proof of sameness** — tests that compare Python vs Rust on the **same inputs → same outputs**.

Rust is a good fit because it compiles to a single binary, uses little RAM, has no garbage-collector pauses, and works well for serial + DSP + a small web server on tiny Linux boards.

You do **not** need to learn all of Rust before starting. You learn it crate-by-crate while porting pure math first (the part that has the best tests already).

---

## Architectural decisions (recommended defaults)

These are opinionated. Change them only if you consciously disagree — they drive every phase below.

| Decision | Recommendation | Why |
|----------|-----------------|-----|
| **Target device (v1)** | **Raspberry Pi Zero 2 W** (or equivalent ~$15–25 Linux SBC with WiFi + USB/UART) | Smallest board that can run the **full** DSP stack + WiFi web UI without rewriting algorithms into fixed-point MCU code. |
| **Not v1** | Bare ESP32 / STM32 as the only brain | Can work later for an OPS-only “micro” build, but FFT/spin/IWR math + WiFi server on an MCU is a second product, not a 1:1 port. |
| **OS** | 64-bit Linux (Raspberry Pi OS Lite) | Keeps serial, WiFi, filesystem logging; Rust targets this easily. |
| **UI** | **Keep the existing React UI** initially | Phone already works today: browse to `http://<device-ip>:8080`. Rewriting UI in Rust (egui, etc.) adds no product value for “phone as monitor.” |
| **Phone connection** | Device WiFi **client** on home network first; optional **AP mode** later (“OpenFlight” SSID) | Client mode is simpler; AP mode is better for a range with no router. |
| **Language split** | Rust = hardware + DSP + HTTP/WebSocket server; TypeScript = UI | Clear boundary; UI tests stay Vitest/Playwright. |
| **Angle hardware** | Port **OPS243 core first**; then **IWR6843** (current); keep **K-LD7** as optional/legacy crate | Matches product direction (K-LD7 deprecated). |
| **Out of v1 scope** | Camera/YOLO, FlightWeb cloud upload, GSPro/sim connectors | Port after core shot path is proven; they are adjacent products. |
| **Python during rewrite** | Keep Python as **oracle** until Rust matches golden vectors | Do not delete Python until parity gates pass. |
| **Float policy** | `f64` for 1:1 parity with NumPy; optimize to `f32` only after golden tests pass | Correctness first; tiny-device speed later. |

### Why not “smallest MCU possible” as the first target?

The production path does roughly:

- 4096 I + 4096 Q samples @ 30 kHz
- Many overlapping FFTs (window 128, pad 4096)
- Envelope / multitaper spin
- Optional IWR6843 dump DSP (much heavier)

That fits comfortably on a Pi Zero 2 W in Rust. On an ESP32-class MCU you would need algorithm simplification, fixed-point math, and a thinner feature set — a **different product**, not a faithful rewrite. Plan a **Phase Micro** only after Linux-on-Zero parity exists.

---

## Mental model for a Rust beginner

Think in **crates** (libraries/packages), not one giant program:

```
openflight/                 # Cargo workspace
├── crates/
│   ├── of-types/           # Shot, ClubType, shared structs (no I/O)
│   ├── of-dsp/             # FFT, speed, spin, carry, ballistics
│   ├── of-ops243/          # Serial protocol to OPS243
│   ├── of-trigger/         # Sound / GPIO / speed triggers
│   ├── of-iwr6843/         # Angle radar (later)
│   ├── of-kld7/            # Legacy angle radar (optional)
│   ├── of-session/         # JSONL session logger
│   └── of-server/          # HTTP + Socket.IO-compatible WS + static UI
├── tests/golden/           # Shared .json / .npy fixtures (Python + Rust)
└── ui/                     # Existing React app (unchanged at first)
```

**How you learn Rust along the way:**

1. Start in `of-types` + `of-dsp` — no serial, no async, just functions + `cargo test`.
2. Add `of-ops243` with a fake serial trait (same idea as Python mocks).
3. Add `of-server` last — async Rust is the steepest learning curve; delay it.

Suggested learning path (only what you need): ownership/borrowing → structs/enums → `Result`/`Option` → traits → `cargo test` → then Tokio/async for the server.

---

## Current system (what must be preserved)

```
SEN-14262 GATE → OPS243 HOST_INT
       → dump 4096 I/Q
       → RollingBufferProcessor (FFT → ball/club/spin)
       → Shot (+ carry)
       → optional IWR6843 / K-LD7 angles
       → Flask-SocketIO emit "shot"
       → React UI on phone/browser
```

**Must-port for phone-monitor MVP:**

1. OPS243 rolling-buffer capture + sound trigger
2. DSP: speed, club, spin, carry tables / spin-adjusted carry
3. Web server serving `ui/dist` + WebSocket `shot` / session events
4. Session JSONL logging (for offline analysis continuity)
5. Mock mode (`simulate_shot`) for UI/dev without hardware

**Defer:** camera, cloud, GSPro/sim, kiosk Chromium, Pi touchscreen paths.

---

## Test strategy: 1:1 Python ↔ Rust comparison

This is the most important part of the rewrite. Do **not** only “rewrite tests in Rust by hand.” That drifts. Use **shared golden vectors**.

### Principle

| Layer | Strategy |
|-------|----------|
| **Pure algorithms** | Same fixture file → Python assert + Rust assert (identical expected outputs within epsilon) |
| **Protocol / serial** | Scripted byte streams (recorded or hand-written) → same parsed structs |
| **Server fusion** | Table-driven cases: Shot + optional angle inputs → same `shot_to_dict` JSON |
| **UI** | Keep existing Vitest + Playwright; point at Rust server |
| **Hardware** | Keep `scripts/hardware-test/*` as manual; optional `#[ignore]` Rust smoke tests |

### Step 0 before any Rust DSP (do this in Python first)

Export goldens from the existing suite so Rust can consume them:

1. **Add a golden exporter** (small Python helper, not a rewrite) that runs key processor paths and writes:
   - `tests/golden/iq/<name>.json` — I/Q arrays + sample_rate + metadata  
   - `tests/golden/expected/<name>.json` — ball_speed, club_speed, spin_rpm, impact_idx, carry, etc.
2. **Prefer deterministic synth** already in `tests/spin_synth.py` and synthetic peaks in `test_rolling_buffer.py`.
3. **For real captures**, check in a **small curated subset** of session I/Q (anonymized, size-limited) that currently live only in gitignored `session_logs/` — otherwise CI cannot guarantee parity.
4. **Document float tolerance:** e.g. speeds ±0.01 mph, spin ±1 RPM, or relative 1e-9 for intermediate spectra where applicable.

### Mapping: existing Python tests → Rust suites

Priority order for parity (highest value first):

| Priority | Python source | Rust crate | Kind |
|----------|---------------|------------|------|
| P0 | `test_rolling_buffer.py` (~143) | `of-dsp` | Pure / golden |
| P0 | `test_multitaper_spin.py`, `test_spin_*.py` | `of-dsp` | Pure / golden |
| P0 | `test_launch_monitor.py`, `test_ballistics.py`, `test_speed_correction.py`, `test_spin_estimate.py` | `of-types` / `of-dsp` | Pure |
| P1 | `test_ops243*.py`, UART/baud/rearm/deadlock | `of-ops243` | Fake serial |
| P1 | Trigger tests inside `test_rolling_buffer.py` + sound-trigger deadlock | `of-trigger` | Mocked |
| P1 | `test_session_logger.py` | `of-session` | Temp dir I/O |
| P2 | IWR: `test_iwr6843_pipeline.py` + related pure tests | `of-iwr6843` | Pure / golden |
| P2 | K-LD7: `test_kld7_radc_lib.py`, geometry, two-ray | `of-kld7` | Pure (legacy) |
| P2 | `test_server.py` shot fusion / `shot_to_dict` / angle gates | `of-server` | Integration |
| P3 | Cloud / GSPro / sim / camera / diagnose scripts | later crates | Optional |
| Keep | `ui/**/*.test.*`, Playwright e2e | UI | Point at Rust `:8080` |

### Parity gate (definition of “ported”)

A module is done only when:

- [ ] Every P0/P1 Python test case for that module has a Rust counterpart **or** an explicit waiver listed in this plan’s waiver table.
- [ ] Golden fixtures that exist are asserted in **both** languages in CI.
- [ ] A CI job runs `uv run pytest` (oracle) **and** `cargo test` (port) on the same goldens.

### Known gaps to close *before* claiming 1:1 (still in Python)

These are weak/untested today; fix or waive explicitly:

| Gap | Action for rewrite honesty |
|-----|----------------------------|
| Live HOST_INT / persist rolling buffer | Keep as hardware scripts; not CI parity |
| Real OPS243 E2E | Record one golden dump byte stream for fake-serial tests |
| Sparse in-repo I/Q corpora | Check in curated goldens |
| No property-based tests | Optional later (`proptest` / Hypothesis); not required for 1:1 |
| UI stores / socketService lightly tested | Keep UI as-is; e2e against Rust server |

---

## Device sizing reality check

| Device | Fits full OpenFlight? | Notes |
|--------|----------------------|-------|
| **Pi 5** (today) | Yes | Overkill once UI is phone-only |
| **Pi Zero 2 W** | Yes (recommended v1) | ~512 MB–1 GB RAM; Rust binary + static UI is fine; avoid Chromium kiosk |
| Pi Zero W (1st gen) | Risky | Single-core / slower; possible for OPS-only, tight for IWR |
| ESP32-S3 alone | OPS-only “micro” later | Need fixed-point DSP + thinner spin; separate plan |
| “Smallest possible” MCU + external WiFi | Custom product | Not a line-by-line port |

**Phone as monitor on Zero 2 W:**

1. Device joins WiFi (or hosts AP).
2. Rust serves `ui/dist` on `0.0.0.0:8080`.
3. Phone opens `http://openflight.local:8080` or the IP.
4. No touchscreen, no kiosk script required for this product shape.

Power/size win vs Pi 5 + 7" display is large even before any MCU fantasy.

---

## Phased plan (tracer-bullet vertical slices)

Each phase is demoable. Do not start phase N+1 until phase N’s acceptance criteria pass.

### Phase 0 — Golden harness (Python only)

**User story:** “I can prove algorithm outputs with files that Rust will later load.”

**What to build:** Exporter + initial golden set from rolling-buffer + spin + carry + ballistics cases; CI job that validates goldens still match Python.

**Acceptance criteria:**

- [ ] ≥ 20 goldens covering ball speed, club speed, spin accept/reject rails, carry table, ballistic carry
- [ ] Documented schema for fixture JSON
- [ ] Pytest reads goldens (not only regenerates them)

---

### Phase 1 — Rust DSP crate + parity

**User story:** “On my laptop, Rust computes the same shot metrics as Python from the same I/Q.”

**What to build:** `of-types` + `of-dsp`; port processor / multitaper / carry / ballistics / speed correction / kinematic spin; `cargo test` against goldens.

**Acceptance criteria:**

- [ ] All P0 goldens pass in Rust within tolerance
- [ ] No serial/network code yet
- [ ] README section: “How to run Python vs Rust parity”

**Beginner note:** This phase teaches Rust with the smallest surface area and the strongest existing tests (`test_rolling_buffer.py`).

---

### Phase 2 — OPS243 + trigger + mock capture path

**User story:** “Rust can parse a recorded dump and fake a sound trigger the way unit tests do today.”

**What to build:** `of-ops243` + `of-trigger` with trait-based serial/GPIO; port fake-serial tests from `test_ops243*.py` and trigger tests.

**Acceptance criteria:**

- [ ] Baud negotiation / dump parse / rearm / write-timeout behaviors covered
- [ ] `SoundTrigger` accept/reject + timestamp propagation parity
- [ ] Integration test: scripted dump → `ProcessedCapture` → `Shot`

---

### Phase 3 — Thin server + phone UI

**User story:** “I open my phone to the device IP and see mock shots; then real shots on a Pi Zero 2 W.”

**What to build:** `of-server` serving static `ui/dist`; WebSocket (Socket.IO protocol compatibility **or** a thin adapter — decide below); `simulate_shot`; session logger; mock monitor.

**Critical sub-decision (choose before coding):**

| Option | Pros | Cons |
|--------|------|------|
| **A. Speak Socket.IO** (recommended) | Existing UI unchanged | Need Rust Socket.IO server crate or small bridge |
| **B. Plain WebSocket + change UI** | Simpler Rust | Touches every `socketService` event |
| **C. Keep tiny Python Socket.IO shim calling Rust DSP via FFI/RPC** | Fastest phone demo | Not a real rewrite; temporary only |

**Recommendation:** **A** for the end state; optional short **C** spike only if Socket.IO in Rust blocks learning.

**Acceptance criteria:**

- [ ] Phone on same WiFi shows live UI
- [ ] `shot` payload field-compatible with current `shot_to_dict`
- [ ] Playwright e2e pass against Rust mock server
- [ ] Session JSONL written with same entry types for core events

---

### Phase 4 — Hardware bring-up on Pi Zero 2 W

**User story:** “OPS243 + sound trigger on the smallest Linux board; phone is the only display.”

**What to build:** Deployment docs (Lite OS, UART/USB, WiFi, systemd service); replace `start-kiosk.sh` Chromium path with `openflight-rust` service; hardware-test checklist.

**Acceptance criteria:**

- [ ] Cold boot → WiFi → phone connects → swing → shot on phone
- [ ] Power/size notes documented vs Pi 5 kiosk
- [ ] Rolling-buffer persist setup still documented (firmware bug unchanged)

---

### Phase 5 — Angle stack (IWR6843 first)

**User story:** “Launch angle / aim / club path match Python on goldens, then on device.”

**What to build:** `of-iwr6843` pure pipeline + monitor wiring into server fusion tests (`test_server.py` angle gates).

**Acceptance criteria:**

- [ ] IWR P2 pure tests / goldens pass
- [ ] Server fusion parity for vertical/horizontal + carry adjustments
- [ ] K-LD7 only if you still ship those builds (`of-kld7`)

---

### Phase 6 — Optional product surfaces

Cloud upload, GSPro/sim connectors, AP-mode WiFi wizard, mDNS (`openflight.local`).

Each gets its own mini-plan; not required for “phone monitor on tiny device.”

---

### Phase Micro (future, separate product)

ESP32 (or similar) OPS-only build: fixed-point FFT, reduced spin, BLE or WiFi to phone. **Only after** Phases 0–4 prove algorithm ownership. Do not mix into the 1:1 rewrite.

---

## Suggested Cargo / tooling choices (when you do start)

| Need | Crate direction | Avoid at first |
|------|-----------------|----------------|
| FFT | `rustfft` | Writing your own FFT |
| Linear algebra / vectors | `ndarray` (NumPy-like) | Premature `nalgebra` for everything |
| Serial | `serialport` | |
| HTTP static files | `axum` or `actix-web` | |
| Async runtime | `tokio` | Learning async before Phase 3 |
| JSON | `serde` / `serde_json` | |
| Tests | `cargo test` + shared `tests/golden` | Duplicating numbers in source |

Python stays on `uv` / pytest as the oracle until parity gates pass.

---

## What “done” looks like for your stated goal

1. A Pi Zero 2 W (or similar) runs a single Rust binary + `ui/dist`.
2. User’s phone is the monitor (browser).
3. OPS243 + sound trigger produce shots with metrics matching Python goldens.
4. CI runs Python golden checks and Rust golden checks on the same fixtures.
5. Optional IWR6843 angles on the same device class.
6. No requirement that the implementer knew Rust at the start — only that each phase is small enough to learn inside.

---

## Waiver table (fill as you go)

| Python test / area | Waive? | Reason |
|--------------------|--------|--------|
| Camera / YOLO | Yes for v1 | Disabled in prod |
| Chromium kiosk scripts | Yes | Phone replaces display |
| Missing gitignored session_logs cases | Until goldens checked in | Cannot CI |
| Hardware-only scripts | Manual only | No device in CI |

---

## Open choices to confirm before implementation

1. **Device:** Agree Pi Zero 2 W as v1 target? (vs stay on Pi 5 hardware but Rust+phone-only UX)
2. **Socket.IO:** Keep protocol (A) vs change UI (B)?
3. **Angle radar in v1:** OPS-only first, or IWR in the first device image?
4. **K-LD7:** Port for existing customers, or document “Python-only legacy”?
5. **Golden check-in:** OK to commit curated I/Q fixtures to the repo?

---

## Immediate next steps (still no product Rust code)

1. Confirm the five open choices above.
2. Implement **Phase 0** golden exporter + fixture schema (Python-only).
3. Scaffold empty Cargo workspace with `of-types` / `of-dsp` and one failing golden test (first Rust code).
4. Proceed Phase 1 until P0 parity is green.

Until those choices are confirmed, do not start the rewrite proper.

# Plan: Rust on ESP32 — Phone as the Monitor

> Status: **planning only** — no implementation in this document.
> Audience: OpenFlight maintainers who may be new to Rust (and new to embedded).
> Goal: Run the launch-monitor brain on an **ESP32**, connect from a phone over WiFi, see shots live.
> Prior decision update: **ESP32 is the target** (not Pi Zero). That is cooler and smaller — and it forces a deliberate, thinner product than a full Pi port.

---

## The honest pitch

Yes — we can target an ESP32, and it will feel magical: a matchbox-sized board, golf radar plugged in, phone is the screen.

But this is **not** “copy the whole Pi app onto a microcontroller.” It is:

> **Same OPS243 shot math (proven by golden tests) + WiFi + phone UI, on ESP32-S3.**  
> Angle radars (IWR6843 / K-LD7), camera, cloud, GSPro, and the full React kiosk UI stay off this device (at least for v1).

If we pretend the full Python stack fits unchanged, the project fails. If we scope it as an **ESP32 edition** with shared algorithm goldens, it is achievable and genuinely cool.

---

## Recommended hardware (be specific)

| Choice | Recommendation | Why |
|--------|----------------|-----|
| **Chip** | **ESP32-S3** (not original ESP32, not C3) | Dual core, USB, enough CPU for FFTs, modern `esp-rs` support |
| **Module** | **S3 with PSRAM + ≥8–16 MB flash** (e.g. N16R8 class: 16 MB flash / 8 MB PSRAM) | 4096 I/Q + FFT workspaces need RAM; UI assets need flash |
| **Board** | Any S3 DevKit with PSRAM broken out + spare UART | Easy bring-up; later a custom PCB |
| **Radar link** | OPS243 on **UART** (not USB-host) | ESP32 is not a comfortable USB host; OPS243 already supports UART |
| **Trigger** | Keep **SEN-14262 GATE → OPS243 HOST_INT** (hardware trigger on radar); ESP32 only receives the dump over UART | Lowest latency; avoids bit-banging HOST_INT from the MCU unless needed |
| **Fallback trigger** | GATE → ESP32 GPIO → software `S!` dump | Same as today’s `sound-gpio` path |
| **Phone link** | ESP32 **WiFi soft-AP** named e.g. `OpenFlight` → phone joins → open `http://192.168.4.1/` | No home router required at the range |

**Power:** 5 V for OPS243 + 3.3 V for ESP32/sound board; shared ground. Document a single USB-C PD or barrel supply in bring-up docs (later).

---

## What fits on ESP32 vs what does not

### In scope (ESP32 v1 — “cool path”)

1. OPS243 rolling-buffer dump over UART  
2. Sound-triggered capture (hardware HOST_INT on radar, or GPIO fallback)  
3. DSP: ball speed, club speed, impact estimate, spin (start with envelope; multitaper if it fits timing/RAM)  
4. Carry from existing tables + spin-adjusted carry  
5. Tiny HTTP server + **plain WebSocket**  
6. **Slim phone UI** (not the full current React app as-is)  
7. Mock / simulate-shot for UI without radar  
8. Optional: small ring of recent shots in RAM / little flash log (not full Pi JSONL sessions)

### Explicitly out of ESP32 v1

| Feature | Why out |
|---------|---------|
| **IWR6843** | ~1 Mbaud binary dumps + heavy DOA/LCMF — wrong class of MCU workload |
| **K-LD7** | Up to 3 Mbaud streaming + dual radar — needs a Linux USB host |
| Full **React `ui/`** as today | Bundle + Socket.IO + Zustand app is Pi-sized; trim or replace |
| **Socket.IO** | Too heavy / awkward on MCU; use plain WebSocket |
| Camera / YOLO | Impossible here |
| Cloud / GSPro / sim connectors | Network + protocol bulk; later companion or stay on Pi edition |
| Full session JSONL offline science pipeline | Use phone download of last-N shots JSON instead |

### Product framing (important)

Treat two editions:

| Edition | Hardware | Role |
|---------|----------|------|
| **OpenFlight ESP** (this plan) | ESP32-S3 + OPS243 + sound trigger + phone | Portable speed/spin/carry monitor |
| **OpenFlight Pi** (existing) | Pi + optional IWR/K-LD7 + full UI + sims/cloud | Full lab / sim / angle product |

Shared **DSP goldens** keep the ESP edition honest against Python. The Pi edition can keep evolving separately until/unless you later port more.

---

## Architectural decisions (ESP32 edition)

| Decision | Recommendation | Why |
|----------|----------------|-----|
| **Target** | ESP32-S3 + PSRAM | Cool + smallest practical for this DSP |
| **Rust style** | **Host-tested `of-dsp` library** + **ESP firmware crate** | You learn/test math on a laptop first; flashing is last |
| **ESP framework (v1)** | **`esp-idf` + Rust (`std`)** via esp-rs | WiFi + HTTP are far easier than pure `no_std` Embassy for a beginner |
| **Later optional** | Embassy `no_std` rewrite | Only if you outgrow IDF or want tighter control |
| **Float policy** | Develop goldens in **`f64` on host**; run **`f32` on device** with documented tolerances | S3 has FP assist; `f64` everywhere blows RAM/time |
| **UI** | New **slim mobile web UI** (one screen: last shot + short history + club picker) | Fits flash; loads fast on phone over ESP AP |
| **Phone protocol** | REST for config + **WebSocket** for `shot` events | Simple; easy to test |
| **AP vs station** | Soft-AP first; station mode later | Range use without a router |
| **Python** | Remains **oracle** forever for goldens; Pi app can keep shipping | ESP does not replace every feature on day one |

---

## Mental model for a Rust beginner (ESP-shaped)

You will **not** start by fighting the ESP toolchain. Order matters:

```
1) Laptop: of-dsp + golden tests          ← learn Rust here
2) Laptop: of-ops243 with fake UART       ← still no hardware
3) ESP: blink + UART echo to OPS243       ← first flash
4) ESP: dump → DSP → print speeds         ← serial monitor proof
5) ESP: WiFi AP + WebSocket + slim UI     ← phone moment
```

### Crate layout

```
openflight/
├── crates/
│   ├── of-types/        # Shot, ClubType (no_std-friendly if possible)
│   ├── of-dsp/          # FFT / speed / spin / carry — runs on host AND esp
│   ├── of-ops243/       # Protocol parsing + command sequences
│   ├── of-proto/        # WebSocket JSON schema (shot messages)
│   └── of-esp/          # Firmware only: WiFi, HTTP, UART drivers, main
├── tests/golden/        # Shared fixtures (Python + host Rust)
├── ui-esp/              # Slim phone UI (static files baked into firmware)
└── ui/                  # Existing full React app — Pi edition (unchanged)
```

**Learning path:** ownership → structs/enums → `Result` → `cargo test` → then ESP flash tools. Avoid async/Embassy until the phone UI phase.

---

## Data flow (ESP edition)

```
SEN-14262 GATE ──► OPS243 HOST_INT
                      │
                      ▼
              UART dump (4096 I + 4096 Q) ──► ESP32-S3
                      │
                      ▼
              of-dsp (f32): ball / club / spin / carry
                      │
                      ▼
              WebSocket "shot" JSON
                      │
                      ▼
              Phone browser on OpenFlight WiFi AP
```

---

## Test strategy: 1:1 where it matters

Same golden-vector idea as before — with ESP reality baked in.

### Principle

| Layer | Where it runs | Parity target |
|-------|---------------|---------------|
| DSP algorithms | **Host** `cargo test` + **Python pytest** on same fixtures | Strict 1:1 (f64 host); device uses f32 tolerance band |
| OPS243 protocol | Host tests with scripted bytes | 1:1 with Python fake-serial tests |
| Firmware smoke | ESP device / QEMU if available | “Dump in → shot JSON out” few cases |
| Slim UI | Vitest/Playwright against a **host mock WS server** | Behavioral, not pixel-perfect vs full UI |

### Phase 0 goldens (still do first, in Python)

Export from existing suites:

- P0: `test_rolling_buffer.py`, spin synth, multitaper (mark multitaper optional on-device), carry, ballistics subset  
- P1: OPS243 dump parse fixtures  
- **Waive for ESP:** IWR/K-LD7/server Flask fusion/cloud/sim/camera tests — not part of ESP edition parity

### Float / tolerance policy

| Metric | Host f64 vs Python | Device f32 vs golden |
|--------|--------------------|----------------------|
| Ball / club speed (mph) | ±0.01 or tighter | ±0.05 (tune after measurement) |
| Spin (rpm) | ±1 | ±25 or relative band |
| Carry (yd) | ±0.1 | ±1 |

Document every loosened tolerance in the waiver table — that is still “engineered enough,” not hand-wavy.

### Definition of “ported” for ESP

- [ ] Every **in-scope** P0 algorithm has host Rust tests on shared goldens  
- [ ] ESP firmware uses the **same `of-dsp` code path** (not a second rewrite)  
- [ ] At least one on-device (or hardware-in-loop) test: recorded dump bytes → shot fields within f32 band  
- [ ] Out-of-scope Python tests listed as **waived for ESP edition**, not silently ignored  

---

## DSP constraints on the S3 (so the cool demo actually works)

Rough budget to design against:

| Resource | Constraint | Mitigation |
|----------|------------|------------|
| RAM | Internal SRAM is tight | Put I/Q + FFT buffers in **PSRAM**; keep hot loops aware of speed |
| Time | Shot UX can tolerate ~50–200 ms process | Overlapping FFTs OK; profile before adding multitaper on-device |
| Flash | UI + firmware share flash | Slim UI; compress assets; no source maps |
| CPU | Dual core | UART/WiFi on one pattern; DSP on the other (careful locking) |

**Algorithm staging on device:**

1. **Must:** standard + overlapping FFT speed path, club speed, impact, carry tables  
2. **Should:** envelope spin with existing validation rails  
3. **Could:** multitaper spin (port for host parity first; enable on ESP only if timing OK)  
4. **Won’t (v1):** full ballistics RK4 every shot if table carry is enough for phone UI (optional later)

---

## Phased plan (tracer bullets)

### Phase 0 — Golden harness (Python)

**Demo:** fixtures on disk; pytest reads them.

- [ ] ≥ 20 goldens for speed / club / spin rails / carry  
- [ ] JSON schema documented  
- [ ] Waiver list for ESP-out-of-scope tests committed  

---

### Phase 1 — Host Rust DSP parity

**Demo:** on a laptop, `cargo test` matches Python goldens.

- [ ] `of-types` + `of-dsp`  
- [ ] P0 goldens green in f64  
- [ ] Feature flag or separate path preparing f32  

**This is where you learn Rust.** No ESP yet.

---

### Phase 2 — OPS243 protocol on host

**Demo:** scripted UART bytes → `Shot`.

- [ ] Port dump JSON parse, command sequences, rearm behaviors relevant to UART mode  
- [ ] Trigger accept/reject logic as pure functions  

---

### Phase 3 — ESP bring-up (no phone yet)

**Demo:** serial monitor prints ball/club/spin after a real or injected dump.

- [ ] Toolchain: esp-rs + flash + monitor documented for beginners  
- [ ] UART to OPS243 (or inject fixture dump over serial for lab without radar)  
- [ ] Call `of-dsp` on device; confirm f32 band vs golden  

---

### Phase 4 — WiFi AP + WebSocket + slim phone UI

**Demo:** join `OpenFlight`, open the page, see a live shot.

- [ ] Soft-AP + static file server from flash  
- [ ] WebSocket `shot` message (`of-proto`)  
- [ ] Slim UI: last shot metrics, history list, club select, simulate button  
- [ ] Host mock server so UI tests run in CI without hardware  

---

### Phase 5 — Sound trigger hardening + packaging

**Demo:** hit a ball (or clap/trigger) → phone updates hands-free.

- [ ] HOST_INT path verified; GPIO fallback documented  
- [ ] Rolling-buffer persist setup still required on OPS243 (firmware bug unchanged)  
- [ ] Power/wiring one-pager  
- [ ] Enclosure note (optional CAD later)  

---

### Phase 6 — Stretch (only after the phone moment)

- Multitaper on-device if profiled OK  
- Station WiFi mode + mDNS  
- Phone download of session JSON  
- Custom PCB  
- **Not on ESP:** IWR/K-LD7 — keep on Pi edition  

---

## Beginner toolchain (when you eventually code)

You will use roughly:

1. **Rustup** + `cargo` on your laptop  
2. **espup** / esp-rs install for S3  
3. `cargo test` for goldens (daily driver)  
4. `cargo espflash` + serial monitor for device  

Do **not** start with Embassy + `no_std` + async executors on day one. Get host goldens green first; that success keeps motivation through flashing pain.

---

## Risks (and how we de-risk)

| Risk | Mitigation |
|------|------------|
| “Full UI won’t fit / be slow” | Slim `ui-esp` from day of Phase 4; do not bake full `ui/dist` |
| PSRAM latency makes DSP late | Profile early in Phase 3; shrink overlap count if needed |
| Beginner + embedded = stall | Host Phases 0–2 deliverable without a board |
| Feature envy (angles, sims) | Written waivers; Pi edition remains the full product |
| f32 drift vs TrackMan expectations | Publish tolerance bands; keep Python oracle |

---

## Waiver table (ESP edition)

| Area | Waive on ESP? | Notes |
|------|---------------|-------|
| IWR6843 / K-LD7 | Yes | Pi edition |
| Full React UI / Socket.IO | Yes | Replaced by slim UI + WS |
| Camera / cloud / GSPro / sim | Yes | Pi edition |
| Flask `test_server.py` fusion | Mostly | Replaced by `of-proto` + slim fusion |
| Multitaper on-device | Soft waive | Required on host; optional on chip |
| Full JSONL session science | Yes | Last-N shots / phone export instead |
| Hardware-only scripts | Manual | Same as today |

---

## Open choices left (narrower now)

Device choice is settled: **ESP32-S3 + PSRAM**.

Please confirm:

1. **Board class:** OK to standardize on **S3 with PSRAM + ≥8 MB flash** (N16R8-class)?  
2. **UI:** Agree to a **new slim phone UI** (not shipping the full current React app on the ESP)?  
3. **Angles:** Agree ESP v1 is **OPS-only** (speed/spin/carry), angles stay on Pi?  
4. **Spin depth:** Envelope spin required on device; multitaper host-only until proven?  
5. **Framework:** OK starting with **esp-idf + Rust std** (easier WiFi) rather than Embassy no_std?

---

## Immediate next steps (still no product firmware)

1. Confirm the five choices above.  
2. Phase 0: golden exporter + ESP waiver list.  
3. Phase 1: host `of-dsp` + failing-then-passing golden tests.  
4. Buy/order an **ESP32-S3 DevKit with PSRAM** so Phase 3 is unblocked when you get there.

Until those are confirmed, do not start firmware — but the direction is now **ESP-first**, and the plan is built around making that cool demo real without lying about scope.

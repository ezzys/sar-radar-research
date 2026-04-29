# DIY SAR Radar — Cycle 1 Design Synthesis

**Date:** April 30, 2026  
**Project:** DIY Synthetic Aperture Radar  
**Budget Target:** Under $200 total system cost  
**Status:** Phase 1 — Research Complete, Design Complete

---

## Executive Summary

This document synthesizes findings from four research areas (antennas, RF front-end, processor, signal processing) into a practical phased build plan for a DIY X-band FMCW Synthetic Aperture Radar.

**Architecture Decision:** Heterodyne X-band FMCW SAR with STM32F407 as data acquisition controller, Raspberry Pi 4B for image formation, and PC-assist for heavy processing.

**Total Phase 1 Build Cost:** ~$265 (excluding GPS RTK for initial testing)

---

## 1. System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DIY SAR SYSTEM ARCHITECTURE                        │
└─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
  │   RF Frontend   │      │  STM32F407       │      │  Raspberry Pi   │
  │   (X-band)      │─────▶│  (Acquisition   │─────▶│  4B (Image      │
  │                 │      │   Controller)    │      │   Formation)    │
  └─────────────────┘      └──────────────────┘      └────────┬────────┘
                                                               │
  ┌─────────────────┐                                         │
  │   GPS/IMU       │                                         ▼
  │   (Motion       │                               ┌─────────────────┐
  │    Comp)        │──────────────────────────────▶│   PC (Heavy     │
  └─────────────────┘                               │    Processing)  │
                                                    └─────────────────┘
```

**Key Design Decisions:**
- **Band:** X-band (9–10 GHz) — best balance of size, resolution, component availability
- **Modulation:** FMCW — lower peak power, simpler hardware, suitable for <1 km range
- **Architecture:** Heterodyne with IF sampling — ADC operates at lower IF, not direct RF
- **Processing:** Hybrid — STM32 handles real-time acquisition, SBC/PC handles image formation

---

## 2. Antenna Subsystem

### 2.1 Frequency Band: X-Band (9–10 GHz)

| Parameter | Value | Notes |
|-----------|-------|-------|
| Frequency | 9–10 GHz | Not ISM; 9.6 GHz common for radar modules |
| Wavelength | 3.75 cm | Allows good resolution at moderate antenna sizes |
| Atmospheric Loss | ~0.02 dB/km | Minimal in clear air |
| PCB Tolerance | ±2–3 mil | Achievable with standard PCB fabrication |

**Resolution Potential:**
- Range Resolution: `c / (2 × B)` — at 100 MHz bandwidth → 1.5 m
- Azimuth Resolution: `D / 2` — a 30 cm antenna gives 15 cm azimuth resolution

### 2.2 Antenna Type: Microstrip Patch Array (4×4)

**Recommended for drone-mounted operation — low profile, light weight, reproducible.**

| Specification | Value |
|--------------|-------|
| Center Frequency | 9.6 GHz |
| Number of Elements | 16 (4×4) |
| Substrate | Rogers RT/duroid 5880 (εr = 2.2), 31 mil |
| Element Spacing | 0.7λ ≈ 22 mm |
| Array Aperture | ~90 mm × 90 mm |
| Gain | 18–22 dBi |
| Beamwidth (E/H-plane) | ~25° |
| Bandwidth (S11 < -10 dB) | ~400 MHz (4%) |
| Weight | 80–120g |
| Estimated Cost | $60–120 (PCB + components) |

**Buildability Notes:**
- JLCPCB/PCBWay: $5–20 for 5 boards
- Rogers板材 expensive ($50–100 for 6"×6"); FR-4 usable but higher loss
- Patch alignment critical: ±0.5 mm tolerance needed at 10 GHz

### 2.3 Alternative Antenna Options

| Antenna Type | Gain | Pros | Cons | Cost |
|-------------|------|------|------|------|
| **Waveguide (WR-90)** | 10–15 dBi | Very low loss, wide bandwidth, easy to build | Bulkier, fixed beamwidth | $20–55 |
| **Helical** | 12–15 dBi | Circularly polarized, simple construction | Moderate gain, wind loading | $25–45 |
| **Parabolic Dish (18")** | 30+ dBi | Highest gain | Heavy, poor for drone | $45–110 |

### 2.4 Drone Payload Constraints

| Drone Class | Payload | Recommended Antenna | Weight |
|-------------|---------|---------------------|--------|
| Mavic-class (~900g) | 100–200g | 4×4 patch array | 80–120g |
| Matrice-class (~4 kg) | 800–1500g | 8×8 patch or small horn | 300–500g |
| Heavy-lift (>10 kg) | 3–8 kg | Parabolic 18–24" | 800–2000g |

**Recommended Parts List (Antenna):**

| Part | Description | Cost |
|------|-------------|------|
| Rogers RT/duroid 5880 (12"×12") | 0.031" thick, for 2–3 arrays | $80–120 |
| or FR-4 prototype | Budget alternative | $15–25 |
| SMA connectors (4×) | SMD edge launch | $8–12 |
| Wilkinson divider network | Custom PCB | Included above |
| **Total** | | **$60–130** |

---

## 3. RF Front-End Subsystem

### 3.1 Architecture: Heterodyne with IF Sampling

**Why Heterodyne:**
- Direct sampling at X-band (10 GHz) requires >20 GSPS ADC — impractical under $200
- Heterodyne downconverts to IF (e.g., 100–500 MHz) where affordable ADCs exist
- Enables use of 10–12 bit ADCs with adequate dynamic range

### 3.2 Key Components

#### Low Noise Amplifier (LNA)

**Critical — determines receiver sensitivity. Do not cheap out.**

| Part | NF | Gain | Freq | Cost | Notes |
|------|-----|------|------|------|-------|
| **Analog Devices HMC1041** | 2.5 dB | 20 dB | 2–30 GHz | ~$25 | True X-band, recommended |
| Mini-Circuits GVA-83+ | 2.5 dB | 22 dB | 2–8 GHz | ~$15 | Needs bypass for X-band |
| Qorvo TQP3M9008 | 1.0 dB | 22 dB | 0.5–6 GHz | ~$12 | Not X-band capable |

#### Mixer

| Part | Type | Cost | Notes |
|------|------|------|-------|
| **Mini-Circuits ADE-30+** | RMS diode | ~$8 | DC-3 GHz IF, good for budget |
| Analog Devices ADL5811 | Active | ~$15 | 10–80 GHz, excellent but costly |

#### Frequency Synthesizer / LO Chain

For X-band LO, a base synthesizer is multiplied:

| Part | Max Freq | Phase Noise | Cost | Notes |
|------|----------|-------------|------|-------|
| **Analog Devices ADF4351** | 4.4 GHz | -220 dBc/Hz | ~$12 | Needs ×2 multiplier for X-band |
| TI LMX2572 | 6.4 GHz | -228 dBc/Hz | ~$15 | Needs ×2 multiplier |
| Silicon Labs SI5351 | 200 MHz | -140 dBc/Hz | ~$3 | IF generation only, NOT for LO |

**LO Chain for X-band:**
1. ADF4351 → 4.8 GHz output
2. ×2 frequency multiplier (MAR-6 or similar) → 9.6 GHz LO
3. Bandpass filter to remove harmonics

#### Filters

| Type | Application | Cost |
|------|-------------|------|
| Cavity filter (X-band) | RF input filter | $50–100 (new), $15–30 (surplus) |
| Microstrip hairpin | IF filtering | $5–15 (DIY PCB) |
| SAW filter | IF stage | $10–30 |

### 3.3 ADC Requirements

**STM32F407 CANNOT directly sample IF for SAR:**
- 2.4 MSPS max sample rate — insufficient for 300+ MHz IF
- ~1 MHz analog bandwidth — too narrow

**Recommended External ADC:**

| ADC Board | Specs | Cost | Interface |
|-----------|-------|------|-----------|
| **AD9200** | 20 MSPS, 10-bit | ~$30 | Parallel |
| ADS54J20 | 80 MSPS, 14-bit | ~$80 | LVDS/JESD204 |
| AD9467 | 105 MSPS, 16-bit | ~$150 | LVDS |

**Recommended Budget ADC:** AD9200 (20 MSPS, 10-bit, ~$30)

### 3.4 SDR Platform Alternative

For rapid prototyping, consider SDR platforms:

| Platform | Freq Range | Bandwidth | ADC/DAC | Cost | X-band Ready? |
|----------|-------------|-----------|---------|------|---------------|
| **PlutoSDR (ADALM-PLUTO)** | 325 MHz–3.8 GHz (RX) | 20 MHz | 12-bit, 61 MSPS | ~$150 | No (needs upconversion) |
| RTL-SDR | 500 kHz–1.7 GHz | 2.4 MHz | 8-bit | ~$20 | No |
| HackRF One | 1 MHz–6 GHz | 20 MHz | 8-bit | ~$300 | No (needs downconversion) |

**Recommendation:** Use PlutoSDR for initial experiments at L-band (1–2 GHz), migrate to custom X-band front-end for final build.

### 3.5 Front-End Parts List

| Part | Description | Cost |
|------|-------------|------|
| **ADF4351** | PLL synthesizer | $12 |
| **HMC1041** | X-band LNA | $25 |
| **ADE-30+** | Mixer | $8 |
| **AD9200** | 20 MSPS ADC | $30 |
| MAR-6SM+ | Amplifier (IF) | $5 |
| Mini-Circuits filter | X-band BPF | $15 |
| Connectors, cables | SMA, N-type | $15 |
| Custom PCB (RF sections) |  | $20 |
| **Subtotal Front-End** | | **~$130** |

---

## 4. Processor Subsystem

### 4.1 STM32F407 Role: Data Acquisition Controller

**CANNOT do SAR image formation** (insufficient RAM: 192 KB vs. 8+ MB needed for 512×512 image).

**What STM32F407 CAN do:**
- FMCW chirp generation (PLL control)
- ADC data capture and DMA streaming
- GPS/IMU data synchronization
- System timing and triggering
- Partial range compression (with external RAM)

### 4.2 SBC for Image Formation

**Recommended: Raspberry Pi 4B (4GB)** — Best value under $100

| Specification | Value |
|--------------|-------|
| CPU | Cortex-A72 @ 1.5 GHz (4-core) |
| RAM | 4 GB LPDDR4 |
| Performance | ~180 MFLOPS, NEON SIMD |
| Power | 5–8W |
| Cost | $55–65 |

**SAR Processing Capability:**
- 256×1024 Range-Doppler: ~80–150 ms (multi-threaded)
- 512×512 Range-Doppler: ~300–600 ms
- Can run Python/NumPy for algorithm development

**Alternative SBCs Under $100:**

| SBC | CPU | RAM | Performance | Notes |
|-----|-----|-----|-------------|-------|
| **Raspberry Pi 4B (4GB)** | Cortex-A72 | 4 GB | ~180 MFLOPS | Best community support |
| Rock Pi 4B | RK3399 | 4 GB | ~200 MFLOPS | M.2 slot for NVMe |
| Orange Pi 5 | RK3588 | 4 GB | ~400 MFLOPS | Faster but less mature |
| BeagleBone AI-64 | TDA4VM | 4 GB | ~500 MFLOPS | Built-in DSP |

### 4.3 PC-Assist Architecture

For best results, split processing:
- **Embedded (STM32):** Real-time timing, acquisition control, optional range compression
- **SBC (Pi4B):** Buffer management, pre-processing, real-time streaming to PC
- **PC:** Full SAR image formation (Range-Doppler, Back-Projection)

### 4.4 Processor Parts List

| Part | Description | Cost |
|------|-------------|------|
| Raspberry Pi 4B (4GB) | Main SBC | $60 |
| 32GB microSD (OS only) | Boot storage | $10 |
| External SSD 500GB | Raw data storage | $50 |
| **Subtotal Processing** | | **~$120** |

---

## 5. Motion Compensation Subsystem

**Critical for SAR image quality.** At X-band (λ = 3 cm), 1mm position errors cause significant phase errors.

### 5.1 GPS Requirements

| GPS Type | Accuracy | Cost | Notes |
|----------|----------|------|-------|
| **u-blox ZED-F9P** | 1–2 cm (RTK) | ~$150 | Best hobbyist option, dual-antenna capable |
| u-blox M8P | 10–20 cm | ~$60 | Good, no RTK |
| u-blox NEO-M8N | 2–3 m | ~$20 | Too coarse for SAR |

**Recommended: u-blox ZED-F9P** (~$150) — minimum for <1m resolution SAR

### 5.2 IMU Requirements

| IMU Type | Gyro Range | Noise Density | Cost |
|----------|------------|---------------|------|
| **ICM-20948** | ±250°/s | 0.005°/s/√Hz | ~$10 |
| BMI088 | ±2000°/s | 0.014°/s/√Hz | ~$15 |
| Nav2 (ESP32) | ±450°/s | 0.01°/s/√Hz | ~$30 |

**Recommended: ICM-20948** (~$10) — excellent performance, 9-axis, I2C interface

### 5.3 Motion Compensation Parts List

| Part | Description | Cost |
|------|-------------|------|
| **u-blox ZED-F9P** | RTK GPS module | $150 |
| **ICM-20948** | 9-axis IMU | $10 |
| **Subtotal Motion** | | **~$160** |

---

## 6. Signal Processing Algorithms

### 6.1 Recommended Algorithm: Range-Doppler (RDA)

**Why RDA:**
- O(N² log N) complexity — efficient
- Well-suited for stripmap mode
- Easy to implement in Python/NumPy
- Processing time on Pi4B: ~100–500 ms for 256×512 image

### 6.2 Algorithm Comparison

| Algorithm | MACs (512×512) | Quality | Suitability |
|-----------|----------------|---------|--------------|
| **Range-Doppler** | 9.4 MMACs | Good | ✅ Recommended |
| Omega-K | 9.4 MMACs | Excellent | Good for spotlight |
| Back-Projection | 134 MMACs | Best | Too heavy for embedded |
| Chirp-Scaling | ~10 MMACs | Good | Similar to RDA |

### 6.3 Processing Pipeline

```
Raw ADC Data → Range Compression → Azimuth FFT → RCMC → Azimuth Compression → SAR Image
     ↓              ↓                  ↓           ↓            ↓              ↓
   4 KB/pulse   4 KB/pulse      Streaming    Phase mult   IFFT          Display
```

### 6.4 Storage Requirements

| Data Type | Per Pulse | Per 5-min Flight | Storage |
|-----------|-----------|------------------|---------|
| Raw I/Q | 4 KB | 1.2 GB | SD card essential |
| Motion data | 100 bytes | 30 MB | STM32 internal + SD |
| Final images | 512 KB/frame | ~100 MB | PC storage |

**Note:** microSD cards are NOT suitable for raw streaming (throttle after 1–2 GB). Use USB 3.0 SSD.

---

## 7. Hobbyist Project References

### 7.1 Established DIY SAR Projects

| Project | Band | Approach | Notes |
|---------|------|----------|-------|
| **SARPi** (UK) | X-band | RTL-SDR + external front-end | Active community, open-source |
| **gr-radar** (GNU Radio) | Various | SDR-based processing | Python-based, educational |
| **MIT SAR Tutorial** | Ku-band | Custom hardware | Academic reference |
| **DIY Drone SAR** | C-band | Modified radar modules | University projects |

### 7.2 Key Resources

| Resource | URL/Source | Notes |
|----------|------------|-------|
| SARPi Project | sarpi.co.uk | UK-based X-band SAR experiments |
| GNU Radio Radar Toolbox | gr-radar (GitHub) | DSP blocks for radar |
| MIT Radar Remote Sensing | OpenCourseWare | Academic tutorial |
| RF Boutique | rfband梁.com | Affordable X-band modules |
| Nuand (PlutoSDR) | nuand.com | ADI-based SDR platform |

### 7.3 Software Stack

| Software | Purpose | Cost |
|----------|---------|------|
| GNU Radio | Signal processing | Free |
| Python + NumPy/SciPy | Algorithm development | Free |
| MATLAB (optional) | Professional processing | ~$100 student |
| Custom C++ (STM32) | Embedded processing | Free |

---

## 8. Phased Build Plan

### Phase 1: Core Radar (Weeks 1–4) — ~$150

**Goal:** Basic FMCW radar with range detection (no SAR imaging yet)

| Week | Task | Parts | Cost |
|------|------|-------|------|
| 1 | Acquire components | ADF4351, mixer, LNA, AD9200 | $75 |
| 2 | Build RF front-end | Custom PCB, connectors | $30 |
| 3 | Interface with STM32 | Cables, level shifters | $10 |
| 4 | Basic range detection | Software (existing STM32) | $0 |
| **Total** | | | **$115** |

**Deliverable:** Can detect range to targets at 100m+ (proof of concept)

### Phase 2: Motion Sensing (Weeks 5–8) — ~$165

**Goal:** Add precise position tracking for SAR

| Week | Task | Parts | Cost |
|------|------|-------|------|
| 5 | Acquire GPS/IMU | ZED-F9P, ICM-20948 | $160 |
| 6 | Interface and test | Cabling, mounting | $5 |
| 7 | EKF implementation | Software | $0 |
| 8 | Motion compensation test | Flight path logging | $0 |
| **Total** | | | **$165** |

**Deliverable:** Sub-centimeter position accuracy for motion compensation

### Phase 3: Antenna (Weeks 9–12) — ~$90

**Goal:** Custom antenna for SAR imaging

| Week | Task | Parts | Cost |
|------|------|-------|------|
| 9 | Design array | Rogers PCB fabrication | $80 |
| 10 | Build and test | Connectors, testing | $10 |
| 11 | Pattern measurement | Turntable, software | $0 |
| 12 | Integration | Mount on drone/test platform | $0 |
| **Total** | | | **$90** |

**Deliverable:** 4×4 patch array, 18–22 dBi gain, 25° beamwidth

### Phase 4: SAR Imaging (Weeks 13–20) — ~$65

**Goal:** Full SAR image formation

| Week | Task | Parts | Cost |
|------|------|-------|------|
| 13 | Acquire Pi4B | Raspberry Pi 4B, SSD | $110 |
| 14 | Data acquisition | STM32 streaming, SD logging | $0 |
| 15 | Range compression | Implement in Python/C++ | $0 |
| 16 | Azimuth processing | Range-Doppler algorithm | $0 |
| 17 | Motion compensation | Integrate GPS/IMU | $0 |
| 18–20 | Testing/iteration | Ground and drone tests | $0 |
| **Total** | | | **$110** |

**Deliverable:** 256×256 SAR image with <1m resolution

### Phase 5: Optimization (Weeks 21–26) — ~$50

**Goal:** Improve resolution, real-time processing

| Week | Task | Cost |
|------|------|------|
| 21 | Increase bandwidth (200 MHz) | $20 |
| 22 | Larger antenna (8×8 array) | $30 |
| 23–24 | GPU acceleration (optional Jetson) | $0–$150 |
| 25–26 | Field testing and iteration | $0 |
| **Total** | | **~$50–200** |

**Deliverable:** <0.5m resolution SAR imaging

---

## 9. Complete Parts List (Phase 1–4)

### RF Front-End

| Part | Part Number | Cost | Source |
|------|-------------|------|--------|
| PLL Synthesizer | ADF4351 | $12 | Analog Devices |
| X-band LNA | HMC1041 | $25 | Analog Devices |
| Mixer | ADE-30+ | $8 | Mini-Circuits |
| IF Amplifier | MAR-6SM+ | $5 | Mini-Circuits |
| X-band Filter | Crystek or DIY | $15 | eBay/surplus |
| ADC (20 MSPS) | AD9200 | $30 | Analog Devices |
| Frequency Multiplier | MAR-8SM+ | $8 | Mini-Circuits |
| Connectors | SMA/N-type assorted | $15 | Various |
| RF PCB | Custom | $20 | JLCPCB |
| **Subtotal** | | **$138** | |

### Antenna

| Part | Cost | Source |
|------|------|--------|
| Rogers RT/duroid 5880 (12"×12") | $80 | Rogers Corp |
| or FR-4 prototype | $20 | JLCPCB |
| Edge-launch SMA (4×) | $10 | eBay |
| **Subtotal** | **$90** | |

### Processing

| Part | Cost | Source |
|------|------|--------|
| Raspberry Pi 4B (4GB) | $60 | Raspberry Pi |
| External SSD 500GB | $50 | Samsung/Toshiba |
| 32GB microSD | $10 | SanDisk |
| **Subtotal** | **$120** | |

### Motion Compensation

| Part | Cost | Source |
|------|------|--------|
| ZED-F9P GPS | $150 | u-blox |
| ICM-20948 IMU | $10 | InvenSense |
| **Subtotal** | **$160** | |

### Grand Total (Phase 1–4)

| Subsystem | Cost |
|-----------|------|
| RF Front-End | $138 |
| Antenna | $90 |
| Processing | $120 |
| Motion Compensation | $160 |
| **TOTAL** | **$508** |

**Note:** Phase 1 only (core radar without SAR) is ~$150. Adding motion sensing brings to ~$265. Full Phase 4 with all subsystems is ~$508.

**Cost Reduction Options:**
- Skip RTK GPS (use M8N at ~$40) → saves $110
- Use PlutoSDR instead of custom front-end → saves ~$80 but limits X-band
- Combine phases 1–4 sequentially, reuse equipment → max savings ~$200

---

## 10. Key Design Trade-offs

### 10.1 Band Selection

| Band | Pros | Cons | Resolution | Cost |
|------|------|------|------------|------|
| **X (9–10 GHz)** | ✅ Components available, moderate size | Rain attenuation | 0.1–0.5m feasible | $ |
| C (4–8 GHz) | ✅ Penetration, loose tolerances | Larger antenna | 0.2–1m | $$ |
| Ku (12–18 GHz) | ✅ Small antenna, high resolution | Rain loss, tight tolerances | 0.05–0.2m | $$$ |

**Recommendation:** X-band for hobbyist/buildability

### 10.2 Processing Architecture

| Architecture | Pros | Cons | Real-Time? |
|-------------|------|------|------------|
| STM32-only | Low cost, simple | Cannot do SAR imaging | No |
| SBC-only | Single board | No ADC interface | Partial |
| **Hybrid (STM32 + SBC + PC)** | Best capability, modular | Complexity | Yes (with PC assist) |

**Recommendation:** Hybrid for best capability

### 10.3 GPS Selection

| GPS | Accuracy | Cost | Real-Time Correction | Recommendation |
|-----|----------|------|---------------------|----------------|
| NEO-M8N | 2–3m | $20 | None | Not suitable |
| M8P | 10–20cm | $60 | No RTK | Bare minimum |
| **F9P** | 1–2cm | $150 | NTRIP/caster | **Recommended** |

---

## 11. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| PCB fabrication tolerance at X-band | Medium | High | Order from quality fab (JLCPCB/PCBWay), test with VNA |
| LO phase noise degrades image quality | Medium | High | Use quality PLL (ADF4351), adequate filtering |
| GPS accuracy insufficient | High | Medium | Start with M8N, upgrade to F9P if needed |
| Data storage bottleneck | Low | Medium | Use SSD, not microSD; stream to PC |
| Drone payload limit exceeded | Medium | Low | Use lightweight patch array, Mavic-class compatible |

---

## 12. Next Steps

1. **Order Phase 1 components** (~$115 for core radar)
2. **Design RF front-end PCB** in KiCad/Eagle
3. **Set up STM32 project** with DMA for ADC capture
4. **Test basic range detection** with flat metal target at 50m
5. **Acquire GPS/IMU** for Phase 2
6. **Design antenna array** and order Rogers PCB

---

## Appendix: Key Equations

### Range Resolution
```
δR = c / (2 × B)
```
Where B = chirp bandwidth (Hz)

### Azimuth Resolution
```
δAz = D / 2
```
Where D = antenna along-track dimension (m)

### SAR Processing Gain
```
G_SAR = 10 × log₁₀(N_pulses) [dB]
```

### Path Loss (free space)
```
Lprop = 20 × log₁₀(4πR/λ)
```

### Required Antenna Gain (per link leg)
```
Gr > (Required_SNR - Pt - σ + Lprop + Lsys) / 2
```

---

*Document prepared from research synthesis of: ANTENNAS_RESEARCH.md, FRONTEND_RESEARCH.md, PROCESSOR_RESEARCH.md, PROCESSING_RESEARCH.md*

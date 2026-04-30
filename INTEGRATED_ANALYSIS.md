# Integrated FPV-SAR Drone Analysis — €500 Budget, 0.2m Target

**Date:** April 30, 2026  
**Sources:**  
- FPV Drone Research: https://github.com/ezzys/fpv-drone-research  
- SAR Radar Research: https://github.com/ezzys/sar-radar-research (Cycle 3 corrections)  
- Henrik Forstén DIY SAR: https://hforsten.com/homemade-polarimetric-synthetic-aperture-radar-drone.html  
- Hackaday SAR: https://hackaday.com/2025/02/13/budget-minded-synthetic-aperture-radar-takes-to-the-skies/  
**Author:** Claw (OpenClaw Agent)

---

## Executive Summary

This document integrates the FPV drone platform research with the SAR radar research to define a complete drone-mounted SAR system. The analysis was triggered by a request for a **€500 system with 0.2m SAR accuracy**.

**Key Findings:**

1. **€500 is not achievable for 0.2m resolution.** The honest budget is €635–790 depending on configuration.
2. **0.2m range resolution requires 750 MHz bandwidth.** This is extremely difficult at X-band DIY level. Realistic first-build resolution is 0.75–1.5m.
3. **STM32F407 is eliminated** as SAR processor — replaced by Raspberry Pi CM5 (3000× faster).
4. **System is split into two independent subsystems:** FPV platform and SAR payload.
5. **Phased approach recommended:** Start ground-based at C-band, then upgrade to X-band on drone.

---

## 1. System Decomposition

### 1.1 Two-System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMPLETE DRONE SAR SYSTEM                     │
├─────────────────────────────┬───────────────────────────────────┤
│     SYSTEM 1: FPV DRONE    │      SYSTEM 2: SAR PAYLOAD        │
│                             │                                   │
│  • Airframe (7" CF)         │  • RF Front-End (X-band)         │
│  • Flight Controller        │  • Processor (RPi CM5)           │
│  • Motors + ESC             │  • ADC + Data Logging            │
│  • Battery + Power          │  • Antenna (4×4 patch)           │
│  • RC Link (ELRS)           │  • GPS/IMU (shared with FC)      │
│  • FPV Video System         │  • SAR Processing Software       │
│  • GPS (RTK, shared)        │                                   │
│                             │                                   │
│  Independent: flies itself  │  Independent: processes radar     │
│  Needs: weight + power headroom │ Needs: stable flight path     │
├─────────────────────────────┴───────────────────────────────────┤
│                     GROUND STATION                               │
│  • RTK Base Station (or NTRIP service)                          │
│  • Laptop (SAR post-processing, display)                        │
│  • FPV Goggles / Monitor                                        │
│  • RC Controller                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Interface Between Systems

| Interface | Direction | Protocol | Purpose |
|-----------|-----------|----------|---------|
| GPS data | FC → RPi CM5 | UART (NMEA) | Position for SAR motion compensation |
| IMU data | FC → RPi CM5 | UART/MAVLink | Attitude for motion compensation |
| Power | Drone battery → DC-DC → SAR | 5V/3.3V rails | SAR payload power |
| Mechanical | Frame → 3D mount → SAR stack | M3 screws | Physical attachment |
| Telemetry | RPi CM5 → Ground | WiFi (SSH/MAVLink) | SAR status monitoring |

---

## 2. System 1: FPV Drone Platform

### 2.1 Component Selection

| Component | Part | Cost (€) | Notes |
|-----------|------|----------|-------|
| Frame | iFlight Nazgul5 V3 7" | 50 | Carbon fiber, proven design |
| Flight Controller | Speedybee F405 V4 | 35 | ArduPilot compatible, baro, OSD |
| ESC | Speedybee 40A 4-in-1 | 30 | BLHeli_32, DShot600 |
| Motors | iFlight XING-E 2806.5 1400KV ×4 | 60 | 6S, good thrust for payload |
| Battery | 6S 3000mAh 100C LiPo | 25 | ~8 min with SAR payload |
| Props | HQProp 7×3.5×3 ×4 pairs | 10 | Durable, efficient |
| RC Receiver | BetaFPV SuperD ELRS (2.4GHz) | 15 | ExpressLRS SPI |
| RC Controller | RadioMaster Boxer (existing) | 0 | ELRS module |
| VTX | Rush TinyBee 5.8G 1W | 25 | Analog, reliable |
| Camera | RunCam Phoenix 2 | 35 | 1200TVL, good WDR |
| VTX Antenna | Pagoda-2 (SMA) | 10 | Circular polarization |
| GPS | u-blox ZED-F9P (RTK) | 80 | **Mandatory for SAR** — shared system |
| Hardware | Standoffs, screws, straps, pads | 10 | |
| Wiring | XT60, servo leads, heatshrink | 5 | |
| **SUBTOTAL** | | **€390** | |

### 2.2 Weight Budget

| Item | Weight (g) |
|------|-----------|
| Frame (Nazgul5 7") | 180 |
| FC + ESC stack | 35 |
| Motors ×4 | 120 |
| Battery (6S 3000mAh) | 400 |
| Props ×4 | 20 |
| RC Receiver | 5 |
| VTX + Camera + Antenna | 45 |
| GPS (ZED-F9P) | 25 |
| Wiring + hardware | 30 |
| **FPV Platform Total** | **~860g** |
| **SAR Payload (see §3)** | **~550g** |
| **All-Up Weight** | **~1,410g** |

### 2.3 Flight Performance with Payload

| Parameter | Value | Notes |
|-----------|-------|-------|
| All-up weight | ~1,410g | |
| Max thrust (4× 2806.5 on 6S) | ~4,800g | 3.4:1 thrust ratio |
| Hover time | ~6-8 min | With 3000mAh, depends on SAR power draw |
| Max speed | ~80 km/h | Adequate for SAR survey |
| Wind resistance | ~40 km/h | Stable enough for SAR lines |
| Autopilot accuracy | <5cm with RTK | ArduPilot + EKF + ZED-F9P |

### 2.4 Autopilot Configuration for SAR

ArduPilot mission planner settings:
- Flight mode: AUTO (waypoint mission)
- Survey pattern: Straight parallel lines
- Line spacing: 0.5–1.0m (depends on antenna beamwidth)
- Altitude: 10–30m AGL
- Speed: 3–5 m/s (slow for better SAR integration)
- Overlap: 80%+ azimuth overlap for SAR aperture
- Trigger: Distance-based (pulse every N cm of travel)

---

## 3. System 2: SAR Payload

### 3.1 Processor Upgrade: STM32F407 → Raspberry Pi CM5

**Why STM32F407 was rejected:**

| Parameter | STM32F407 | RPi CM5 | Improvement |
|-----------|-----------|---------|-------------|
| RAM | 192 KB | 4 GB | 21,000× |
| Clock | 180 MHz | 2.4 GHz (4-core) | 13× |
| Floating Point | Single | NEON SIMD (4×) | ~50× |
| SAR 256×256 RDA | 144 seconds | 50 ms | 2,880× |
| SAR 512×512 RDA | N/A (RAM) | 200 ms | ∞ |
| OS | Bare metal | Linux | Python, NumPy |
| Storage | None | NVMe/microSD | — |
| Cost | €10 | €65 | 6.5× |

**RPi CM5 handles ALL roles that STM32F407 was supposed to do:**
- PLL/chirp control via SPI (ADF4351 programming)
- ADC data capture via SPI (AD9200)
- GPS/IMU polling and EKF fusion
- Raw data logging to NVMe
- Full SAR image formation (Python + NumPy)
- WiFi telemetry to ground station

### 3.2 Revised SAR Architecture

```
┌──────────────────────────────────────────────────────┐
│                  SAR PAYLOAD STACK                     │
│                                                       │
│  ┌──────────────────────────────────────────────────┐│
│  │       RPi CM5 (4GB) — SAR Processor              ││
│  │  ┌──────────────────────────────────────────┐    ││
│  │  │ Software Stack (Linux)                    │    ││
│  │  │ • Python + NumPy + SciPy                  │    ││
│  │  │ • Range compression (FFT per pulse)       │    ││
│  │  │ • Azimuth compression (RDA/BPA)           │    ││
│  │  │ • PGA autofocus                           │    ││
│  │  │ • GPS/IMU EKF (100 Hz)                    │    ││
│  │  │ • Raw data → NVMe (4 MB/sec sustained)    │    ││
│  │  │ • WiFi telemetry (SSH/MAVLink)             │    ││
│  │  └──────────────────────────────────────────┘    ││
│  └──────────────┬──────────────────┬────────────────┘│
│                 │ SPI+GPIO         │ UART             │
│  ┌──────────────▼──────┐  ┌───────▼──────────────┐  │
│  │  Custom Carrier PCB  │  │   u-blox ZED-F9P     │  │
│  │  • AD9200 ADC        │  │   (RTK GNSS)         │  │
│  │  • ADF4351 PLL       │  └──────────────────────┘  │
│  │  • ICM-20948 IMU     │                            │
│  │  • DC-DC regulators  │                            │
│  │  • NVMe connector    │                            │
│  └──────────┬───────────┘                            │
│             │ RF                                      │
│  ┌──────────▼───────────────────────────────────────┐│
│  │  X-band RF Front-End Module                       ││
│  │                                                    ││
│  │  TX: ADF4351(4.4GHz) → KZD-1+(×2) → 8.8GHz      ││
│  │      → PA → Directional Coupler → Antenna         ││
│  │                                                    ││
│  │  RX: Antenna → HMC1041L LNA → ZAM-50+ Mixer      ││
│  │      → ZFM-2+(IF1→IF2) → AD9200 ADC → CM5        ││
│  └───────────────────────────────────────────────────┘│
│             │                                         │
│  ┌──────────▼───────────────────────────────────────┐│
│  │  Antenna: 4×4 Patch Array (Rogers RT/duroid 5880) ││
│  │  90×90mm, 18-22 dBi, 25° beamwidth               ││
│  └───────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────┘
```

### 3.3 SAR Payload BOM

#### Option A: C-band ISM (No License, Lower Resolution)

| Component | Part | Cost (€) | Notes |
|-----------|------|----------|-------|
| Processor | RPi CM5 (4GB) | 65 | SAR + DAQ + logging |
| Carrier PCB | Custom (JLCPCB) | 30 | KiCad design |
| Storage | 64GB microSD | 10 | For initial builds |
| SDR | PlutoSDR (AD9361) | 100 | TX+RX, 12-bit, 56 MHz BW |
| PA | 5.8 GHz WiFi PA module | 15 | ~20 dBm output |
| LNA | 5.8 GHz LNA module | 10 | ~2 dB NF |
| Coupler | Directional coupler | 10 | -20 dB tap |
| Antenna | DIY 5.8 GHz patch | 15 | PCB fabrication |
| Cables + connectors | SMA, adapters | 15 | |
| 3D mount | Printed | 5 | |
| **Subtotal** | | **€265** | |
| **Range Resolution** | | **2.7m** | 56 MHz BW |
| **License** | | **None** | ISM band |

#### Option B: X-band Custom FMCW (Licensed, Better Resolution)

| Component | Part | Cost (€) | Notes |
|-----------|------|----------|-------|
| Processor | RPi CM5 (4GB) | 65 | SAR + DAQ + logging |
| Carrier PCB | Custom (JLCPCB) | 30 | ADC + PLL + IMU onboard |
| Storage | 64GB microSD | 10 | Or NVMe on carrier |
| Synthesizer | ADF4351 breakout | 15 | LO generation |
| Multiplier | KZD-1+ (×2) | 20 | → X-band LO |
| Mixer | ZAM-50+ | 25 | X-band wideband |
| LNA | HMC1041L | 40 | X-band, NF 2.5 dB |
| IF chain | ZFM-2+ + MAR-6SM+ | 20 | Two-stage downconversion |
| ADC | AD9200 (20 MSPS) | 30 | 10-bit, IF sampling |
| RF PCB | Custom (JLCPCB) | 30 | X-band layout |
| Antenna | 4×4 patch (Rogers) | 60 | 18-22 dBi |
| Cables + connectors | SMA (X-band rated) | 25 | |
| 3D mount | Printed | 5 | |
| **Subtotal** | | **€375** | |
| **Range Resolution** | | **0.75–1.5m** | 100-200 MHz BW |
| **License** | | **Amateur radio** | 10 GHz band |

### 3.4 SAR Payload Weight

| Component | Weight (g) |
|-----------|-----------|
| RPi CM5 + heatsink | 30 |
| Carrier PCB + components | 40 |
| RF front-end module | 80 |
| Antenna (4×4 patch) | 100 |
| Cables + connectors | 50 |
| 3D mount + hardware | 30 |
| MicroSD/NVMe | 10 |
| **Total SAR Payload** | **~340g** |

*Note: GPS (ZED-F9P, 25g) counted in FPV platform since it's shared.*

### 3.5 SAR Payload Power

| Component | Power (W) |
|-----------|-----------|
| RPi CM5 (active) | 5–8 |
| RF front-end | 3 |
| GPS + IMU | 0.5 |
| **Total** | **8.5–11.5W** |

Draws from main flight battery via DC-DC converter. Impact on flight time: ~1-2 min reduction.

---

## 4. Ground Station

| Component | Cost (€) | Notes |
|-----------|----------|-------|
| RTK Base Station | 50 | ZED-F9P + antenna, OR use free NTRIP (EUREF/EPOS) |
| Laptop | 0 | Existing — SAR post-processing |
| FPV Monitor/Goggles | 0 | Existing |
| **Total** | **€0–50** | |

NTRIP services in Lithuania: <https://epos-gnss.eu> — free RTK corrections via mobile internet.

---

## 5. Budget Summary

| System | Option A (C-band) | Option B (X-band) |
|--------|-------------------|-------------------|
| FPV Platform | €390 | €390 |
| SAR Payload | €265 | €375 |
| Ground Station | €0–50 | €0–50 |
| **TOTAL** | **€655–705** | **€765–815** |
| Range Resolution | 2.7m | 0.75–1.5m |
| License Required | No | Yes (amateur) |

### Cost Reduction Options

| Change | Saves | Impact |
|--------|-------|--------|
| FR-4 antenna instead of Rogers | €40 | -3 dB gain |
| RPi 4B instead of CM5 | €10 | Heavier, slower |
| microSD instead of NVMe | €15 | Slower data logging |
| NTRIP instead of RTK base | €50 | Needs mobile signal |
| PlutoSDR ground-based first | €200+ | No flight, learning phase |
| Cheaper motors (EMAX) | €20 | Slightly less thrust |

---

## 6. Path to 0.2m Resolution

### 6.1 Why 0.2m Is Hard

Range resolution: δR = c/(2B)
- 0.2m requires 750 MHz bandwidth
- At X-band (9.6 GHz): 750/9600 = 7.8% fractional BW — difficult but possible
- At C-band (5.8 GHz): 750/5800 = 12.9% fractional BW — very difficult
- At 77 GHz: 750/77000 = 1% fractional BW — *easy*

### 6.2 Options for 0.2m

| Approach | Feasibility | Cost | Risk |
|----------|-------------|------|------|
| **77 GHz automotive radar** (TI AWR1642) | Medium | +€50 | Firmware lock-in, limited raw data access |
| **Wideband X-band VCO** (HMC587LC4B) | Hard | +€80 | Phase noise, chirp linearity |
| **SAR azimuth processing only** (azimuth ≠ range) | Best first step | €0 | 0.15m azimuth, but range stays 0.75m |
| **Multi-look averaging** | Easy | €0 | Improves SNR but not resolution |
| **Buy commercial SAR data** (ICEYE) | Trivial | €€€ | 0.25m data, no DIY learning |

### 6.3 Realistic Resolution Targets

| Build Phase | Range Res | Azimuth Res | Combined | Bandwidth |
|-------------|-----------|-------------|----------|-----------|
| Phase 1 (ground C-band) | 2.7m | N/A | 2.7m | 56 MHz |
| Phase 2 (drone X-band) | 1.5m | 0.15m | ~0.5m | 100 MHz |
| Phase 3 (wideband X) | 0.75m | 0.15m | ~0.25m | 200 MHz |
| Phase 4 (77 GHz or 750 MHz BW) | 0.2m | 0.15m | ~0.15m | 750 MHz |

**Recommendation:** Aim for Phase 2 as the deliverable. Phase 3 is a realistic stretch. Phase 4 is research territory.

---

## 7. SAR Processing Pipeline

### 7.1 Data Flow

```
1. CAPTURE (on drone)
   Radar chirp → IF → ADC (20 MSPS) → RPi CM5 SPI
   GPS position (RTK, 20 Hz) → RPi CM5 UART
   IMU attitude (ICM-20948, 200 Hz) → RPi CM5 I2C
   ↓
   Log to microSD/NVMe: raw I/Q + timestamps + GPS + IMU

2. PREPROCESSING (on RPi CM5, post-flight or streaming)
   Parse binary log → NumPy arrays
   GPS/IMU EKF fusion → trajectory (x,y,z per pulse)
   Range compression: FFT each pulse × matched filter
   ↓
   Range-compressed data (2D: range × azimuth)

3. IMAGE FORMATION (on RPi CM5 or laptop)
   Back-Projection Algorithm (BPA):
     For each pixel (r, x):
       For each pulse n at position P(n):
         range = ||P(n) - (r,x)||
         phase_corrected[n] = data[n, range] × exp(-j×4π×range/λ)
       image[r,x] = sum(phase_corrected)
   OR
   Range-Doppler Algorithm (RDA):
     Range FFT → RCMC → Azimuth FFT → Azimuth compression
   
   PGA autofocus (3-5 iterations):
     Estimate phase error from bright scatterers
     Apply correction, re-form image
   ↓
   SAR image (geo-referenced if GPS quality good)

4. OUTPUT
   GeoTIFF / PNG with metadata
   Compare with optical imagery for validation
```

### 7.2 Processing Performance (RPi CM5)

| Image Size | Algorithm | Time | Notes |
|------------|-----------|------|-------|
| 256×256 | RDA | ~50 ms | Real-time capable |
| 512×512 | RDA | ~200 ms | ~5 fps |
| 512×512 | BPA | ~2 sec | Post-flight |
| 1024×1024 | RDA | ~800 ms | ~1 fps |
| 256×256 | PGA autofocus | +30 ms | Per-frame addition |

### 7.3 Storage Requirements

| Duration | Raw Data | Compressed | Storage |
|----------|----------|------------|---------|
| 5 min flight | ~1.2 GB | ~600 MB | microSD OK |
| 30 min survey | ~7.2 GB | ~3.6 GB | NVMe preferred |
| Full calibration set | ~20 GB | ~10 GB | Laptop |

---

## 8. Execution Plan

### Phase 1: Ground-Based Learning (€100, Weeks 1–4)

**Goal:** First SAR image, no drone needed

- Buy PlutoSDR (~€100)
- Build simple C-band FMCW on bench
- Implement range compression + BPA in Python
- Use car on straight road as "platform"
- **Gate:** SAR image of parking lot, 3m resolution

### Phase 2: Drone Platform Build (€390, Weeks 5–8)

**Goal:** Flying platform ready for SAR payload

- Build 7" quad from fpv-drone-research BOM
- Install RTK GPS, configure ArduPilot
- Fly automated survey patterns (without SAR payload first)
- Validate position logging accuracy
- **Gate:** 5 min stable flight with <5cm RTK accuracy on straight lines

### Phase 3: X-band SAR on Drone (€375, Weeks 9–16)

**Goal:** SAR images from drone flights

- Build X-band FMCW front-end (use Cycle 3 corrected BOM from sar-radar-research)
- Design and fabricate custom carrier PCB (KiCad → JLCPCB)
- Integrate RPi CM5 + SAR payload on drone
- First flights: raw data capture only
- Post-process on laptop (Python pipeline from Phase 1)
- **Gate:** SAR image from drone flight, 1.5m resolution

### Phase 4: Optimization (€50–100, Weeks 17–20)

**Goal:** Push resolution toward 0.5m

- Replace VCO with wider bandwidth option
- Implement PGA autofocus
- Tune flight parameters (speed, altitude, path spacing)
- Calibrate with trihedral corner reflectors
- **Gate:** demonstrated <0.5m combined resolution

### Phase 5: 0.2m Target (research, Weeks 21+)

**Goal:** Approach 0.2m resolution

- Evaluate 77 GHz AWR1642BOOST for raw data extraction
- OR design wideband X-band front-end (HMC587LC4B VCO)
- OR accept 0.5m as practical limit and optimize image quality
- **Gate:** 0.2m resolution on calibration targets (ambitious)

---

## 9. Key Risks and Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| RTK GPS accuracy degrades in flight (vibration, multipath) | High | IMU fusion via EKF, vibration-damped mount |
| SAR payload weight reduces flight time below usable | Medium | Optimize weight, use larger battery (6S 5000mAh) |
| X-band RF debugging without spectrum analyzer | High | Use second SDR as spectrum monitor; start with C-band |
| Data rate exceeds RPi CM5 SPI bandwidth | Medium | Range-compress on-device before logging |
| Motion compensation insufficient for 0.2m | High | PGA autofocus essential; accept 0.5m as realistic |
| Regulatory (X-band transmission without license) | Medium | Get amateur radio license OR stick to C-band ISM |
| Carrier PCB design errors | Medium | Start with breadboard prototype, iterate before PCB order |

---

## 10. Corrections from sar-radar-research Repo

The SAR research repo went through 3 review cycles. The following critical errors were caught in Cycle 2 and corrected in Cycle 3:

| Error | Original (Cycle 1) | Corrected (Cycle 3) |
|-------|--------------------|----------------------|
| HMC1041 package | Bare die (wire-bond required) | HMC1041L module (€40) |
| Frequency multiplier | MAR-6/MAR-8 (amplifiers!) | KZD-1+ (actual ×2 multiplier) |
| Mixer | ADE-30+ (doesn't exist) | ZAM-50+ (X-band mixer) |
| Budget | $200 claimed | $570–650 honest estimate |
| Resolution math | Implied <1m from 100 MHz | Corrected: 100 MHz = 1.5m |
| X-band legal | Not addressed | Requires amateur radio license |
| ADC sampling | No IF strategy specified | Two-stage heterodyne to 10 MHz IF |
| STM32F407 SAR | Implied it could process | Cannot — 192KB vs 8MB needed |
| Link budget | Not provided | Negative margin at 100m, need higher power or closer range |
| Weight/power | Not provided | 585–670g, 8–24W |

**ALWAYS use Cycle 3 (CYCLE_3_FINAL_ROADMAP.md) as the canonical design. Cycle 1 has dangerous errors.**

---

## 11. Recommended Reading

1. **Henrik Forstén's DIY Polarimetric SAR:** <https://hforsten.com/homemade-polarimetric-synthetic-aperture-radar-drone.html>
2. **Hackaday Budget SAR:** <https://hackaday.com/2025/02/13/budget-minded-synthetic-aperture-radar-takes-to-the-skies/>
3. **RTL-SDR Drone SAR:** <https://www.rtl-sdr.com/creating-a-drone-based-synthetic-aperture-radar/>
4. **Cumming & Wong:** *Digital Processing of Synthetic Aperture Radar Data* (textbook)
5. **TI IWR6843 SAR application note:** Search for "mmWave SAR" on TI.com

---

*Document compiled by Claw (OpenClaw Agent) — April 30, 2026*  
*Based on: fpv-drone-research + sar-radar-research (Cycle 3) + web research*

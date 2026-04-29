# DIY SAR Radar — Cycle 2 Critical Review

**Date:** April 30, 2026  
**Reviewing:** CYCLE_1_SYNTHESIS.md  
**Reviewer:** Hermes Agent (Automated Critical Review)  
**Status:** Issues Found — Do Not Proceed to Build

---

## Executive Summary

**The document claims a "$200" budget target but the actual Phase 1-4 total is $508.** This is a 2.5× budget overrun before any unexpected costs. Several part numbers are incorrect or inappropriate for hobbyist DIY. X-band regulatory/licensing issues are not addressed. The STM32F407 role is correctly identified but other technical claims have accuracy issues.

**Verdict: Not ready for implementation. Budget and parts list require significant correction.**

---

## 1. BUDGET VIOLATION — CRITICAL

### 1.1 The $200 Claim is False

| Section | Claim | Actual |
|---------|-------|--------|
| Header | "Budget Target: Under $200" | — |
| Executive Summary | "Total Phase 1 Build Cost: ~$265" | $265 exceeds $200 by 32% |
| Section 9 (Grand Total) | — | **$508** for Phase 1-4 |
| Phase 4 | "$110" | Pi4B+SSD alone = $110 |

**The "$200 budget" stated in the header is violated by all build phases.** Even Phase 1 alone (~$115-$150) plus minimal motion sensing (~$160 for GPS alone) exceeds $200.

### 1.2 Per-Subsystem Budget Errors

| Subsystem | Listed Cost | Issue |
|-----------|-------------|-------|
| RF Front-End | $138 | Underestimates; real pricing likely higher |
| Antenna (Rogers PCB) | $90 | Accurate for Rogers; $80 just for 12"×12" substrate |
| Processing | $120 | Only if Phase 1-4 sequential (reusing equipment); real total is $120+$50(SSD) = $170 if bought together |
| Motion Compensation | $160 | ZED-F9P alone is ~$150; correct |
| **TOTAL** | **$508** | Exceeds $200 by 154% |

**Recommendation:** Either abandon the "$200" claim entirely, or specify which components can be deferred/skipped to stay under $200 for a minimal proof-of-concept only.

---

## 2. PART NUMBER / PRICING ERRORS — MAJOR

### 2.1 HMC1041 — Incorrect Package for DIY

| Issue | Details |
|-------|---------|
| **Part listed** | Analog Devices HMC1041 |
| **Problem** | HMC1041 is a **bare die GaAs MMIC** — requires wire bonding, special handling, carrier PCB, and is not available in tape-and-reel for hobbyist assembly |
| **Reality check** | The HMC1041 data sheet shows it as a 1.45mm × 1.0mm die, not an SMT package |
| **Correct alternative** | HMC1041L (plug-in module) or use a different X-band LNA in a real package |
| **Actual cost** | If found on query, HMC1041 die requires minimum order from Analog Devices; not available from typical distributors |

**Impact:** The $25 LNA cost and the entire X-band receiver chain is unbuildable as described.

### 2.2 ADE-30+ Mixer — Part Number Likely Invalid

| Issue | Details |
|-------|---------|
| **Part listed** | Mini-Circuits ADE-30+ |
| **Problem** | Mini-Circuits does not have an "ADE-30+" part in their standard product line. Their RMS detector product line uses different naming (e.g., AD-30+, AD-50+). |
| **Confusion** | The ADE prefix typically denotes their RMS detectors, not mixers. The ADE-30+ may not exist. |
| **Correct mixer alternatives** | Mini-Circuits ZFM-4212 ($15), ZFM-3 ($10), or TUF-3+ ($20) |
| **Impact** | Budget understated; actual X-band mixer would be different |

### 2.3 MAR-6 / MAR-8 Confusion — Wrong Part Type

| Issue | Details |
|-------|---------|
| **Listed** | MAR-6SM+ (amplifier, $5) and MAR-8SM+ (multiplier, $8) |
| **Problem** | MAR-6 is a **amplifier** (DC-6 GHz), not a frequency multiplier. Using it as a ×2 multiplier in the LO chain is incorrect. |
| **MAR-8** | Also an amplifier (2-8 GHz), not a multiplier |
| **What the LO chain actually needs** | A proper frequency multiplier (e.g., A/D type diode-based, or a multiplier module from Kyocera, Custom Power, etc.) |
| **Impact** | The 9.6 GHz LO generation plan is unbuildable as described |

### 2.4 Pricing Accuracy

| Part | Listed | Market Check |
|------|--------|--------------|
| ADF4351 | $12 | ~$15-20 (DigiKey/Mouser) |
| AD9200 | $30 | ~$20-35 (ADI) |
| Raspberry Pi 4B (4GB) | $60 | ~$55-65 (correct) |
| ZED-F9P | $150 | ~$150-170 (correct) |
| HMC1041 | $25 | Not available as hobbyist part |
| ICM-20948 | $10 | ~$10-15 (correct) |

---

## 3. STM32F407 ROLE — CORRECT

The document correctly identifies that the STM32F407:
- **CANNOT** do SAR image formation (192 KB RAM vs. 8+ MB needed)
- **CAN** do: FMCW chirp generation, ADC data capture and DMA, GPS/IMU sync, system timing

This section is accurate and well-reasoned.

---

## 4. REGULATORY ISSUES (X-BAND) — INADEQUATELY ADDRESSED

### 4.1 No License Discussion

The document states "X-band (9–10 GHz) — Not ISM; 9.6 GHz common for radar modules" but **does not mention**:

1. **FCC Licensing requirement** — In the US, X-band radar operation requires either:
   - Part 15 (unlicensed) — NOT available for radar at these power levels
   - Part 90 (business/land mobile) — requires license
   - Experimental Radio License — possible but limited scope
   - Amateur radio (3 cm band, 10 GHz) — legal but power limited to 10W EIRP

2. **ITU allocation** — X-band (8-12 GHz) is **primary allocated for radiolocation** (radar) but national regulations vary

3. **Flight regulations** — Operating radar from a drone may require additional waivers (FAA Part 107 restrictions on intentional emission)

4. **Power limits** — The document nowhere specifies transmit power, which affects both regulatory compliance and safety

### 4.2 ISM Band Missed Opportunity

| Band | Frequency | License Status |
|------|-----------|----------------|
| **C-band** | 5.8 GHz (5725-5875 MHz) | **ISM — unlicensed** |
| **Ku-band** | 13.75-14.5 GHz | Not ISM, requires license |
| **Ka-band** | 24.05-24.25 GHz | ISM portion available |
| **X-band** | 9.6 GHz | **NOT ISM** — radar primary |

**The document recommends X-band (9-10 GHz) but does not discuss regulatory burden or consider ISM alternatives like C-band for a hobbyist build.**

---

## 5. TECHNICAL INCOMPLETENESS / ERRORS

### 5.1 Range Resolution Math Error

| Claim | Problem |
|-------|---------|
| Section 2.1: "at 100 MHz bandwidth → 1.5 m" | Correct formula: δR = c/(2B) = 3×10⁸/(2×100×10⁶) = 1.5m ✓ |
| Section 6: "256×256 SAR image with <1m resolution" | With 100 MHz bandwidth, resolution is 1.5m — cannot achieve <1m resolution |

**To get <1m range resolution requires >150 MHz bandwidth. For 0.5m resolution, need 300 MHz bandwidth.**

### 5.2 Missing System Gain Budget

The document never provides a **complete link budget** showing:
- Transmit power (never specified)
- Antenna gain (Tx and Rx)
- Path loss at intended range
- Receiver noise figure
- ADC sensitivity
- Required SNR for detection

Without this, claims like "100m+ range" (Section 6) are unsubstantiated.

### 5.3 LO Chain Design Issues

The proposed LO chain: `ADF4351 → 4.8 GHz → ×2 multiplier → 9.6 GHz LO`

Issues:
1. **ADF4351 maximum output is 4.4 GHz** (not 4.8 GHz) — section states "4.8 GHz output" which is out of spec
2. **×2 multiplier missing proper part** — MAR-6/MAR-8 are amplifiers, not multipliers
3. **No specification of multiplier loss** — typically 6-10 dB loss in a passive doubler
4. **No LO power level specified** — mixer LO drive requirement not given

### 5.4 ADC/IF Sampling Issues

| Claim | Issue |
|-------|-------|
| "STM32F407 CANNOT directly sample IF for SAR" | Correct |
| "AD9200 (20 MSPS, 10-bit, ~$30)" | Correct part, but 20 MSPS cannot directly sample 300 MHz IF even with undersampling — needs bandpass sampling or downconversion to <10 MHz IF |
| **Missing IF downconversion stage** | The heterodyne architecture shows "IF (e.g., 100-500 MHz)" but doesn't specify a second LO/synthesizer for the IF downconversion to a sampleable range |

### 5.5 Motion Compensation: Missing EKF Detail

Section 5.1 states "EKF implementation" but provides:
- No sensor fusion architecture
- No state vector definition
- No process/measurement noise models
- No coupling between GPS/IMU data rates (GPS: 10 Hz, IMU: typically 100+ Hz)

This is a significant software gap not reflected in the schedule (Weeks 5-8 for "EKF implementation" with no prior work).

---

## 6. PHASE PLAN INCONSISTENCIES

| Phase | Week 13 | Parts Listed | Cost |
|-------|---------|--------------|------|
| Phase 4 | Acquire Pi4B | Pi4B + SSD | $110 |
| Phase 1 | — | RF components | $115 |
| Phase 2 | — | GPS/IMU | $165 |
| Phase 3 | — | Antenna | $90 |

**Cumulative cost if ordered together:**
- Phase 1 + 2 + 3 + 4 = $115 + $165 + $90 + $110 = **$480** (plus shipping)
- Plus STM32F407 (assumed pre-existing, but should be listed)

**Phase 5 "Optimization" adds $50-200 more, further from "$200 budget."**

---

## 7. DRONE PAYLOAD/WEIGHT CONSIDERATIONS

The document mentions:
- Mavic-class (~900g): 100-200g payload
- 4×4 patch array: 80-120g

But:
- RF front-end (custom PCB, connectors, shields, LNA, mixer, filter) is likely 200-400g
- GPS/IMU: ~50-100g
- Total: likely 400-600g — **may exceed Mavic-class payload limit**

No consolidated weight budget is provided.

---

## 8. ITEMS NOT ADDRESSED

1. **Power consumption** — No budget for voltage regulation, battery sizing, or power distribution
2. **Mechanical mounting** — RF shielding, vibration isolation for IMU, antenna mounting
3. **VNA/calibration** — Required for X-band but not in budget (~$200 for used VNA)
4. **Software stack** — STM32 firmware, Pi4B data pipeline, PC processing — all "software" tasks in phases with no actual code references
5. **RF connector torque/specification** — SMA connections at X-band require proper torque (5-8 in-lbs) and care

---

## 9. RECOMMENDATIONS

### Must Fix Before Build:
1. **Remove "$200" from title/header** — or specify a truly minimal $200 proof-of-concept scope
2. **Correct HMC1041** — Replace with HMC1041L (if available) or specify alternative X-band LNA in a real package (e.g., Skyworks SKY65804, or use waveguide-based frontend)
3. **Fix mixer part number** — Use ZFM-4212 or similar confirmed Mini-Circuits mixer
4. **Fix LO chain** — Specify actual frequency multiplier, not amplifier
5. **Add regulatory section** — Discuss FCC Part 15/90/experimental options or recommend C-band ISM alternative
6. **Correct bandwidth/resolution claims** — State bandwidth requirements for target resolution
7. **Add system gain budget** — Prove the link works before claiming range

### Should Fix:
1. **Clarify ADC sampling strategy** — Bandpass sampling or additional IF downconversion
2. **Add weight budget** — Confirm total system fits on target drone
3. **Specify power budget** — Voltage rails, current draw, battery life
4. **Add EKF architecture detail** — Even high-level state vector and sensor fusion approach

---

## 10. SUMMARY TABLE

| Issue | Severity | Type |
|-------|----------|------|
| Budget claim ($200) vs actual ($508) | **Critical** | Budget |
| HMC1041 bare die, not hobbyist-usable | **Critical** | Parts |
| LO chain uses amplifiers as multipliers | **Critical** | Technical |
| X-band regulatory/licensing not addressed | **Critical** | Regulatory |
| ADC/IF sampling strategy incomplete | **Major** | Technical |
| Range resolution math inconsistent | **Major** | Technical |
| ADE-30+ mixer part number likely wrong | **Major** | Parts |
| No system gain/loss budget | **Major** | Technical |
| Phase plan cumulative cost not reconciled | **Major** | Budget |
| No weight/power budgets | **Moderate** | Technical |
| EKF/software gap not reflected in schedule | **Moderate** | Technical |

**Overall Assessment:** The document provides a reasonable research framework but **cannot be used as a build guide** without significant corrections to parts selection, budget reconciliation, and regulatory compliance. The "$200" constraint is fundamentally incompatible with the stated architecture.

---

*Review generated: April 30, 2026*  
*Reviewing: CYCLE_1_SYNTHESIS.md (595 lines)*

# DIY Synthetic Aperture Radar (SAR) — Research Repository

**Status:** Research complete (3 review cycles)  
**Goal:** Practical DIY SAR for hobbyist / drone-mounted use  
**Budget:** ~$430-650 depending on configuration

---

## What We Learned (Summary)

### ❌ X-band (9-10 GHz) — Harder Than Expected
- Custom X-band RF front-end is complex: needs LO multiplier chain, specific mixers, bare-die components
- HMC1041 (LNA) is **bare die**, not hobbyist-mountable — needs HMC1041L (~$40 module)
- ADE-30+ mixer doesn't exist — use ZFM-4212
- LO chain needs proper frequency multiplier, not amplifier (MAR-6/MAR-8 are amplifiers)
- **X-band also requires amateur radio license or experimental license** (not ISM)
- Full custom X-band SAR: **~$570-650**

### ✅ C-band (5.8 GHz ISM) — Recommended for Hobbyists
- No license needed (ISM band)
- Cheap PA/LNA modules widely available ($5-20)
- Decent resolution: 0.75m at 200 MHz bandwidth
- Full C-band SAR: **~$430-480**
- Components: AD9361 SDR (PlutoSDR), 5.8 GHz PA + LNA, patch antenna

### STM32F407 Role
- **CAN:** FMCW chirp generation, ADC capture/DMA, GPS/IMU sync, system timing
- **CANNOT:** Full SAR image formation (192 KB RAM vs. 8+ MB needed)
- **Best use:** Data acquisition controller streaming to Raspberry Pi 4B

### Realistic Budget
| Phase | Scope | Cost |
|-------|-------|------|
| Phase 0 | SDR-only SAR (PlutoSDR + Python) | ~$150 |
| Phase 1 | Basic FMCW radar (C-band) | ~$200-300 |
| Phase 2 | Add GPS/IMU motion comp | +$160 |
| Phase 3 | Custom RF front-end | +$100-200 |
| Full | Complete drone SAR | **~$430-650** |

### Key Corrections from Initial Design
1. ~~HMC1041~~ → HMC1041L or CMD328 QFN
2. ~~MAR-6 as multiplier~~ → Proper diode doubler or KZD-1+
3. ~~ADE-30+ mixer~~ → ZFM-4212
4. ~~$200 budget~~ → Reality: $430-650 for working SAR
5. ~~X-band without license~~ → C-band ISM recommended

---

## Repository Structure

```
sar-radar-research/
├── README.md                          # This file
├── PROJECT_STATE.md                   # Research objectives & questions
├── research/
│   ├── 01_antennas/                  # Antenna subsystem research
│   ├── 02_front_end/                 # RF front-end research
│   ├── 03_processor/                 # Processor/hardware research
│   └── 04_processing/                # Signal processing research
└── review_cycles/
    ├── CYCLE_1_SYNTHESIS.md          # Initial design synthesis
    ├── CYCLE_2_REVIEW.md             # Critical review (found 4 critical issues)
    └── CYCLE_3_FINAL_ROADMAP.md     # Corrected final design
```

---

## Key Decisions Made

1. **C-band (5.8 GHz ISM) primary recommendation** — legal, cheap, available components
2. **X-band still viable** with amateur radio license, but higher complexity/cost
3. **Hybrid architecture**: STM32F407 (DAQ) + Raspberry Pi 4B (preprocessing) + PC (image formation)
4. **Range-Doppler Algorithm** (RDA) for SAR imaging — O(N² log N), ~100-500 ms on Pi4B
5. **Two-stage heterodyne** for IF sampling (RF→400 MHz IF1→10 MHz IF2→ADC)
6. **Resolution reality**: 100 MHz → 1.5m; <0.75m requires 200+ MHz bandwidth

---

## Not Covered (Future Work)
- STM32 firmware development
- KiCad PCB designs
- Python/NumPy SAR processing code
- Drone integration mechanical design
- Specific RTL-SDR/PlutoSDR configuration for SAR

---

*Research conducted: April 2026*
*Agent: Hermes (MiniMax-M2.7)*

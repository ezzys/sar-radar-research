# RF Front-End Hardware Research for DIY SAR

**Date:** April 29, 2026
**Budget Constraint:** Under $200 total system cost
**Target Application:** Synthetic Aperture Radar (DIY/ Amateur)

---

## 1. Architecture Comparison: Direct Sampling vs Heterodyne

### Direct Sampling (Zero-IF / RF Sampling)

**Principle:** The RF signal is sampled directly at the carrier frequency without frequency conversion.

**Advantages:**
- Simpler hardware (fewer mixing stages)
- No image frequency problems
- Lower component count = lower cost and complexity
- Better phase noise characteristics (fewer PLLs)
- Easier calibration

**Disadvantages:**
- ADC must sample at >2× carrier frequency (Nyquist)
- Very high ADC sample rates required for X-band (10 GHz)
- Dynamic range limited by ADC ENOB at high sample rates
- Anti-aliasing filter requirements are stringent

**Viability for DIY SAR:**
- Impractical for X-band (10 GHz) with current ADC technology
- Viable for L-band (1-2 GHz) with affordable ADCs
- Playout stages like the AD9200 (20 MSPS) work at lower frequencies

### Heterodyne Architecture

**Principle:** RF signal is downconverted to intermediate frequency (IF) or baseband before sampling.

**Advantages:**
- ADC operates at lower frequencies (easier to find high-performance ADCs)
- Can use narrowband bandpass filters for image rejection
- Gain distribution across multiple stages
- Better selectivity

**Disadvantages:**
- More components (mixers, filters, local oscillators)
- Image frequency responses require filtering
- LO leakage and feedthrough issues
- Phase noise from multiple PLLs can accumulate
- Higher complexity and cost

**Viability for DIY SAR:**
- **Recommended for X-band** - easier to achieve good performance
- Enables use of slower, higher-resolution ADCs
- Allows channelization of wide bandwidths

### Recommendation for DIY SAR

**X-band (9-10 GHz):** Heterodyne with IF sampling is the only practical approach with a $200 budget. Direct sampling would require ADCs sampling at >20 GSPS, which are beyond budget.

**L/C-band (1-6 GHz):** Direct sampling becomes viable with affordable 100+ MSPS ADCs.

---

## 2. Dynamic Range and ENOB Requirements

### SAR Radar Dynamic Range Fundamentals

For a SAR system to detect small backscatter changes against strong clutter:

| Scenario | Required Dynamic Range |
|----------|------------------------|
| Basic detection (1-bit shadow) | 10-20 dB |
| Moderate resolution imaging | 40-60 dB |
| High-quality SAR (weak scatterers) | 60-80 dB |
| Professional-grade SAR | 80-100+ dB |

### ENOB Requirements by Architecture

**Effective Number of Bits (ENOB)** determines usable dynamic range:

```
ENOB = (SNR - 1.76) / 6.02
Dynamic Range (dB) ≈ 6.02 × ENOB + 1.76
```

| ENOB | Dynamic Range | Application |
|------|---------------|-------------|
| 6 bits | ~38 dB | Basic detection, strong targets |
| 8 bits | ~50 dB | Simple SAR imaging |
| 10 bits | ~62 dB | Moderate SAR (DIY feasible) |
| 12 bits | ~74 dB | Quality SAR imaging |
| 14 bits | ~86 dB | Professional systems |

### Minimum ENOB for DIY SAR

**Recommended minimum: 10-12 bits ENOB**

- 10-bit ADC (e.g., AD9200, ADS54J20) provides ~62 dB dynamic range
- This is sufficient for basic to moderate SAR imaging
- Budget ADCs often advertise 10-bit resolution but effective bits degrade at high input frequencies

### FMCW Radar vs Pulse Radar

**FMCW (Frequency Modulated Continuous Wave):**
- Measures beat frequency (difference between TX and RX)
- Requires linear sweep - chirp linearity critical
- Lower peak power requirements
- Easier to achieve with budget hardware
- Dynamic range needs: 60-80 dB

**Pulse Radar:**
- Measures time-of-flight
- Requires high peak power pulses
- stricter timing requirements
- Better for long-range
- Dynamic range needs: 40-60 dB for basic systems

**Recommendation:** FMCW is far more practical for DIY SAR under $200.

---

## 3. Key RF Components

### 3.1 Low Noise Amplifier (LNA)

**Purpose:** Amplify weak received signal while adding minimal noise

**Critical Parameters:**
- Noise Figure (NF): Should be <2 dB for X-band
- Gain: 20-30 dB typical
- Bandwidth: Must cover operating frequency
- P1dB (compression point): Higher is better
- Input IP3: High linearity

**Budget X-band LNA Options:**

| Part | NF | Gain | Freq | Cost | Notes |
|------|-----|------|------|------|-------|
| Mini-Circuits GVA-83+ | 2.5 dB | 22 dB | 2-8 GHz | ~$15 | May need bypass for X-band |
| Qorvo TQP3M9008 | 1.0 dB | 22 dB | 0.5-6 GHz | ~$12 | Not X-band |
| Analog Devices HMC1041 | 2.5 dB | 20 dB | 2-30 GHz | ~$25 | True X-band |
| LNN X-band demo board | 1.5 dB | 25 dB | 9-11 GHz | ~$30 | Module solution |

**DIY Alternative:** Use cascaded cheaper LNAs (e.g., BGA2817) for partial X-band performance at lower cost.

**Critical Note:** LNA is the most critical component for sensitivity. Never skip on the LNA budget.

### 3.2 Mixer

**Purpose:** Frequency conversion (up/down conversion)

**Key Parameters:**
- Conversion loss (active) or noise figure (passive)
- LO drive level requirements
- Port-to-port isolation
- Spurious responses

**Mixer Types:**

| Type | Conversion Loss | LO Power | Cost | Notes |
|------|-----------------|----------|------|-------|
| Passive (diode) | 5-8 dB | +7 to +23 dBm | Low | No noise figure, just loss |
| Active (Gilbert) | Gain positive | +0 to +10 dBm | Medium | Integrated in many chips |
| Image reject | 5-8 dB | +10 to +23 dBm | Higher | Built-in filtering |

**Budget Options:**
- Mini-Circuits ADE-30+ (RMS diode, DC-3 GHz IF) - ~$8
- Analog Devices ADL5811 (active, 10-80 GHz) - ~$15
- DIY discrete diode mixers possible but performance variable

**For X-band:** Integrated front-end modules often include mixer + LNA + filters

### 3.3 VCO / PLL (Voltage Controlled Oscillator / Phase Locked Loop)

**Purpose:** Generate stable local oscillator signals

**Key Parameters:**
- Phase noise (dBc/Hz @ 10 kHz offset)
- Frequency range
- Step size / resolution
- Lock time
- Spurii levels

**PLL vs Direct VCO:**

| Approach | Phase Noise | Flexibility | Cost | Complexity |
|----------|-------------|-------------|------|------------|
| Direct VCO | Better | Less | Lower | Lower |
| PLL with divider | Worse | High | Higher | Higher |

### 3.4 Filters

**Purpose:** Select desired frequency band, reject interference and images

**Filter Types for RF:**

| Type | Q factor | Cost | Application |
|------|----------|------|-------------|
| LC (lumped) | Medium | Low | <1 GHz |
| Cavity | High | High | X-band, narrowband |
| Microstrip | Medium | Medium | X-band |
| SAW | Very High | Medium | <1 GHz |
| Helical | High | Medium | VHF-UHF |

**X-band Filters:**
- Cavity filters: Best performance but expensive (>$100)
- Microstrip hairpin/combline: Moderate cost, DIY possible
- Waveguide iris: Good X-band, can be fabricated

**Practical Budget Filters:**
- SMAf (Surface Mount) low-pass/bandpass from Mini-Circuits: $5-20
- DIY microstrip filters: Cost of PCB material only
- Sawtek/Epcos bandpass for IF stages: $10-30

---

## 4. Synthesizers: LMX2572 vs ADF4351 vs SI5351

### 4.1 TI LMX2572

**Specifications:**
- Frequency range: 0.1 to 6.4 GHz (fundamental)
- Output power: +7 dBm
- Phase noise: -228 dBc/Hz (PLL)
- Resolution: 0.09 Hz (64-bit divider)
- Reference input: 50 MHz TCXO typical
- Power: 350 mW
- Package: 32-pin QFN

**Pros:**
- Excellent phase noise performance
- Low spurious emissions
- Wide frequency range
- SPI/I2C programming
- Frequency ramping capability (FMCW ready)

**Cons:**
- Upper limit 6.4 GHz (needs harmonic generation for X-band)
- Requires external VCO (integrated in newer LMX259x)
- More complex programming

**Use for SAR:** Good choice for L-band transmit, needs multiplier for X-band LO.

**Cost:** ~$15 (single)

---

### 4.2 Analog Devices ADF4351

**Specifications:**
- Frequency range: 0.22 to 4.4 GHz (fundamental)
- Output power: -4 to +5 dBm
- Phase noise: -220 dBc/Hz (PLL)
- Resolution: 0.1 Hz (24-bit divider)
- Integrated VCO
- Power: 300 mW
- Package: 32-lead LFCSP

**Pros:**
- Integrated VCO (smaller BOM)
- Proven, well-documented
- Good community support
- Reference for many DIY projects

**Cons:**
- Limited to 4.4 GHz (needs multiplier for X-band LO)
- Higher phase noise floor than LMX2572
- Some spurious issues reported

**Use for SAR:** Popular for SDR transceivers, good for IF generation, needs multiplication for X-band.

**Cost:** ~$12 (single)

---

### 4.3 Silicon Labs SI5351

**Specifications:**
- Frequency range: 8 kHz to 200 MHz (via dividers)
- Output power: +3 to +6 dBm
- Phase noise: -140 dBc/Hz @ 10 kHz (typical)
- Resolution: 1 Hz (28-bit divider)
- Three independent outputs
- Power: 100 mW
- Package: 10-pin MSOP/QFN

**Pros:**
- Extremely low cost (~$3)
- I2C programmable (Arduino/STM32 friendly)
- Three independent outputs
- Very flexible
- Excellent availability

**Cons:**
- Not a true RF synthesizer - outputs are square waves
- Limited to 200 MHz directly
- High harmonic content (square wave = rich harmonics)
- Poor phase noise vs dedicated RF PLLs
- Clock jitter can be problematic for sensitive receivers

**Use for SAR:** Can generate IF/local reference signals but NOT suitable for LO chain at X-band. Good for clocking ADCs and digital logic.

**Cost:** ~$3

---

### 4.4 Comparison Summary

| Parameter | LMX2572 | ADF4351 | SI5351 |
|-----------|---------|---------|--------|
| Max Frequency | 6.4 GHz | 4.4 GHz | 200 MHz |
| Phase Noise | Best | Good | Poor |
| Integrated VCO | No | Yes | N/A |
| Output Power | +7 dBm | +5 dBm | +6 dBm |
| Programming | SPI | SPI | I2C |
| Cost | ~$15 | ~$12 | ~$3 |
| X-band Ready | No* | No* | No |

*Requires frequency multiplication (×2, ×4) and filtering

### Recommendation

For X-band SAR under $200:
1. **Primary LO:** Use ADF4351 + ×2 frequency multiplier + cavity/SAW filter
2. **Alternative:** LMX2572 + ×2 multiplier + filter
3. **SI5351:** Use for ADC/DAC clocks and low-frequency IF generation only

---

## 5. ADC Requirements and STM32F407 Viability

### 5.1 ADC Requirements for SAR Radar

**Key ADC Parameters:**

| Parameter | Requirement | Why |
|-----------|-------------|-----|
| Sample Rate | ≥2× bandwidth (complex) | Nyquist for baseband |
| Resolution | ≥10 bits ENOB | Dynamic range for imaging |
| Input Bandwidth | ≥200 MHz IF | Must handle IF frequency |
| SFDR | >60 dB | Spurious-free dynamic range |
| ENOB @ input freq | ≥10 bits | Real-world performance |

**Bandwidth Calculation for FMCW SAR:**

```
Required Bandwidth = c / (2 × range_resolution)
Example: 0.5m resolution → 300 MHz bandwidth
Example: 0.1m resolution → 1.5 GHz bandwidth
```

For 0.5m range resolution (reasonable DIY target): ~300 MHz IF bandwidth needed.

### 5.2 STM32F407 ADC Analysis

**STM32F407VG Specifications:**
- Core: ARM Cortex-M4, 168 MHz
- ADCs: 12-bit, 2.4 MSPS (up to 7.2 MSPS in interleaved mode)
- Input channels: 16 single-ended
- Analog supply: 2-3.6V
- Input impedance: Programmable (via PGA)
- DMA: Yes, multiple channels

**Viability Assessment:**

| Requirement | STM32F407 | Verdict |
|-------------|-----------|---------|
| Sample Rate | 2.4 MSPS (12-bit) | **INSUFFICIENT** for direct IF sampling |
| Bandwidth | ~1 MHz analog (limited by SAR) | **INSUFFICIENT** |
| Resolution | 12-bit (advertised) | Good, but ENOB at speed? |
| DMA | Yes | Good for data transfer |
| Processing | 168 MHz DSP instructions | Good for DSP |
| External interface | Limited to parallel/FSMC | Could interface external ADC |

**Critical Problems:**
1. **Sample rate too low:** 2.4 MSPS cannot digitize 300 MHz IF
2. **Input bandwidth too low:** SAR architecture limits input frequency
3. **No external ADC bus:** Cannot easily interface high-speed ADC

### 5.3 Viable Alternatives for STM32F407

**Option A: STM32F407 as Controller Only**
- Use STM32F407 to control external high-speed ADC via SPI/JESD204
- Delegate all signal processing to external hardware or PC
- STM32 handles: LO synthesis, timing, data streaming to PC
- Cost: +$20-50 for external ADC

**Option B: Dedicated High-Speed ADC Boards**

| ADC Board | Specs | Cost | Interface |
|-----------|-------|------|-----------|
| AD9200 | 20 MSPS, 10-bit | ~$30 | Parallel |
| ADS54J20 | 80 MSPS, 14-bit | ~$80 | LVDS/JESD204 |
| AD9467 | 105 MSPS, 16-bit | ~$150 | LVDS |
| RTL-SDR | 3.2 MSPS, 8-bit | ~$20 | USB |

**Option C: SDR Platform (See Section 6)**

### 5.4 Recommendation

**STM32F407 CANNOT directly sample IF for SAR.**

Recommended architecture:
1. **STM32F407 as system controller** - handles PLL programming, timing, GPIO, USB/UART data streaming
2. **External ADC** - either discrete (ADS54J20) or SDR platform
3. **PC-based processing** - handles FFT, SAR imaging algorithms

**Viable budget approach:**
- RTL-SDR or PlutoSDR for initial experiments (~$20-150)
- STM32F407 controls the SDR via USB or SPI
- PC handles imaging

---

## 6. SDR Platform Evaluation

### 6.1 Analog Devices PlutoSDR (ADALM-PLUTO)

**Specifications:**
- Frequency: 325 MHz to 3.8 GHz (RX), 325 MHz to 6 GHz (TX)
- ADC/DAC: 12-bit, 61.44 MSPS
- Bandwidth: 20 MHz (adjustable up to 56 MHz)
- Interface: USB 3.0, also standalone ARM Linux
- TX Output: +3 dBm
- Size: USB stick form factor

**Pros:**
- Self-contained RF platform with TX and RX
- 12-bit ADC provides ~72 dB dynamic range (theoretical)
- Firmware can be modified (iio, libad9361)
- Integrated ARM processor (Zynq) runs Linux
- Can operate standalone with proper firmware
- **Excellent driver support** (Python, MATLAB, GNU Radio)

**Cons:**
- Frequency limit 3.8 GHz (no X-band without external upconversion)
- 20 MHz bandwidth may limit range resolution
- USB 3.0 required for full performance
- Some phase noise concerns for high-precision SAR

**Cost:** ~$150 (single), sometimes found for $100

**SAR Viability:** Good for L-band and S-band SAR experiments. Can be used as IF digitizer with external mixer for X-band.

---

### 6.2 RTL-SDR (R820T/RTL2832)

**Specifications:**
- Frequency: 500 kHz to 1.7 GHz (tunable to 110 MHz-1 GHz typical)
- ADC: 8-bit, 3.2 MSPS (software defined)
- Bandwidth: ~2.4 MHz usable
- Interface: USB 2.0
- RX Noise Figure: ~3-5 dB (varies)

**Pros:**
- Extremely low cost (~$20)
- Massive community support
- Excellent for learning and prototyping
- Works with every SDR software

**Cons:**
- 8-bit resolution limits dynamic range to ~50 dB
- 2.4 MHz bandwidth severely limits range resolution
- No transmit capability
- High noise figure with typical LNA
- Not suitable for serious SAR

**Cost:** ~$20

**SAR Viability:** Only suitable for the most basic experiments. 8-bit and narrow bandwidth make real SAR imaging difficult.

---

### 6.3 HackRF One

**Specifications:**
- Frequency: 1 MHz to 6 GHz
- ADC/DAC: 8-bit, 20 MSPS
- Bandwidth: 20 MHz
- Interface: USB 2.0
- TX Output: +3 to +15 dBm (variable)
- Half-duplex (TX or RX, not simultaneous)

**Pros:**
- TX and RX capability
- Wide frequency range
- Open source hardware/software
- Good for prototyping
- Reasonable cost

**Cons:**
- 8-bit resolution (same as RTL-SDR)
- Half-duplex (cannot transmit and receive simultaneously)
- 20 MHz bandwidth
- Phase noise can be an issue
- No onboard preamplifier

**Cost:** ~$300 (new), sometimes $200 used

**SAR Viability:** Better than RTL-SDR due to TX and bandwidth, but 8-bit limits quality.

---

### 6.4 Comparison Table

| Platform | Cost | RX Bits | Bandwidth | TX | Freq Max | ENOB Est |
|----------|------|---------|-----------|-----|----------|----------|
| RTL-SDR | $20 | 8 | 2.4 MHz | No | 1.7 GHz | ~6-7 |
| PlutoSDR | $150 | 12 | 20-56 MHz | Yes | 6 GHz | ~10-11 |
| HackRF | $300 | 8 | 20 MHz | Yes | 6 GHz | ~6-7 |
| USRP B200 | $700 | 12 | 30 MHz | Yes | 6 GHz | ~10-11 |

### 6.5 Recommendation

**For DIY SAR under $200:**

1. **Best option: Buy used PlutoSDR (~$100-130)**
   - 12-bit resolution for quality imaging
   - TX capability for FMCW
   - Can be controlled by STM32F407 via libiio
   - Extend frequency with external mixer for X-band

2. **Alternative: RTL-SDR for learning (~$20)**
   - Start here to learn SDR basics
   - Build algorithms, test processing chain
   - Very limited but educational

3. **Avoid HackRF** - 8-bit and half-duplex make it poor value at $300.

---

## 7. ISM and X-Band Regulatory Considerations

### 7.1 ISM Bands (Industrial, Scientific, Medical)

**Key ISM Bands for Radar:**

| Band | Frequency | License | Typical Radar Use |
|------|-----------|---------|-------------------|
| L-band | 1-2 GHz | No (with power limits) | Air traffic, weather |
| S-band | 2-4 GHz | No (with power limits) | Airport surveillance |
| C-band | 4-8 GHz | No (with power limits) | Military, SAR (Sentinel) |
| X-band | 8-12 GHz | No (with power limits) | Military, maritime |
| Ku-band | 12-18 GHz | No (with power limits) | SAR (TerraSAR) |
| K-band | 18-27 GHz | No (with power limits) | Automotive radar |

**ISM Power Limits (FCC Part 18 / ETSI EN 301 489):**
- Intentional radiators: Typically 1W ERP for non-licensed use
- Certain bands allow higher with specific modulation
- Must use spread spectrum techniques in some bands

### 7.2 X-Band Radar Regulations

**X-band: 8-12 GHz (specifically 8-12.5 GHz for radar)**

**United States (FCC):**
- Part 15 (unlicensed): 1W max ERP for FMCW/impulse
- Part 18 (ISM): Up to 1W for industrial heating equipment (not radar)
- Amateur radio: 10 GHz amateur band at 10-10.5 GHz, requires license
- Part 23 (commercial): Requires license for most radar applications

**Europe (ETSI):**
- EN 301 489: Generic EMC standard
- EN 302 065: Short Range Devices (SRD) - various power levels
- 10 GHz band: 25 mW EIRP typical for SRD

**United Kingdom (Ofcom):**
- Similar to ETSI framework
- 10 GHz amateur band: 10-10.5 GHz, license required
- SRD standards apply for <10 GHz

### 7.3 Practical DIY Legal Boundaries

**Safe Operating Zones for DIY SAR:**

| Scenario | Frequency | Max Power | Status |
|----------|-----------|-----------|--------|
| RTL-SDR receive only | Any | N/A | Always legal |
| Low-power TX | 2.4-2.5 GHz | <100mW | Legal (Wi-Fi band) |
| Low-power TX | 5.8 GHz | <100mW | Legal (Wi-Fi band) |
| X-band TX | 10.5 GHz | <25mW | Legal (amateur with license) |
| X-band TX | 9-10 GHz | <10mW | Questionable (SRD) |

**Key Legal Considerations:**
1. **Receive-only is always legal** - no license needed for listening
2. **Transmit requires consideration** - higher power = more regulation
3. **Amateur license** (Technician/Class A) enables 10 GHz amateur band
4. **Ham radio laws** vary by country - check local regulations
5. **Experimental licenses** available in some jurisdictions

### 7.4 Recommendation for Legal DIY SAR

**Safest approach:**
1. **Start receive-only** - legal with any SDR
2. **Use L-band or S-band** for transmit experiments (more leeway)
3. **If X-band required** - use external service or obtain amateur license
4. **Keep power low** - <10 mW for initial experiments
5. **Use ISM-allocated bands** where possible

**For serious X-band SAR:**
- Consider using aircraft radar band (ATCRBS) receivers: 1090 MHz
- Or subscribe to commercial satellite SAR data (ICEYE, Capella)
- Build X-band receive-only and rely on反射 of opportunity signals

---

## 8. Front-End Block Diagram Under $200

### 8.1 Budget Architecture: Heterodyne FMCW

**Target:** X-band (9-10 GHz), 0.5m range resolution, 100m range

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DIY SAR RF FRONT-END ($150-180)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TRANSMIT CHAIN:                           RECEIVE CHAIN:                     │
│  ───────────────                           ──────────────                     │
│                                                                              │
│  ┌──────────────┐     ┌──────────┐      ┌─────────────┐    ┌──────────────┐ │
│  │ Chirp Source │────▶│  ×4      │─────▶│ Bandpass    │───▶│ PA          │ │
│  │ ADF4351 +    │     │ Multiplier│     │ Filter      │    │ (20 dBm)    │ │
│  │ STM32 Ctrl   │     │ + Filter │     │ (9-10 GHz)  │    │             │ │
│  └──────────────┘     └──────────┘      └─────────────┘    └──────┬───────┘ │
│                           │                                       │         │
│                           │          ┌─────────────┐             │         │
│                           └─────────▶│ Directional │◀────────────┘         │
│                                       │ Coupler     │                      │
│                                       │ (-20 dB)    │                      │
│                                       └──────┬──────┘                      │
│                                              │                              │
│                                       ┌──────▼──────┐                      │
│                                       │ Antenna     │                      │
│                                       │ (Patch or   │                      │
│                                       │ waveguide)  │                      │
│                                       └──────┬──────┘                      │
│                                              │                              │
│                                       ┌──────▼──────┐                      │
│                                       │ LNA         │                      │
│                                       │ (X-band,   │                      │
│                                       │ NF<2 dB)   │                      │
│                                       └──────┬──────┘                      │
│                                              │                              │
│                                       ┌──────▼──────┐                      │
│                                       │ Mixer       │◀──── LO IN            │
│                                       │ (Image reject│    (same source)     │
│                                       │  type)      │                      │
│                                       └──────┬──────┘                      │
│                                              │                              │
│                                       ┌──────▼──────┐                      │
│                                       │ IF Amplifier│                      │
│                                       │ (bandpass   │                      │
│                                       │  100 MHz)   │                      │
│                                       └──────┬──────┘                      │
│                                              │                              │
│                                       ┌──────▼──────┐                      │
│                                       │ ADC / SDR   │                      │
│                                       │ (Pluto or   │                      │
│                                       │  AD9200)    │                      │
│                                       └──────┬──────┘                      │
│                                              │                              │
│                                       ┌──────▼──────┐                      │
│                                       │ STM32F407   │                      │
│                                       │ (Control & │                      │
│                                       │  Streaming)│                      │
│                                       └─────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Detailed Budget Breakdown

| Component | Part / Source | Cost | Notes |
|-----------|--------------|------|-------|
| **Synthesizer** | ADF4351 breakout board | $12 | Programmed by STM32 |
| **×4 Multiplier** | Mini-Circuits LCM-84+ or DIY | $15 | Critical - needs filtering |
| **Bandpass Filter** | DIY microstrip or cavity | $20 | Rejects multiplier spurs |
| **LNA** | HMC1041 or BGA427 | $25 | X-band low noise |
| **Mixer** | ADE-30+ or Mini-Circuits | $10 | Diode ring type |
| **IF Amplifier** | ADL5535 or cascaded MAR-8 | $10 | 100 MHz IF stage |
| **PA (Optional)** | TQP3M9009 or similar | $15 | If >10mW needed |
| **Directional Coupler** | Mini-Circuits or DIY | $10 | -20 dB coupling |
| **ADC/SDR** | PlutoSDR (used) or AD9200 | $50-130 | Main cost variable |
| **STM32F407** | Dev board (black pill) | $10 | Already have |
| **Power Supply** | Bench supply or LM317 | $5 | 5V, 12V rails |
| **Cables/Connectors** | SMA, adapters | $15 | Don't cheap out here |
| **Misc (PCB, etc)** | | $10 | |
| | | | |
| **TOTAL** | | **$147-227** | Depending on ADC choice |

### 8.3 Component Details

**Chirp Source (ADF4351 + ×4 Multiplier):**
```
Frequency range needed: 9-10 GHz
ADF4351 output: 2.25-2.5 GHz (×4 = 9-10 GHz)
Chirp bandwidth: 300 MHz (for 0.5m resolution)
Chirp rate: Programmable via STM32
```

**Multiplier Critical Note:** 
Frequency multipliers generate rich harmonic content. The ×4 stage MUST be followed by a bandpass filter to remove spurs. Without proper filtering, spurs will appear in the IF and corrupt radar data.

**Alternative: Use Two AD4361s:**
- One for TX LO, one for RX LO with offset
- More expensive but cleaner

### 8.4 Simplified Budget Option ($80-100)

If ADC/SDR cost is reduced by using RTL-SDR for receive-only experiments:

| Component | Cost |
|-----------|------|
| ADF4351 breakout | $12 |
| ×4 multiplier + filter | $25 |
| LNA | $20 |
| Mixer | $10 |
| IF amp + filter | $10 |
| RTL-SDR (receive only) | $20 |
| STM32F407 | $10 (already have) |
| Misc | $10 |
| **TOTAL** | **~$90-100** |

**Limitation:** Transmit only, no FMCW. Use reflected signals from existing X-band sources (radar, satellite).

---

## 9. Minimum Viable Front-End

### 9.1 Absolute Minimum for SAR Experiments

**Receive-Only Passive SAR (~$50):**
1. RTL-SDR or PlutoSDR - $20-50
2. X-band LNA - $25
3. Standard gain X-band antenna (patch or conical) - $10
4. STM32F407 for control - $10 (already have)

**This can:**
- Detect large metal objects (vehicles, buildings)
- Map coastline/shoreline using existing X-band illumination
- Receive satellite radar signals
- Learn SAR processing techniques

### 9.2 Basic Active SAR Front-End (~$150)

**Components:**
1. ADF4351 synthesizer - $12
2. ×4 multiplier with filter - $25
3. PA (10-20 dBm) - $15
4. Directional coupler - $10
5. X-band antenna - $15
6. LNA - $25
7. Mixer - $10
8. PlutoSDR for IF capture - $50 (used)
9. STM32F407 control - $10
10. Cables/connectors - $15

**This can:**
- Generate FMCW chirps at X-band
- Receive and digitize returns
- Basic SAR imaging up to ~100m range
- 0.5-1m resolution

### 9.3 Recommended Build Sequence

**Phase 1: Receive-Only (Week 1-2, ~$50)**
- Connect antenna → LNA → RTL-SDR
- Verify X-band reception
- Capture and analyze existing signals
- Test STM32 control of RTL-SDR

**Phase 2: Transmit Source (Week 3-4, ~$50)**
- Build ADF4351 + multiplier chain
- Add bandpass filter
- Verify X-band output with spectrum analyzer (or RTL-SDR as detector)
- Test chirp generation

**Phase 3: Full FMCW (Week 5-6, ~$50)**
- Add mixer and IF amplifier
- Connect PlutoSDR as IF digitizer
- Full system integration
- Initial range measurements

**Phase 4: Imaging (Week 7+, software)**
- PC-based signal processing
- SAR algorithms (FFT, backprojection)
- Antenna motion/synthetic aperture

---

## 10. Key Recommendations Summary

### Architecture
- **Use heterodyne** with FMCW modulation
- X-band (9-10 GHz) for good resolution with modest antenna size
- IF sampling at 100-300 MHz using PlutoSDR or external ADC

### Critical Components (Don't Skimp)
1. **LNA** - determines sensitivity, use quality X-band amp (~$25)
2. **LO chain** - ADF4351 + multiplier + filtering (~$40)
3. **ADC** - PlutoSDR for best cost/performance (~$100 used)

### Budget Allocation Priority
1. ADC/SDR: $50-130 (most important)
2. LNA + Mixer: $35-40
3. Synthesizer + multiplier: $40
4. STM32F407: $10 (already available)

### STM32F407 Role
- **NOT** an ADC replacement
- Use as system controller: PLL programming, timing, data streaming
- Interface to PlutoSDR via USB or to external ADC via SPI
- Handle chirp generation control signals

### Regulatory Compliance
- Start with receive-only experiments (always legal)
- Keep transmit power low (<10 mW) for initial tests
- Consider L-band/S-band for easier legal compliance
- Get amateur radio license for 10 GHz band access

### Testing & Verification
- Use spectrum analyzer if available
- Use second RTL-SDR or signal detector for X-band verification
- Start with simple range measurements before SAR imaging
- Document all components and configurations

---

## References and Further Reading

**Component Datasheets:**
- TI LMX2572: www.ti.com/lit/ds/symlink/lmx2572.pdf
- AD ADF4351: www.analog.com/en/products/adf4351.html
- AD AD9200: www.analog.com/en/products/ad9200.html
- AD ADL5811: www.analog.com/en/products/adl5811.html

**Open Source SAR Projects:**
- MIT SAR (various academic projects)
- GNU Radio radar examples
- Open-Project-SAR (if exists)

**SDR Resources:**
- IIO (Industrial I/O) framework for PlutoSDR
- librtlsdr for RTL-SDR
- HackRF documentation

---

*Document compiled: April 29, 2026*
*Next: Antenna design, Signal processing algorithms, Mechanical scanning system*

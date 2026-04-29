# DIY SAR Antenna Design Research

**Date:** April 29, 2026  
**Project:** DIY Synthetic Aperture Radar  
**Hardware Context:** STM32F407 based signal processing, very limited budget  
**Target:** On-ground or drone-mounted SAR

---

## 1. Frequency Band Selection: X vs C vs Ku

### 1.1 Band Overview

| Band | Frequency Range | Wavelength (center) | Atmospheric Loss | Penetration |
|------|-----------------|---------------------|------------------|-------------|
| **X** | 8–12 GHz | 3.75 cm (10 GHz) | ~0.02 dB/km (clear air) | Moderate (foliage, light rain) |
| **C** | 4–8 GHz | 5 cm (6 GHz) | ~0.01 dB/km | Good (vegetation, dry soil) |
| **Ku** | 12–18 GHz | 2 cm (15 GHz) | ~0.05 dB/km | Poor (heavy rain attenuation) |

### 1.2 DIY Feasibility by Band

#### X-Band (8–12 GHz)
- **Pros:**
  - Mature technology, abundant cheap components (horizontally-polarized patches, WR-90 waveguides)
  - Good resolution at reasonable antenna sizes (λ ≈ 3 cm)
  - Reasonable PCB trace tolerances: ±2–3 mil at this frequency
  - Mature radar modules available (e.g., K-band transceivers can be pulled from automotive radar)
- **Cons:**
  - Smaller wavelength = tighter manufacturing tolerances for microstrip
  - Moderate atmospheric attenuation in rain
- **Typical ISM/use:** 10 GHz is not ISM, but 9.6 GHz is used in some short-range radar modules
- **Component availability:** WR-90 waveguides (X-band) readily available from surplus; patch antennas at 10 GHz are standard

#### C-Band (4–8 GHz)
- **Pros:**
  - Best penetration through vegetation and dry materials
  - Larger wavelength = more forgiving PCB tolerances (±5 mil acceptable)
  - Good for soil moisture applications
  - Less rain attenuation than Ku
- **Cons:**
  - Larger antenna size for equivalent gain vs X or Ku
  - Some frequencies (5.8 GHz) are ISM; 6 GHz is becoming available for SAR
- **Typical ISM/use:** 5.8 GHz is ISM band; 6.0–6.5 GHz used for some satellite comms

#### Ku-Band (12–18 GHz)
- **Pros:**
  - Highest resolution potential (smallest λ)
  - Smallest antenna for given gain
  - Excellent for imaging detailed structures
- **Cons:**
  - High atmospheric/rain attenuation (significant in heavy rain)
  - Tightest PCB tolerances (±1 mil or better)
  - More expensive components generally
  - WR-62 waveguides less common than WR-90

### 1.3 Recommendation for DIY SAR

**Primary: X-Band (9–10 GHz)** for most hobbyist/DIY builds.

**Reasoning:**
- Best balance of size, resolution, component availability, and manufacturing tolerance
- WR-90 waveguide and standard 10 GHz patch arrays are commercially available at low cost
- Resolution formula: `Range Resolution = c / (2 × Bandwidth)` — at 100 MHz bandwidth, ~1.5 m range resolution; X-band allows fine azimuth resolution via SAR processing
- Can achieve `Azimuth Resolution = D / 2` where D is antenna physical dimension — a 30 cm antenna gives 15 cm azimuth resolution

**Secondary: C-Band (5.6–6 GHz)** if penetration through vegetation or soil is priority.

---

## 2. Antenna Type Trade-offs

### 2.1 Overview of Candidate Antenna Types

| Antenna Type | Typical Gain | Beamwidth (H/V) | Profile | Cost | Hobbyist Build |
|--------------|-------------|-----------------|---------|------|----------------|
| **Microstrip Patch Array** | 20–30 dBi | 10°–30° / 10°–30° | Low (flush) | $15–80 | Moderate (PCB) |
| **Waveguide (Rectangular)** | 10–20 dBi | 40°–80° / 20°–40° | Medium | $20–60 | Easy (surplus) |
| **Helical** | 10–17 dBi | 30°–50° (circular) | Medium-High | $10–40 | Easy |
| **Parabolic Dish** | 20–35 dBi | 5°–20° | High | $30–200 | Moderate |
| **Vivaldi** | 6–15 dBi | 60°–90° | Medium | $20–100 | Moderate (PCB) |

### 2.2 Microstrip Patch Array

**Description:** Series-fed or corporate-fed array of resonant patches on PCB substrate.

**Pros:**
- Low profile, conformal to mounting surface
- Flexible beam shaping via array geometry
- Fully PCB-fabricated (no mechanical assembly)
- Can be dual-polarization with two feeds per element
- Consistent phase center for SAR processing

**Cons:**
- Narrow bandwidth: typically 3–5% (10 dB return loss)
- Efficiency reduced by dielectric and conductor losses at higher frequencies
- Corporate feed networks have loss (0.5–2 dB for large arrays)
- Mutual coupling between elements affects pattern

**Typical Specifications (4×4 patch array at 10 GHz):**
```
Center Frequency: 9.6 GHz
Number of Elements: 16 (4×4)
Substrate: Rogers RT/duroid 5880 (εr = 2.2), 31 mil thick
Element Spacing: 0.7λ ≈ 22 mm
Array Aperture: ~90 mm × 90 mm
Gain: 18–22 dBi
Beamwidth (E-plane): ~25°
Beamwidth (H-plane): ~25°
Bandwidth (S11 < -10 dB): ~400 MHz (4%)
Feed: Corporate (Wilkinson divider network)
```

**Buildability Notes:**
- Requires PCB fabrication (JLCPCB, PCBWay — $5–20 for 5 boards)
- Gerber files can be generated from array synthesis tools
- Rogers板材 is expensive ($50–100 for 6"×6"); FR-4 is usable but higher loss
- Patch alignment is critical: ±0.5 mm tolerance needed at 10 GHz

**Cost Estimate:**
- Rogers RT/duroid 5880 4×4 array prototype: $60–120 (including PCB fabrication)
- FR-4 equivalent: $15–40

### 2.3 Rectangular Waveguide

**Description:** Standard WR-90 (X-band) or WR-62 (Ku-band) waveguide with probe or slot coupling.

**Pros:**
- Very low loss (virtually no dielectric loss)
- Wide bandwidth: typically 10–15% (WR-90: 8.2–12.4 GHz)
- High power handling
- Simple to build from surplus parts
- No dielectric constant variation issues

**Cons:**
- Bulkier than patch arrays
- Fixed beamwidth based on waveguide dimension
- Mechanical tuning required
- Standard rectangular waveguides have moderate beamwidth

**Typical Specifications (WR-90 open-ended waveguide at 10 GHz):**
```
Center Frequency: 10 GHz
Waveguide: WR-90 (inner: 22.86 mm × 10.16 mm)
Aperture: 22.86 mm × 10.16 mm
Gain: ~10 dBi (open end)
Beamwidth (E-plane): ~60°
Beamwidth (H-plane): ~90°
Bandwidth: 8.2–12.4 GHz
```

**Waveguide Antenna with Stepped Septum (X-band):**
```
Aperture: 60 mm × 40 mm
Gain: 15–17 dBi
Beamwidth (E-plane): ~30°
Beamwidth (H-plane): ~40°
Bandwidth: ~1 GHz
```

**Buildability Notes:**
- WR-90 waveguide sections and flanges available on eBay/SurplusSales for $5–20
- Can be fabricated from aluminum bar stock with basic machining
- Simple probe feed from N-type or SMA connector
- E-plane sectoral horns can be built from sheet metal

**Cost Estimate:**
- Surplus WR-90 waveguide sections (6"–12"): $5–15
- N-to-WR-90 waveguide adapter: $10–25
- DIY sheet metal E-plane horn: $5–15 (materials only)
- Total: $20–55

### 2.4 Helical Antenna

**Description:** Axial-mode helical antenna with 10–20 turns, typically 0.5–1.0 λ diameter.

**Pros:**
- Circularly polarized (excellent for SAR — reduces specular reflections)
- Simple construction from coax and template
- Wideband: typical 10–15% bandwidth
- No ground plane required (unlike patch)
- Consistent phase center

**Cons:**
- Moderate gain per turn; need many turns for high gain
- Narrow axial beamwidth (wider at low frequencies)
- Can be bulky lengthwise
- Mechanical stability matters (wind loading on drone)

**Typical Specifications (Axial-mode helix at 10 GHz):**
```
Center Frequency: 9.6 GHz
Number of Turns: 12
Helix Diameter: 30 mm (≈1.0 λ)
Turn Spacing: 10 mm (≈0.33 λ)
Total Length: ~120 mm
Gain: 12–15 dBi
Beamwidth (3 dB): ~35°
Polarization: RHCP or LHCP
Bandwidth: 9.0–10.5 GHz typical
```

**Buildability Notes:**
- Can be built from copper wire (18–20 AWG) wrapped around a cylindrical form
- Ground plane screen recommended at base
- Simple coax probe feed with gamma match
- Mechanical rigidity critical for drone mounting (use PVC pipe form with epoxy)

**Cost Estimate:**
- Copper wire (50 ft 18 AWG): $10–15
- Coaxial cable (RG-402): $5–10
- N-type connector: $5–15
- Form (PVC pipe): $5
- Total: $25–45

### 2.5 Parabolic Dish

**Description:** Reflector antenna with parabolic shape, fed by waveguide probe or small feed antenna.

**Pros:**
- Highest gain for given aperture size
- Very well understood design
- Can achieve very fine resolution
- Feed is outside the main beam path

**Cons:**
- Larger profile/depth than patch or waveguide
- Requires precise reflector shaping
- Mechanical alignment critical
- Wind loading significant on drone

**Typical Specifications (18" dish at 10 GHz):**
```
Center Frequency: 10 GHz
Dish Diameter: 457 mm (18 inches)
F/D Ratio: 0.6
Gain: ~32 dBi
Beamwidth (3 dB): ~5°
Aperture Efficiency: 55–65%
```

**Buildability Notes:**
- Off-axis parabolic dishes available as surplus (TVRO, satellite)
- DIY from aluminum sheet: accurate parabola cutting is difficult without CNC
- Wire mesh reflectors work but efficiency is reduced (~50%)
- Offset feed geometry reduces feed blockage

**Cost Estimate:**
- Surplus 18" dish: $15–40
- Feed waveguide assembly: $20–40
- Mount hardware: $10–30
- Total: $45–110

### 2.6 Comparison Summary

| Criterion | Patch Array | Waveguide | Helical | Parabolic |
|-----------|------------|-----------|---------|-----------|
| **Gain** | ★★★★ | ★★★ | ★★★ | ★★★★★ |
| **Bandwidth** | ★★ | ★★★★ | ★★★★ | ★★★ |
| **Low Profile** | ★★★★★ | ★★★ | ★★ | ★ |
| **Buildability** | ★★★ | ★★★★ | ★★★★ | ★★ |
| **Cost** | ★★★ | ★★★★ | ★★★★★ | ★★ |
| **Drone Friendly** | ★★★★★ | ★★★★ | ★★★ | ★★ |
| **SAR Phase Center** | ★★★★★ | ★★★★ | ★★★★ | ★★★★ |

---

## 3. Physical Size and Aperture Requirements

### 3.1 Fundamental Relationships

**Gain Equation (for aperture antennas):**
```
G = (4π × η × A) / λ²

Where:
  G = gain (linear, not dB)
  η = aperture efficiency (typically 0.55–0.70)
  A = physical aperture area (m²)
  λ = wavelength (m)
```

**Converting to dB:**
```
G(dBi) = 10 × log10(G)
```

**Antenna beamwidth approximation (for circular aperture):**
```
HPBW ≈ 58 × (λ / D)  [degrees]

Where:
  D = antenna diameter (for circular aperture)
  λ = wavelength
```

**For rectangular array:**
```
HPBW_H ≈ 50 × (λ / L_H)  [degrees]
HPBW_E ≈ 50 × (λ / L_E)  [degrees]

Where:
  L_H = horizontal dimension
  L_E = elevation dimension
```

### 3.2 Size Requirements by Resolution Target

SAR azimuth resolution is fundamentally tied to the **physical antenna dimension**:

```
Azimuth Resolution (single-pass) = D / 2

Where:
  D = antenna along-track dimension (m)
```

This is a hard physical limit for real-aperture systems. SAR processing doesn't overcome this without multiple passes or motion compensation.

**Required Antenna Size for Target Resolution:**

| Target Azimuth Resolution | Required D (λ units) | X-Band (3.75 cm) | C-Band (5 cm) |
|--------------------------|---------------------|-----------------|---------------|
| 0.5 m | 1.0 m | 27 λ | 20 λ |
| 0.25 m | 0.5 m | 13.3 λ | 10 λ |
| 0.1 m | 0.2 m | 5.3 λ | 4 λ |

**Example:** At X-band (10 GHz), a 0.5 m antenna gives 0.25 m single-pass azimuth resolution.

### 3.3 Aperture Dimensions for Target Gain

Solving the gain equation for aperture area:

```
A = (G × λ²) / (4π × η)

Required aperture area for target gain (at 10 GHz, η = 0.65):
```

| Gain (dBi) | Linear Gain | Aperture Area (cm²) | Approx. Dimensions |
|------------|-------------|---------------------|-------------------|
| 15 | 31.6 | 28 cm² | 53 mm × 53 mm (square) |
| 20 | 100 | 89 cm² | 94 mm × 94 mm |
| 25 | 316 | 283 cm² | 168 mm × 168 mm |
| 30 | 1000 | 893 cm² | 299 mm × 299 mm |
| 35 | 3162 | 2827 cm² | 531 mm × 531 mm |

### 3.4 Drone Payload Constraints

**Small Drone (DJI Mavic class, ~900g MTOW):**
```
Payload capacity: 100–200g
Typical battery life with payload: 15–25 minutes
Flat mounting area: ~10 cm × 10 cm maximum
Wind sensitivity: high
Recommended antenna: Microstrip patch array (< 100g, low profile)
```

**Medium Drone (DJI Matrice 100 class, ~4 kg MTOW):**
```
Payload capacity: 800–1500g
Typical battery life with payload: 20–35 minutes
Flat mounting area: 20 cm × 20 cm
Wind sensitivity: moderate
Recommended antenna: Patch array or small waveguide/horn (< 500g)
```

**Heavy Lift Drone (custom hex/octo, >10 kg MTOW):**
```
Payload capacity: 3–8 kg
Typical battery life with payload: 25–60 minutes
Flat mounting area: 30 cm × 30 cm+
Wind sensitivity: low
Recommended antenna: Parabolic dish or large patch array (up to 2 kg)
```

**Drone-Specific Antenna Recommendations:**

| Drone Class | Recommended Antenna Type | Est. Weight | Est. Dimensions |
|-------------|-------------------------|-------------|-----------------|
| Mavic-class | 4×4 patch array | 80–120g | 100×100×15 mm |
| Matrice-class | 8×8 patch array or small horn | 300–500g | 200×200×30 mm |
| Heavy-lift | Parabolic 18–24" or large array | 800–2000g | 500–600mm diameter |

---

## 4. Bandwidth and Range Resolution

### 4.1 Range Resolution Fundamentals

**Range Resolution (in slant range):**
```
δR = c / (2 × B)

Where:
  δR = range resolution (m)
  c = speed of light (299,792,458 m/s)
  B = chirp bandwidth (Hz)
```

**Ground Range Resolution (accounting for depression angle):**
```
δRg = δR / sin(θ)

Where:
  θ = depression angle from horizontal
```

### 4.2 Bandwidth Requirements

| Target Range Resolution | Required Bandwidth (Single Channel) |
|------------------------|-------------------------------------|
| 3.0 m | 50 MHz |
| 1.5 m | 100 MHz |
| 0.75 m | 200 MHz |
| 0.30 m | 500 MHz |
| 0.15 m | 1000 MHz (1 GHz) |

**Note:** STM32F407 ADC is limited to ~2.4 MSPS at full resolution, which constrains IF bandwidth to approximately 1 MHz without undersampling tricks. For higher bandwidth SAR, external ADC or direct RF sampling with undersampling is needed.

**Bandwidth vs. Practical Considerations:**

| Bandwidth | Pros | Cons |
|-----------|------|------|
| 50 MHz | Easy to generate, simple hardware | 3 m resolution, large pixel size |
| 100 MHz | 1.5 m resolution, good for many targets | Moderate hardware complexity |
| 200 MHz | 0.75 m resolution, good imaging | Requires faster ADC or I/Q sampling |
| 500 MHz | 0.3 m resolution, high detail | Significant hardware and processing challenge |

### 4.3 FMCW vs. Pulse Radar Bandwidth

**FMCW (Frequency Modulated Continuous Wave):**
- Continuous transmission with linear frequency sweep
- Lower peak power needed
- Bandwidth = sweep range (which equals resolution)
- Typical for short-range SAR (< 1 km)
- Can achieve 1 GHz bandwidth with modern VCOs

**Pulse Radar:**
- High peak power pulses
- Bandwidth = inverse of pulse width
- Requires high-power amplifier
- Less common for hobbyist SAR

**For STM32F407-based DIY SAR:**
- FMCW is the practical choice (simpler hardware, lower power)
- Recommended: 100–200 MHz FMCW chirp at X-band
- ADF4351 or similar PLL + VCO for chirp generation

---

## 5. Gain and Beamwidth Specifications

### 5.1 System Link Budget Requirements

For SAR imaging, the antenna must provide sufficient gain to detect returns from the target area. Typical received signal power (simplified):

```
Pr = Pt + Gt + Gr + σ + Gproc - Lprop - Lsys

Where:
  Pr = received power (dBm)
  Pt = transmit power (dBm)
  Gt = transmit antenna gain (dBi)
  Gr = receive antenna gain (dBi)
  σ = radar cross section (dBsm)
  Gproc = SAR processing gain (dB) ≈ 10×log10(Npulses)
  Lprop = path loss (dB)
  Lsys = system losses (dB)
```

**Path Loss (free space):**
```
Lprop = 20×log10(4πR/λ)

At 10 GHz, 500 m range: Lprop ≈ 132 dB
```

### 5.2 Minimum Gain Requirements

For detectability of typical targets (σ ≈ 0 dBsm = 1 m²), with Pt = 20 dBm (100 mW), 100 MHz bandwidth, STM32F407 receiver noise floor ≈ -100 dBm:

```
Required Gr + Gt ≈ 20 + 2×Gr > 60 dB (for 10 dB SNR with processing gain)
→ Gr > 20 dBi each (identical transmit and receive antennas)
```

**SAR Processing Gain:**
```
Gproc = 10×log10(N) [dB]

Where N = number of pulses integrated

Typical values:
  N = 100 → Gproc = 20 dB
  N = 1000 → Gproc = 30 dB
  N = 10000 → Gproc = 40 dB
```

### 5.3 Antenna Pattern and SAR Imaging Quality

**Beamwidth Impact on Swath Width:**
```
Swath = 2 × H × tan(θ_beam/2)

Where:
  H = altitude above ground (m)
  θ_beam = beamwidth (radians)
```

**Cross-track Ground Coverage:**
- Narrow beamwidth = smaller swath per pass
- Wider beamwidth = larger swath but lower gain
- Trade-off based on drone altitude and desired coverage

**Recommended Beamwidths by Application:**

| Application | Altitude | Recommended 3 dB Beamwidth |
|-------------|----------|---------------------------|
| High-res mapping | 100–200 m | 15°–25° |
| Medium-res survey | 200–500 m | 25°–40° |
| Wide-area search | 500–1000 m | 40°–60° |

### 5.4 sidelobe Requirements

For SAR imaging, low sidelobes are important to avoid artifacts:

**Recommended sidelobe levels:**
- Primary sidelobes: < -20 dB relative to main beam
- Integrated sidelobe ratio (ISLR): < -13 dB

**Antenna type implications:**
- Taylor or Dolph-Chebyshev distribution for patch arrays: -25 to -35 dB sidelobes
- Uniform illumination: -13.2 dB (first sidelobe)
- Cosecant-squared patterns for wide swath: -18 to -20 dB

---

## 6. Cost Estimates

### 6.1 Component-Level Cost Breakdown

**X-Band (9–10 GHz) Antenna Solutions:**

| Component | Source | Cost Range |
|-----------|--------|------------|
| **Microstrip Patch Array (4×4, Rogers)** | PCB fabrication (JLCPCB) | $30–60 |
| Rogers RT/duroid 5880 (6"×6") | Digikey/Mouser | $60–100 |
| FR-4 patch array (4×4) | PCB fabrication | $15–25 |
| WR-90 waveguide (12") | SurplusSales/eBay | $8–20 |
| WR-90 E-plane sectoral horn (DIY) | Sheet aluminum | $10–20 |
| N-to-WR90 adapter | Pasternack/DFS | $15–30 |
| Helical antenna (DIY) | Copper wire + coax | $15–30 |
| 18" parabolic dish | Surplus (TVRO) | $20–40 |
| Parabolic feed assembly | DIY or surplus | $15–30 |

**Complete Antenna Solutions:**

| Antenna Type | Total Cost | Performance |
|--------------|------------|--------------|
| FR-4 4×4 patch array (9.6 GHz) | $20–35 | 18–20 dBi, 25° beamwidth |
| Rogers 4×4 patch array | $100–150 | 20–22 dBi, 20° beamwidth |
| WR-90 open waveguide + DIY horn | $30–50 | 12–15 dBi, 40–60° beamwidth |
| Helical (12-turn, 10 GHz) | $20–35 | 12–15 dBi, 35° beamwidth |
| 18" parabolic + feed | $50–80 | 30–33 dBi, 5–8° beamwidth |
| 24" parabolic + feed | $80–150 | 32–35 dBi, 4–6° beamwidth |

### 6.2 Full System Cost Estimates

**Budget Build (X-Band, 100 MHz BW, 1.5 m resolution):**

| Component | Estimated Cost |
|-----------|---------------|
| FR-4 patch array (4×4) | $25 |
| WR-90 waveguide sections | $15 |
| RF components (mixer, VCO, amplifier) | $50–80 |
| STM32F407 development board | $15–25 |
| Power dividers, cables, connectors | $20–40 |
| **Total Antenna + RF Front End** | **$125–185** |

**Mid-Range Build (X-Band, 200 MHz BW, 0.75 m resolution):**

| Component | Estimated Cost |
|-----------|---------------|
| Rogers 4×4 patch array | $120 |
| Low-noise amplifier | $30–50 |
| Higher-power VCO | $20–40 |
| STM32F407 + external ADC | $40–60 |
| RF components and cabling | $40–60 |
| **Total Antenna + RF Front End** | **$250–330** |

### 6.3 Cost vs. Performance Trade-offs

```
Resolution Budget Guideline:

$100–200: FR-4 patch array + basic RF → 1.5–3 m resolution possible
$200–400: Rogers patch array + better RF → 0.5–1.5 m resolution
$400+:    Large aperture (parabolic) + high-BW RF → < 0.5 m resolution
```

---

## 7. Hobbyist Buildability Assessment

### 7.1 Skill Requirements by Antenna Type

| Antenna Type | Mechanical Skills | RF/Microwave Knowledge | PCB Skills | Difficulty |
|--------------|------------------|------------------------|------------|------------|
| Microstrip Patch Array | Low | Moderate | High | Moderate |
| Waveguide (surplus) | Moderate | Low | None | Easy |
| DIY Waveguide Horn | High | Moderate | None | Moderate |
| Helical | Low | Moderate | None | Easy |
| Parabolic (surplus) | Low | Low | None | Easy |
| Parabolic (DIY) | High | Moderate | None | Hard |

### 7.2 Patch Array Build Process

**Step 1: Design Simulation**
- Use free software: **QMicrostrip** or **OpenEMS** (FDTD)
- Design parameters: resonant frequency, substrate thickness, dielectric constant
- Verify return loss and pattern before fabrication

**Step 2: PCB Fabrication**
- Export Gerber files from simulation tool
- Order from JLCPCB, PCBWay, or OSH Park
- Specify Rogers RT/duroid 5880 (εr = 2.2, 31 mil) for best performance
- FR-4 (εr ≈ 4.3) is acceptable budget alternative but higher loss

**Step 3: Assembly**
- Solder SMA or N connectors to feed network
- Check VSWR with NanoVNA (~$50) before integration
- Verify pattern in far field if possible

**Critical Tolerances at X-Band:**
```
Patch dimensions: ±0.3 mm (critical)
Substrate thickness: ±0.1 mm
Dielectric constant: ±0.1 (for Rogers)
Feed network tolerances: ±0.2 mm
```

### 7.3 Waveguide Build Process

**Step 1: Source Materials**
- WR-90 waveguide sections: eBay, SurplusSales, RF过渡
- Waveguide flanges: UG-39/U or similar

**Step 2: Fabrication**
- Cut waveguide to length
- Drill hole for probe feed (typically 2.5–3 mm diameter)
- Mount N-type or SMA connector
- Basic tuning with probe depth

**Step 3: Testing**
- Check VSWR with NanoVNA
- Open-ended waveguide typically shows S11 ≈ -10 to -15 dB without tuning

### 7.4 Essential Test Equipment

| Equipment | Purpose | Budget Option |
|-----------|---------|---------------|
| NanoVNA (~$50) | VSWR, S-parameter testing | $50–80 |
| DSO Nano or similar | Basic waveform monitoring | $30–50 |
| Frequency counter | Verify oscillator frequency | $20–40 |
| RF power meter | Verify transmit power | $30–50 |
| attenuator kit | Signal level control | $20–40 |

**Minimum viable test setup:** NanoVNA + 20 dB attenuator = ~$80–100

---

## 8. STM32F407 Integration Considerations

### 8.1 ADC Limitations

```
STM32F407 ADC Specifications:
- Resolution: 12-bit
- Sample Rate: Up to 2.4 MSPS (single channel)
- Input Range: 0–3.3V (single-ended)
- Power Supply: 3.3V typical

Effective bandwidth for real sampling: ~1.2 MHz (Nyquist)
For bandwidths above this:
  - Use I/Q sampling (baseband)
  - Use undersampling (bandpass sampling)
```

### 8.2 IF/Baseband Chain Recommendations

**For 50–100 MHz bandwidth:**
- I/Q demodulation to baseband
- Each channel sampled at 2 MSPS
- Provides effective 2 MHz baseband bandwidth

**For >100 MHz bandwidth:**
- Direct RF sampling with undersampling
- Requires proper anti-aliasing filtering
- Bandpass sampling theory: choose f_s to alias band to ADC input range

**Bandpass Sampling Equation:**
```
f_s > 2 × B (Nyquist for band-limited signals)
OR
f_s = (2 × f_c) / N  where N is integer, for bandpass sampling
```

### 8.3 Recommended IF Center Frequency

For STM32F407 with FMCW SAR at X-band:
```
Transmit chirp: 9.6 GHz → 10.6 GHz (100 MHz chirp)
IF center frequency: 5–20 MHz
IF bandwidth: Matches chirp bandwidth (100 MHz)
```

This allows:
- Easy filtering (5-pole Chebyshev LPF at 30 MHz)
- Sampling within ADC Nyquist (2 × 20 MHz = 40 MSPS not needed; 2.4 MSPS is fine)
- Audio-frequency ADC implementation

---

## 9. Recommended Antenna Designs by Budget

### 9.1 Under $100 Budget → Helical or Waveguide

**Design: 12-Turn Helical Antenna (10 GHz)**
```
Frequency: 9.6 GHz
Gain: 12–14 dBi
Beamwidth: 35–40°
Polarization: Circular (RHCP)
Weight: ~100g with feed
Cost: $20–30

Construction:
1. Wind 12 turns of 18 AWG copper wire on 30 mm diameter form
2. Spacing: 10 mm between turns
3. Ground plane: 60 mm × 60 mm aluminum plate at base
4. Feed: RG-402 coax with gamma match
5. Tune by adjusting gamma tap position
```

### 9.2 $100–250 Budget → FR-4 Patch Array

**Design: 4×4 Microstrip Patch Array (10 GHz)**
```
Frequency: 9.6 GHz
Gain: 18–20 dBi
Beamwidth: ~25° (E and H plane)
Bandwidth: ~400 MHz (S11 < -10 dB)
Weight: ~80g (including radome)
Cost: $25–40

Construction:
1. Use FR-4 substrate, 1.6 mm thick
2. Design in QMicrostrip or OpenEMS
3. Order PCBs from JLCPCB ($5–10 for 5 boards)
4. Solder SMA edge launch connectors
5. Verify with NanoVNA before flight
```

### 9.3 $250–500 Budget → Rogers Patch Array + Helical

**Design: 8×8 Corporate-Fed Patch Array (10 GHz)**
```
Frequency: 9.6 GHz
Gain: 24–27 dBi
Beamwidth: ~12° (E and H plane)
Bandwidth: ~500 MHz (S11 < -10 dB)
Weight: ~300g (including radome and frame)
Cost: $120–180 (PCB fabrication + components)

Construction:
1. Rogers RT/duroid 5880, 31 mil thick
2. 8×8 elements with corporate feed network
3. Wilkinson dividers for equal power split
4. Array simulation before fabrication (critical)
5. Precision machining of radome frame
```

### 9.4 $500+ Budget → Parabolic with High-Bandwidth RF

**Design: 18–24" Parabolic + 200 MHz FMCW**
```
Frequency: 9.6 GHz
Gain: 32–35 dBi
Beamwidth: 5–8°
Bandwidth: 200 MHz (0.75 m resolution)
Weight: 1–2 kg
Cost: $150–300 (antenna) + $200–400 (RF)

Construction:
1. Source surplus parabolic dish (TVRO/Satellite)
2. Design scalar feed horn for optimal illumination
3. Integrate low-noise RF front end
4. Precision mount with gimbal for scanning
```

---

## 10. Summary and Recommendations

### 10.1 Antenna Selection Matrix

| Requirement | Best Antenna Choice | Alternative |
|-------------|---------------------|-------------|
| Maximum resolution (drone SAR) | 8×8 Rogers patch or parabolic | 4×4 FR-4 patch (budget) |
| Minimum weight (<150g) | 4×4 FR-4 patch | Helical |
| Minimum cost (<$50) | WR-90 open waveguide | Helical |
| Best penetration (vegetation) | C-band helical or patch | X-band with wider beam |
| Widest swath coverage | Sectoral horn or wide-beam array | Parabolic with scanned feed |
| Easiest to build | WR-90 waveguide or helical | Parabolic (surplus) |

### 10.2 Final Recommendations

**For the typical hobbyist DIY SAR project:**

1. **Start with X-band (9.6 GHz)** — best component availability and manufacturing tolerances

2. **Use a 4×4 FR-4 patch array** ($25–40) as initial antenna:
   - Gain: ~18–20 dBi
   - Beamwidth: ~25° (adequate for 100–300 m altitude)
   - Weight: <100g (drone friendly)
   - Buildability: moderate (PCB fabrication required)

3. **For improved performance**, upgrade to Rogers 5880 8×8 array:
   - Gain: ~26 dBi
   - Beamwidth: ~12° (good resolution at 200 m altitude)
   - Requires careful simulation before fabrication

4. **Consider helical for circular polarization** if specular reflections from water/metal are problematic

5. **For fixed ground-based SAR**, parabolic dish (18–24") gives highest gain and best resolution

### 10.3 Key Formulas Reference

```
Gain from aperture: G = (4πηA)/λ²
Beamwidth (approx): HPBW ≈ 50° × (λ/D)
Azimuth resolution: δa = D/2 (single-pass SAR)
Range resolution: δR = c/(2B)
Path loss: L = 20×log10(4πR/λ)
SAR processing gain: Gproc = 10×log10(N)
```

---

## References and Further Reading

- Balanis, C.A. "Antenna Theory: Analysis and Design" (4th ed.) — comprehensive reference
- Curlander, J.C. & McDonough, R.N. "Synthetic Aperture Radar: Systems and Signal Processing" — SAR-specific
- IEEE Standard for Test Procedures for Antennas (IEEE Std 149-1979)
- NanoVNA user community resources for antenna measurement
- JLCPCB and PCBWay design rules for high-frequency PCB fabrication

---

*Document Version: 1.0*  
*Next Section: See 02_front_end/ for RF front-end design*

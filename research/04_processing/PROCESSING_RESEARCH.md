# SAR Signal Processing: Algorithms & Processor Requirements

**Date:** April 29, 2026  
**Project:** DIY Synthetic Aperture Radar  
**Hardware Context:** STM32F407VG (ARM Cortex-M4, 180MHz, 192KB RAM, 1MB Flash)  
**Target:** X-band FMCW SAR, drone or ground-based

---

## 1. SAR Imaging Fundamentals

### 1.1 The SAR Imaging Principle

Synthetic Aperture Radar achieves fine azimuth (cross-range) resolution not through a large physical antenna, but by coherently combining returns as the radar moves along a synthetic aperture. The radar transmits pulses at regular intervals; each return is phase-history recorded with precise position data.

**Key insight:** A point target at cross-range position `x` produces a sinusoidal phase history across the aperture:
```
φ(n) = (4π/λ) × R(n)
     = (4π/λ) × sqrt(R₀² + (v×n×PRP - x)²)
```
where `R₀` is closest approach range, `v` is platform velocity, `PRP` is pulse repetition interval, and `n` is pulse index. This phase history is a sinusoid in the spatial frequency domain, allowing azimuth compression via matched filtering.

### 1.2 Range Compression

Range compression converts raw IF/chirp returns into range profiles via matched filtering:

```
s_range(t) = I(t) + j×Q(t)  [complex baseband]
Matched filter: h(t) = s*(-t)  [time-reversed conjugate of transmitted chirp]
Output: s_range_compressed(t) = s_range(t) ⊗ h(t) = IFFT{ FFT(s_range) × conj(FFT(transmitted_chirp)) }
```

**For an LFM chirp:** `transmitted(t) = exp(j×π×K×t²)` where `K = BW/T_chirp` (chirp rate)

**Computational cost (per pulse):**
- 1× FFT (N_range samples): N_range×log₂(N_range) complex multiplies
- 1× IFFT: same
- Complex multiply + accumulate for matched filter: N_range per output point
- **Total: ~2×N_range×log₂(N_range) + N_range complex operations**

**For 1024 range bins:** ~22,528 complex operations ≈ 90,112 real MACs per pulse

### 1.3 Azimuth Compression

After range compression, each range bin contains the azimuth phase history for all scatterers at that range. Azimuth compression applies a matched filter matched to the expected phase history for each possible cross-range position:

```
s_azimuth(n) for fixed range = Σ scatterers σ_i × exp(j×(4π/λ)×R_i(n))
Azimuth matched filter: h_az(n) = exp(-j×(4π/λ)×R_expected(n))
Output: IFFT{ FFT(s_azimuth) × conj(FFT(h_az)) } → 2D SAR image
```

**Computational cost:**
- For each range line (N_range bins): N_az×log₂(N_az) FFT + IFFT
- Azimuth processing over N_az pulses
- **Total: N_range × 2×N_az×log₂(N_az) complex operations**

**For 1024 range bins × 1024 azimuth pulses:** ~21M complex operations ≈ 84M real MACs per image frame

### 1.4 SAR Processing Gain

The coherent integration in SAR provides significant processing gain:
```
G_SAR = 10×log₁₀(N_pulses) [dB]
```
- N_pulses = 1024 → ~30 dB processing gain
- This gain is critical for hobbyist systems with low transmit power

---

## 2. Key SAR Processing Algorithms

### 2.1 Algorithm Comparison Overview

| Algorithm | Complexity (MACs) | Memory (N³) | Motion Compensation | Quality |
|-----------|-------------------|-------------|---------------------|---------|
| **Range-Doppler (RD)** | O(N² log N) | O(N²) | Range-dependent | Good |
| **Back-Projection (BP)** | O(N³) | O(N²) | Native, exact | Best |
| **Omega-K (ω-k)** | O(N² log N) | O(N²) | Stolt interpolation | Excellent |
| **Chirp-Scaling (CS)** | O(N² log N) | O(N²) | Phase multiply | Good |

### 2.2 Range-Doppler Algorithm (RDA)

**Principle:** Exploits the separable nature of SAR processing in the range-Doppler domain. Assumes a constant range migration across the aperture, which is valid for moderate squint angles and aperture lengths.

**Processing Steps:**
1. **Range Compression:** FFT → multiply by reference function → IFFT
2. **Range Cell Migration Correction (RCMC):** Phase multiply in 2D frequency domain to align targets across range cells
3. **Azimuth FFT:** Transform to Doppler domain
4. **Azimuth Compression:** Multiply by matched filter (Doppler domain reference function) → IFFT

**Computational Complexity:**
```
Step 1 (Range Compress):      N_range × log₂(N_range) + N_range MACs per pulse
                              × N_az pulses = N_range × N_az × (log₂(N_range) + 1)
                              ≈ 2 × N_range × N_az × log₂(N_range) MACs

Step 2 (RCMC):                N_range × N_az phase multiplies = N_range × N_az MACs

Step 3 (Azimuth FFT):         N_az × log₂(N_az) per range bin
                              × N_range = N_range × N_az × log₂(N_az) MACs

Step 4 (Azimuth Compress):    N_range × N_az × (log₂(N_az) + 1) MACs

Total RDA:                    2 × N_range × N_az × (log₂(N_range) + log₂(N_az)) + 2 × N_range × N_az
                              ≈ 2 × N_range × N_az × (log₂(N_range) + log₂(N_az)) MACs
```

**For 512×512 image:**
```
N_range = 512, N_az = 512
log₂(512) = 9
Total MACs ≈ 2 × 512 × 512 × 18 = 9,437,184 MACs per frame
≈ 9.4 MMACs (mega-MACs)
```

**Memory Requirements:**
- Raw data: 2 × N_range × N_az complex = 2 × 512 × 512 × 16 bytes ≈ 8 MB (16-bit I/Q)
- Range-compressed: can stream, no storage needed
- **Total working memory: ~8 MB for full scene**

**Advantages:** Computationally efficient, well-suited for stripmap mode
**Disadvantages:** Approximation for RCMC limits focusing quality at far ranges or high squint

### 2.3 Back-Projection Algorithm (BPA)

**Principle:** Time-domain correlation that exactly accounts for position-dependent path lengths. For each pixel in the output image, all pulse returns are phase-rotated to compensate for path differences and summed.

**Processing Steps:**
1. For each output pixel (r, x):
   - For each pulse n at position P(n):
     - Compute range R = ||P(n) - (r, x)||
     - Interpolate range-compressed data at delay τ = 2R/c
     - Sum with phase rotation exp(-j×(4π/λ)×R)

**Computational Complexity:**
```
For each output pixel: N_az interpolations + N_az MACs
Total: N_range × N_az output pixels × N_az operations
      = N_range × N_az² operations

For 512×512 image: 512 × 512² = 134,217,728 operations
≈ 134 MMACs per frame
```

**For real-time requirements (10 frames/sec):** 1.34 GMACs/sec — beyond STM32F407 capability.

**Memory Requirements:**
- Raw data must be retained: 2 × N_range × N_az complex = ~8 MB
- Output image: N_range × N_az complex = ~4 MB
- **Total: ~12 MB minimum, exceeds STM32F407's 192KB SRAM**

**Advantages:** Exact motion compensation, handles arbitrary flight paths, best image quality
**Disadvantages:** O(N³) complexity makes it impractical for real-time large images on embedded hardware

### 2.4 Omega-K (ω-k) Algorithm

**Principle:** Exact 2D frequency-domain algorithm using Stolt interpolation to handle range migration. Also called "wavenumber domain" algorithm.

**Processing Steps:**
1. **2D FFT** of raw data: `S(kx, kr)` — transforms to spatial frequency domain
2. **Stolt Interpolation:** Remap `kr` to `sqrt(kr² - kx²)` to account for wavefront curvature
3. **2D IFFT** to get image

**Computational Complexity:**
```
Step 1 (2D FFT):       2 × N_range × N_az × log₂(N_range × N_az) MACs
                        ≈ 2 × N² × log₂(N²) = 4 × N² × log₂(N) for N=N_range=N_az

Step 2 (Stolt):        N_range × N_az interpolations = N² operations

Step 3 (2D IFFT):     Same as Step 1

Total:                 ≈ 4 × N² × log₂(N) + N² MACs
                        ≈ 4 × N² × log₂(N) for large N
```

**For 512×512 image (N=512, log₂(512)=9):**
```
Total MACs ≈ 4 × 512² × 9 = 9,437,184 MACs per frame
≈ 9.4 MMACs (identical to RDA for this size)
```

**Memory Requirements:**
- 2D FFT requires holding full data cube: ~8 MB complex
- **Total: ~8-10 MB, exceeds STM32F407**

**Advantages:** Exact (no approximations), excellent for spotlight SAR, handles wide apertures
**Disadvantages:** Stolt interpolation is memory-intensive, more complex to implement correctly

### 2.5 Algorithm Complexity Summary Table

| Algorithm | MACs (512×512) | MACs (256×256) | Relative Complexity |
|-----------|----------------|----------------|---------------------|
| Range-Doppler | 9.4 MMACs | 2.4 MMACs | 1× (baseline) |
| Omega-K | 9.4 MMACs | 2.4 MMACs | ~1× (similar) |
| Back-Projection | 134 MMACs | 16.8 MMACs | ~14× higher |
| Chirp-Scaling | ~10 MMACs | 2.5 MMACs | ~1.05× |

**Key insight:** Range-Doppler and Omega-K have similar computational loads for typical image sizes. Back-Projection is 10-15× more expensive but handles arbitrary motion exactly.

---

## 3. STM32F407 Viability Assessment

### 3.1 STM32F407 Specifications

```
Core: ARM Cortex-M4 with FPU
Clock: 180 MHz (overclockable to 200+ MHz typical)
Flash: 1 MB (STM32F407VG)
SRAM: 192 KB (128 KB + 64 KB)
ADC: 12-bit, up to 2.4 MSPS (single channel), 7.2 MSPS (in interleaved mode, reduced resolution)
DSP Instructions: Single-cycle MAC, SIMD (SMID is M4-only, limited vs M7)
DMA: 16-channel DMA with FIFO
```

### 3.2 STM32F407 Processing Capacity

**Peak MAC throughput:**
- 180 MHz × 1 MAC/cycle (Cortex-M4 FPU MAC) = 180 MMACs/sec theoretical
- Realistic sustained with memory stalls, loop overhead: 80-120 MMACs/sec

**For SAR processing (512×512 image, RDA algorithm):**
```
9.4 MMACs per frame
At 120 MMACs/sec sustained: 9.4/120 = 0.078 seconds = 12.8 frames/sec theoretical maximum
```

**However:** Memory bandwidth is the bottleneck, not compute.

### 3.3 STM32F407 Memory vs. Requirements

| Memory Need | Requirement | STM32F407 | Ratio |
|-------------|-------------|-----------|-------|
| Raw data (512×512×16b I/Q) | 8 MB | 192 KB | 2.4% of needed |
| Range-compressed (512×512×16b) | 4 MB | 192 KB | 4.8% |
| Full processing buffer | ~12 MB | 192 KB | 1.6% |

**Verdict: STM32F407 CANNOT hold full scene data in memory for 512×512 imaging.**

**What STM32F407 CAN do:**
1. **Range compression only:** Process one pulse at a time, output range line to external storage
2. **Partial azimuth processing:** Small azimuth blocks if external RAM is added
3. **Control and coordination:** Act as system controller, not primary processor

### 3.4 STM32F407 Realistic Processing Role

**Viable tasks on STM32F407:**
1. **FMCW chirp generation:** DDS/PLL control, linear ramp generation
2. **ADC control and data streaming:** DMA-based capture to external memory
3. **Motion data capture:** GPS/IMU data synchronization and timestamping
4. **Partial range compression:** If external memory is available, can do range FFT
5. **System coordination:** Trigger timing, mode switching, PC communication

**NOT viable as standalone SAR processor for meaningful imagery.**

### 3.5 External Memory Extension

Adding external memory enables more processing:
```
STM32F407 + 16 MB external SRAM/SDRAM (e.g., IS42S16400J - 8MB, ~$3):
- Can hold 512×512×16b complex (8 MB) in external RAM
- But external RAM access is ~3-5× slower than internal SRAM
- Bus contention with FMC and processor reduces effective bandwidth
```

**Verdict:** Even with external memory, STM32F407 lacks compute throughput for real-time azimuth processing.

---

## 4. PC-Assist Strategy

### 4.1 Hybrid Processing Architecture

Split processing between embedded (STM32) and PC based on computational and memory requirements:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SAR Signal Processing Pipeline               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │ STM32F407    │    │ Data Link    │    │ PC (Python/C++)  │  │
│  │              │    │ (USB/UART/   │    │                  │  │
│  │ • Chirp Gen  │───▶│  Wireless)   │───▶│ • Range Comp    │  │
│  │ • ADC Stream │    │              │    │ • Azimuth Comp  │  │
│  │ • GPS/IMU    │    │              │    │ • Image Formation│  │
│  │ • Sync/Timing│    │              │    │ • Display/Output │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
│                                                                 │
│  EMBEDDED: Real-time     │  PC: Offline processing            │
│  timing-critical control │  (post-flight or streaming)         │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Embedded-Side Responsibilities (STM32F407)

| Function | Rationale for Embedded | Computational Load |
|----------|------------------------|-------------------|
| **FMCW chirp generation** | Timing-critical, must be deterministic | Low (PLL/DDS control) |
| **ADC data capture** | High-speed streaming, DMA needed | Low (pass-through) |
| **GPS/IMU data fusion** | Precise timestamping required | Medium (Kalman filter) |
| **Trigger generation** | Synchronization with platform position | Low |
| **Range compression** | Possible with external RAM, reduces data rate 100:1 | 2-5 MMACs/pulse |

**Data rate from embedded to PC:**
- Raw IF data: 2.4 MSPS × 12-bit × 2 channels (I/Q) = 57.6 Mbps
- After range compression: N_range complex samples per pulse = 1024×32 = 32.8 Kbps per pulse
- At 1 kHz PRF: 32.8 Mbps → still high, but manageable with compression

**Recommendation:** Do range compression on embedded, stream compressed range lines to PC.

### 4.3 PC-Side Responsibilities

| Function | Computational Load | Memory |
|----------|-------------------|--------|
| **Azimuth compression** | 5-10 MMACs/frame | 4-8 MB |
| **Motion compensation** | 1-2 MMACs/frame | 2-4 MB |
| **Image formation** | Varies by algorithm | Output image |
| **Image display/geo-coding** | Low | Display buffer |

**PC-side minimum requirements:**
- Modern laptop (2018+): 4+ cores, 8+ GB RAM, USB3 for data streaming
- Python with NumPy/SciPy: Can do 512×512 RDA in <1 second

### 4.4 Data Link Options

| Interface | Bandwidth | Latency | Implementation |
|-----------|-----------|---------|----------------|
| **USB 2.0** | 480 Mbps | <1 ms | STM32F407 OTG HS (requires PHY) |
| **USB UART (FT232)** | 9 Mbps | <1 ms | Simple, but limited |
| **WiFi (ESP8266)** | 20+ Mbps | 5-20 ms | Wireless, adds latency |
| **SD card logging** | 20+ MB/sec | N/A (async) | Simplest for full raw data |

**Recommendation for hobbyist:**
1. **Option A (Streaming):** UART at 3-10 Mbps for compressed range lines + separate raw data log to SD card
2. **Option B (Full offline):** Log raw data to SD card during flight, process on PC post-flight

---

## 5. Data Rates and Storage

### 5.1 Raw Data Volume Calculation

**Assumptions:**
- PRF: 1 kHz (pulse repetition frequency)
- Platform speed: 10 m/s
- Azimuth aperture: 512 pulses (51.2 m synthetic aperture)
- Range samples: 1024 (at 2.4 MSPS ADC, 500 μs sweep)
- Format: 16-bit I + 16-bit Q = 32 bits per sample

**Per pulse:**
```
Raw data: 1024 samples × 32 bits = 4 KB per pulse
At 1 kHz PRF: 4 MB/sec = 28.8 GB/hour
```

**For a 5-minute flight:**
```
Duration: 300 seconds
Pulses: 300,000
Raw data: 300,000 × 4 KB = 1.2 GB
```

**After range compression (16-bit complex per range bin):**
```
Compressed: 1024 × 32 bits = 4 KB per pulse (same size, but processed)
After azimuth compression: 512 × 512 × 16-bit = 512 KB per image frame
```

### 5.2 Storage Requirements

| Data Type | Per Pulse | Per 5-min Flight | Storage Medium |
|-----------|-----------|------------------|----------------|
| Raw I/Q | 4 KB | 1.2 GB | SD card (essential) |
| Range-compressed | 4 KB | 1.2 GB | SD card (if streaming fails) |
| Motion data (GPS/IMU) | 100 bytes | 30 MB | STM32F407 internal + SD |
| Final images | 512 KB/frame | ~100 MB (200 frames) | PC storage |

### 5.3 STM32F407 Storage Capability

**STM32F407 internal:**
- 1 MB Flash: Can store program + small lookups
- 192 KB SRAM: Cannot hold raw data

**External storage needed:**
- **SD card (SPI or SDIO):** Essential for hobbyist builds
- Recommended: 16-32 GB microSD for full flight logging
- Data rate: 4 MB/sec requires Class 10 or UHS-I card

**Streaming vs. Logging trade-off:**
```
Streaming to PC:
- Pros: Real-time preview, no data loss
- Cons: Requires reliable high-bandwidth link
- Bandwidth needed: 4 MB/sec → USB 2.0 or WiFi

SD card logging:
- Pros: Simple, reliable, no data loss
- Cons: Must retrieve card post-flight, no real-time preview
- Bandwidth: Direct SD write at 4 MB/sec (achievable)
```

**Recommendation:** SD card logging primary, WiFi streaming as secondary for real-time monitoring.

---

## 6. Motion Compensation (GPS/IMU Requirements)

### 6.1 Why Motion Compensation Matters

SAR imaging coherently integrates returns across the synthetic aperture. Uncompensated platform motion errors manifest as:
- **Phase errors:** Blurring in azimuth
- **Range errors:** Geometric distortion
- **Doppler centroid shifts:** Mis-registration

**Error budget for X-band (λ = 3 cm):**
```
1 radian RMS phase error ≈ λ/4 = 7.5 mm position accuracy needed
For diffraction-limited image: λ/8 = 3.75 mm accuracy needed
```

### 6.2 Motion Measurement Requirements

**Position accuracy requirements:**

| Error Type | RMS Requirement (X-band) | Effect |
|------------|---------------------------|--------|
| Along-track position | < λ/4 = 7.5 mm | Phase error, azimuth blurring |
| Cross-track position | < λ/8 = 3.75 mm | Geometric distortion |
| Along-track velocity | < 0.1 m/s stability | Doppler centroid error |
| Altitude | < 0.5 m (for mapping) | Ground range error |

**Attitude accuracy requirements:**

| Error Type | RMS Requirement | Effect |
|------------|------------------|--------|
| Yaw | < 0.1° (1.7 mrad) | Azimuth focusing |
| Pitch/Roll | < 0.5° | Range plane tilt |

### 6.3 GPS Specifications

**Minimum GPS for SAR motion compensation:**

| GPS Type | Position Accuracy | Velocity Accuracy | Update Rate | Cost |
|----------|-------------------|-------------------|-------------|------|
| **UBlox 6M/7M (SPI/Serial)** | 2.5 m CEP | 0.05 m/s | 5-10 Hz | $10-20 |
| **UBlox F9P (RTK)** | 2 cm + 1 ppm | 0.02 m/s | 20 Hz | $150-200 |
| **u-blox C94-M8P (RTK pair)** | 2 cm (with base) | 0.02 m/s | 10 Hz | $200-400 |
| **Survey-grade (Trimble)** | 1 cm | < 0.01 m/s | 10-20 Hz | $5000+ |

**For hobbyist SAR:**
- **UBlox 6M/7M:** Provides baseline motion data; acceptable for 10m resolution systems
- **UBlox F9P:** Recommended for <1m resolution; enables post-processing kinematic (PPK) correction
- **RTK GPS:** Significant improvement but cost may exceed rest of project

**GPS sampling for SAR:**
- Store GPS position with each pulse (via 1PPS + serial timestamp)
- Interpolate between updates for sub-pulse position accuracy
- Use velocity output directly for Doppler navigation

### 6.4 IMU Specifications

**Why IMU is needed:**
- GPS update rate (5-10 Hz) too slow for pulse-to-pulse position tracking
- IMU provides high-rate (100-1000 Hz) attitude data
- GPS+IMU fusion gives smooth, accurate trajectory

**Minimum IMU for SAR:**

| IMU Type | Gyro Accuracy | Accel Accuracy | Update Rate | Cost |
|----------|---------------|----------------|-------------|------|
| **MPU-6050 (6-axis)** | 1°/sec | 50 mg | 1 kHz | $3-5 |
| **MPU-9250 (9-axis)** | 1°/sec | 50 mg | 1 kHz | $10-15 |
| **BMI160** | 0.5°/sec | 15 mg | 1.6 kHz | $10 |
| **ADXL345 + L3GD20** | 0.5°/sec | 2 mg | 0.8 kHz | $15-20 |

**IMU integration for SAR motion compensation:**
```
Position(t) = Position(t₀) + ∫∫(Accel - Gravity) dt² + GPS(t) × Kalman Fuse
Attitude(t) = Attitude(t₀) + ∫(Gyro) dt + GPS-Attitude × Kalman Fuse
```

**Hobbyist approach:**
1. GPS provides absolute position at 5-10 Hz
2. IMU provides high-rate delta-position between GPS updates
3. Extended Kalman Filter (EKF) fuses GPS + IMU for smooth trajectory
4. STM32F407 can run simplified EKF at 100+ Hz

### 6.5 Recommended Motion Sensing Setup

**Minimum (budget <$50):**
```
1× UBlox 7M GPS module (NEO-7M): $15
1× MPU-6050 IMU: $5
1× STM32F407 (on-hand)
Total: ~$20 additional
```

**Improved (budget <$200):**
```
1× UBlox F9P GPS + RTK base: $150
1× BMI160 or MPU-9250: $15
1× External SD card + enclosure: $20
Total: ~$185
```

**Data synchronization:**
```
GPS: 1PPS pulse output → STM32 EXTI trigger
     Serial (115200-921600 baud) → position dump
IMU: SPI/I2C → high-rate samples (100-1000 Hz)
     DMA buffer → timestamped samples
ADC: Triggered by same clock as GPS PPS
     Aligned sample numbers to GPS time
```

---

## 7. Hobbyist GitHub Projects for Reference

### 7.1 Notable Open-Source SAR Projects

**1. SARPi (Raspberry Pi SAR)**
- GitHub: (search "SARPi" or "raspberry pi sar")
- X-band FMCW radar on Raspberry Pi
- Range-Doppler processing in Python/C
- Good reference for software-defined radar

**2. ReSAR (Research SAR)**
- GitHub: various implementations
- Educational SAR processing algorithms
- Includes RDA, BP, omega-k in Python/MATLAB

**3. DIY SAR by "Tyffel" / Various Hobbyists**
- YouTube + GitHub: "DIY SAR radar"
- Simple FMCW radar + processing
- Some achieve basic imaging with 256×256 resolution

**4. GNU Radio SAR Processing**
- Various gr-sar modules
- Software-defined radio approach
- Can work with ADALM-PLUTO or similar SDR

**5. MiniSAR (Flightgear/hobbyist community)**
- Open-source SAR simulator + processor
- Good for algorithm testing without hardware

### 7.2 Key Software Components to Study

| Component | Where to Find | Relevance |
|-----------|--------------|-----------|
| FMCW chirp generation | HackRF, RTL-SDR, gr-radar | Radar waveforms |
| Range compression | NumPy, SciPy, fftw | Core algorithm |
| RDA implementation | Open-source SAR tools | Azimuth processing |
| Motion compensation | GPS/IMU libraries | Position tracking |
| Image formation | Various hobbyist repos | End-to-end pipeline |

### 7.3 Python Prototyping Approach

**Python is ideal for SAR algorithm development because:**

1. **Rapid iteration:** NumPy vectorized operations, no compile cycle
2. **Debugging:** Interactive REPL, visualization with matplotlib
3. **Libraries:** SciPy (FFT, signal processing), NumPy (linear algebra)
4. **Community:** Extensive SAR resources, MATLAB porting straightforward

**Recommended Python environment:**
```bash
# Core libraries
numpy>=1.20        # Array operations, FFT
scipy>=1.7         # Signal processing, special functions
matplotlib>=3.4    # Visualization
numba>=0.54        # JIT compilation for speedup (optional)

# For STM32 integration
pySerial>=3.5      # Serial communication
numpy-stl>=2.17    # 3D visualization (optional)

# For real-time (desktop)
pyFFTW>=0.13       # Fast FFTW wrappers (faster than NumPy FFT)
```

**Python processing speed estimates (512×512 image, Range-Doppler):**
```
Range compression:  ~50 ms (NumPy FFT)
Azimuth compression: ~200 ms (NumPy FFT)
Total Python: ~300 ms per frame (3 fps)

With numba JIT: ~100 ms per frame (10 fps) - desktop only
```

### 7.4 Phased Build Strategy for Working Demo

**Phase 1: Data Collection (1-2 weeks)**
```
Goal: Collect raw SAR data with motion reference
Hardware: FMCW radar + STM32 + GPS/IMU + SD logger
Deliverable: Raw data files for processing development
Steps:
  1. Build/connect FMCW radar front-end
  2. Implement ADC capture + SD card logging on STM32
  3. Connect GPS (serial) and IMU (I2C/SPI) to STM32
  4. Verify synchronized data collection
  5. Collect test data over known scene (parking lot, building)
```

**Phase 2: Python Processing Prototype (1-2 weeks)**
```
Goal: Verify algorithms work on collected data
Software: Python scripts reading SD card data
Deliverable: Processed SAR images from Phase 1 data
Steps:
  1. Parse STM32 binary format to NumPy arrays
  2. Implement range compression (FFT + matched filter)
  3. Implement RDA azimuth processing
  4. Add basic motion compensation using GPS data
  5. Generate and visualize SAR images
```

**Phase 3: Real-Time Streaming (1-2 weeks)**
```
Goal: Process data in real-time during collection
Hardware: Add WiFi or USB streaming to existing setup
Software: Python receiver + streaming processor
Deliverable: Live SAR imaging during movement
Steps:
  1. Implement STM32 → PC data streaming (UART/WiFi)
  2. Write Python receiver with ring buffer
  3. Integrate Phase 2 processing into streaming pipeline
  4. Add simple display (matplotlib animation or OpenCV)
  5. Test on moving platform (cart, drone)
```

**Phase 4: Embedded Enhancement (2-4 weeks)**
```
Goal: Move processing closer to real-time
Hardware: External RAM for STM32 (optional)
Software: Optimized range compression on STM32
Deliverable: Reduced data bandwidth to PC
Steps:
  1. Add external SRAM/SDRAM to STM32F407
  2. Port range compression to STM32 (CMSIS DSP)
  3. Implement double-buffering for streaming
  4. Profile and optimize critical loops
  5. Characterize throughput and latency
```

**Phase 5: Full Autonomous System (ongoing)**
```
Goal: Complete SAR imaging system
Hardware: Integrate all subsystems
Software: Full motion-compensated processing
Deliverable: Produces geo-rectified SAR images
Steps:
  1. Implement GPS+IMU EKF on STM32
  2. Add precise motion compensation to processor
  3. Implement geo-coding (slant-range to ground-range)
  4. Build enclosure and power system
  5. Flight testing and characterization
```

### 7.5 Expected Results by Phase

| Phase | Resolution | Frame Size | Update Rate | Notes |
|-------|------------|------------|-------------|-------|
| Phase 1-2 | 2-4 m | 512×512 | Offline | Proof of concept |
| Phase 3 | 1-2 m | 256×256 | 1-2 fps | Real-time preview |
| Phase 4 | 0.5-1 m | 256×256 | 2-5 fps | Embedded assist |
| Phase 5 | 0.25-0.5 m | 512×512 | 1-2 fps | Full system |

---

## 8. Computational Complexity Summary

### 8.1 Algorithm MAC Counts by Image Size

| Image Size | Range-Doppler | Omega-K | Back-Projection |
|------------|---------------|---------|-----------------|
| 128×128 | 0.15 MMACs | 0.15 MMACs | 2.1 MMACs |
| 256×256 | 1.2 MMACs | 1.2 MMACs | 16.8 MMACs |
| 512×512 | 9.4 MMACs | 9.4 MMACs | 134 MMACs |
| 1024×1024 | 78 MMACs | 78 MMACs | ~1000 MMACs |

### 8.2 Processing Time by Platform

| Platform | Peak MACs/sec | 256×256 RD | 512×512 RD |
|----------|---------------|------------|------------|
| STM32F407 (180MHz) | 120 MMACs | 1 sec | 8 sec |
| STM32H743 (480MHz) | 400 MMACs | 0.3 sec | 2.4 sec |
| Raspberry Pi 4 | 5,000 MMACs | 0.24 msec | 2 msec |
| Laptop (modern) | 50,000+ MMACs | 0.024 msec | 0.2 msec |

**Note:** STM32F407 cannot achieve real-time for 512×512 at 1 fps; laptop handles easily.

### 8.3 Memory Footprint

| Processing Stage | Data Size (512×512) | STM32F407 Compatible? |
|-----------------|---------------------|----------------------|
| Raw I/Q buffer | 8 MB | No |
| Range-compressed | 4 MB | No |
| Azimuth FFT buffer | 8 MB | No |
| Full image | 4 MB | No |
| Single range line | 4 KB | Yes |
| Single azimuth FFT | 2 KB | Yes |

---

## 9. Conclusions and Recommendations

### 9.1 STM32F407 Role Summary

**CAN do:**
- Chirp generation and timing control
- ADC capture and DMA to external storage
- GPS/IMU data fusion (simplified EKF)
- Single-pulse range compression (with external memory)
- System coordination and PC communication

**CANNOT do:**
- Full 2D SAR image formation (insufficient memory and MACs)
- Real-time azimuth processing at meaningful resolution
- Large FFT operations (needs external RAM)

**Verdict:** STM32F407 is a good **system controller and data collector**, not a SAR processor. Use it as the "front-end controller" in a PC-assist architecture.

### 9.2 Recommended Processing Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    RECOMMENDED ARCHITECTURE                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [Antenna + RF Frontend] ──▶ [STM32F407 + ADC]              │
│                               │                              │
│                               ├──▶ [SD Card: Raw I/Q]        │
│                               │    (1.2 GB per 5-min flight) │
│                               │                              │
│                               └──▶ [Range Compression]       │
│                                    (embedded or PC)           │
│                                         │                    │
│                                    [PC: Azimuth + Display]  │
│                                                              │
│  [GPS/IMU] ──▶ [STM32: EKF fusion] ──▶ [Motion file]       │
│                                      (timestamps each pulse) │
└──────────────────────────────────────────────────────────────┘
```

### 9.3 Next Steps

1. **Start with Phase 1:** Build data collection system
2. **Test with Python offline:** Verify algorithm correctness
3. **Add streaming:** Enable real-time preview
4. **Iterate on motion compensation:** GPS quality drives image quality
5. **Target resolution:** Start with 4m resolution (easy), progress to 0.5m (hard)

---

## References and Further Reading

- Cumming, I. & Wong, F. "Digital Processing of Synthetic Aperture Radar Data" (Artech House, 2005)
- Masters, D. "Tutorial: SAR Processing" - various online resources
- IEEE Xplore: Range Doppler Algorithm, back-projection algorithms
- GitHub: SARPi, gr-radar, various DIY SAR hobbyist projects
- STM32F4 CMSIS-DSP documentation for embedded optimization

---

**Document Version:** 1.0  
**Status:** Research complete, ready for implementation  
**Next Review:** After Phase 1 data collection
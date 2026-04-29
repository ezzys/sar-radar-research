# Processor and Hardware Options for DIY SAR

## Executive Summary

This document evaluates processor and hardware options for a DIY Synthetic Aperture Radar (SAR) system, with a focus on hobbyist budget constraints. The STM32F407 is evaluated as the starting point, with alternatives explored up to the $100 price ceiling.

**Key Finding**: The STM32F407 is **not suitable** for SAR image formation but can handle real-time raw data acquisition and preprocessing. A Raspberry Pi CM4 or PC-assisted architecture is recommended for the computationally intensive image formation step.

---

## 1. STM32F407 Viability Assessment

### Specifications
- **CPU**: ARM Cortex-M4 @ 180 MHz
- **FPU**: Single-precision floating-point
- **RAM**: 192 KB (SRAM)
- **Flash**: 1 MB (F407) / 2 MB (F407ZG)
- **DSP**: Single-cycle MAC, SIMD instructions

### Viability for SAR Processing

| SAR Processing Stage | STM32F407 Verdict | Notes |
|---------------------|-------------------|-------|
| Data Acquisition | ✅ YES | Fast enough for ADC interface at typical IF bandwidths (≤10 MSPS) |
| Pulse Compression (matched filtering) | ⚠️ MARGINAL | Possible for short apertures but extremely limited |
| Range-Doppler Algorithm | ❌ NO | Insufficient RAM for FFT arrays; ~1000x too slow |
| Back-Projection | ❌ NO | Requires O(N³) operations; 192 KB RAM limits image to <50×50 pixels |
| Motion Compensation | ⚠️ MARGINAL | Can compute attitude corrections but cannot apply to full image |

### Why STM32F407 Cannot Do SAR Image Formation

1. **Memory Bottleneck**: A modest SAR image at 256×256 pixels requires:
   - Raw data buffer: 256 × 1024 pulses × 2 bytes = 524 KB (exceeds entire RAM)
   - Intermediate FFT buffers: 2 × 256 × 4 bytes × 2 = 2 MB (10× RAM)

2. **Computational Bottleneck**: A 256×256 Range-Doppler processing requires:
   - ~1.7 billion MAC operations per image
   - At 180 MHz with ~2 MACs/cycle = 90 MHz effective → **~19 seconds minimum**, real-time impossible

3. **Use Case for STM32F407**: Act as a **data acquisition controller** that streams raw ADC data to external storage or a host PC via high-speed USB (FSMC interface).

---

## 2. Computational Requirements for SAR Algorithms

### 2.1 Range-Doppler Algorithm (RDA)

The Range-Doppler algorithm is the most common stripmap SAR processing method.

**Algorithm Steps**:
1. Range compression (matched filtering): 2D FFT → multiply by reference function → 2D IFFT
2. Azimuth FFT
3. Range Cell Migration Correction (RCMC)
4. Azimuth compression

**Computational Complexity**:
- For image size **N_range × N_azimuth** (typically 256×1024 to 4096×4096):
  - **MACs per image**: ~5 × N_range × N_azimuth × log₂(N)
  - For 256×1024 image: ~5 × 256 × 1024 × 10 ≈ **13 million MACs**

| Image Size | MACs (approx) | ARM Cortex-M4 Time | ARM Cortex-A7 Time (Pi 3) |
|------------|---------------|--------------------|-----------------------------|
| 128×512    | 3.4M          | ~38 seconds        | ~40 ms                      |
| 256×1024   | 13M           | ~144 seconds       | ~150 ms                     |
| 512×2048   | 53M           | ~590 seconds       | ~600 ms                     |

**Memory Requirements for RDA**:
- Raw data: N_range × N_azimuth × 4 bytes (complex float) = 2 MB for 256×1024
- FFT twiddle factors: N × 8 bytes ≈ 8 KB
- Reference function: N_range × 4 bytes ≈ 1 KB
- **Total Working Set**: ~8-12 MB

### 2.2 Back-Projection Algorithm (BPA)

Back-projection is computationally intensive but exact and parallelizable.

**Computational Complexity**:
- For N aperture positions and M×M image grid:
  - **MACs**: O(N × M²)
  - Each pixel requires interpolating from range history

| Image Size | Aperture Points | MACs (approx) | Notes |
|------------|-----------------|---------------|-------|
| 100×100    | 256             | 2.5M          | Feasible on Pi-class SBC    |
| 256×256    | 512             | 33M           | Stretch for single-core     |
| 512×512    | 1024            | 268M          | Requires multi-core or PC   |

**Memory Requirements for BPA**:
- Raw aperture data: N × N_range × 4 bytes
- Image grid: M² × 4 bytes
- **Total**: ~50-500 MB for practical images

### 2.3 Omega-K Algorithm

- **MACs**: Similar to RDA but with higher constant factor due to Stolt interpolation
- **Memory**: Similar to RDA
- **Advantage**: More efficient for wide-beam SAR

---

## 3. Hardware Alternatives

### 3.1 Raspberry Pi Compute Module 4 (CM4)

| Specification | Value |
|---------------|-------|
| CPU | Broadcom BCM2711 (Cortex-A72) @ 1.5 GHz |
| RAM | 1 GB, 2 GB, 4 GB, or 8 GB options |
| Storage | eMMC (optional) or microSD |
| GPIO | 28× GPIO |
| Network | Gigabit Ethernet |
| USB | 2× USB 3.0 |

**SAR Processing Capability**:
- Single-core floating-point: ~30-50 MFLOPS
- **Quad-core total**: ~120-200 MFLOPS
- Can process 256×1024 RDA image in ~150 ms (single-threaded)
- **Multi-threaded OpenMP**: ~50-80 ms for same image

**Pros**:
- ✅ Affordable (CM4S module ~$35, carrier board ~$30-60)
- ✅ Linux OS with full toolchain
- ✅ Hardware floating-point with NEON SIMD
- ✅ Low power (~5-10W)
- ✅ 64-bit processing

**Cons**:
- ❌ No built-in ADC interface (need external hardware)
- ❌ eMMC or microSD storage may be slow for raw data streaming

**Recommended Config**: CM4 4GB + carrier board with USB 3.0 and Ethernet

### 3.2 NVIDIA Jetson Nano

| Specification | Value |
|---------------|-------|
| CPU | Cortex-A57 @ 1.43 GHz (4-core) |
| GPU | NVIDIA Maxwell @ 921 MHz (128 CUDA cores) |
| RAM | 4 GB LPDDR4 |
| Power | 5-10W (10W mode) |

**SAR Processing Capability**:
- GPU acceleration can achieve 10-50× CPU speedup for FFT-based algorithms
- 256×1024 image with CUDA: **<10 ms**
- **However**: Cost ~$150 (over budget)

**Verdict**: Excellent performance but exceeds $100 budget ceiling

### 3.3 FPGA Options

FPGAs offer massive parallelism for SAR processing but have higher learning curve and cost.

**Xilinx Artix-7 (XC7A35T)**:
- **Logic Cells**: 33K
- **Block RAM**: 1.8 Mb
- **DSP Slices**: 90 (18×25 multipliers)
- **Cost**: ~$80-100 for development board (e.g., Arty A7)
- **SAR Capability**: Can implement parallel FFT and matched filtering pipelines
- **Processing Time**: 256×256 image in ~50 ms with optimized design

**Lattice ECP5**:
- Lower cost ($30-50 for development board)
- Fewer resources but sufficient for small images
- Good for pulse compression, less suitable for full image formation

**FPGA Pros**:
- ✅ Massively parallel processing
- ✅ Deterministic latency
- ✅ Can implement custom fixed-point pipelines for 10× efficiency gain
- ✅ Real-time processing possible

**FPGA Cons**:
- ❌ Steep learning curve (Verilog/VHDL)
- ❌ Debug is challenging
- ❌ Floating-point support limited without IP cores
- ❌ Development boards push or exceed budget

**Recommendation for FPGA**: Consider only if you have prior FPGA experience and need real-time processing. Otherwise, CM4 + PC assist is more practical.

### 3.4 Other Single-Board Computers Under $100

| SBC | CPU | RAM | Performance | Notes |
|-----|-----|-----|-------------|-------|
| **Raspberry Pi 4B (4GB)** | Cortex-A72 @ 1.5GHz | 4 GB | ~180 MFLOPS | Best overall value, $55-65 |
| **Rock Pi 4B** | RK3399 (2×A72 + 4×A53) | 4 GB | ~200 MFLOPS | M.2 slot for NVMe SSD, ~$70 |
| **Orange Pi 5** | RK3588 (4×A76 + 4×A55) | 4-8 GB | ~400 MFLOPS | Newer, faster, ~$60-80 |
| **BeagleBone AI-64** | TI TDA4VM (8×A72 + C66x DSP) | 8 GB | ~500 MFLOPS + DSP | Built-in DSP, ~$100 |
| **NVIDIA Jetson Orin Nano** | 6×A78 + Ampere GPU | 8 GB | ~1000+ GFLOPS | Over budget ($200+) |

**Best Recommendation Under $100**: **Raspberry Pi 4B (4GB)** or **Rock Pi 4B**

---

## 4. PC-Assist Architecture

For hobbyist SAR systems, a hybrid architecture often makes the most sense:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SAR SYSTEM ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐      ┌──────────────────┐      ┌──────────────────┐
  │   RF/Analog  │      │   STM32F407      │      │   SBC (CM4/Pi4)  │
  │   Frontend   │─────▶│   (Acquisition   │─────▶│   (Image        │
  │              │      │    Controller)    │      │    Formation)    │
  └──────────────┘      └──────────────────┘      └────────┬─────────┘
                                                           │
                         ┌──────────────────┐              │
                         │   GPS/IMU        │              │
                         │   (Motion        │              │
                         │    Comp)         │              │
                         └──────────────────┘              │
                                                           │
                                                           ▼
                                                  ┌──────────────────┐
                                                  │   Storage (USB   │
                                                  │   SSD or PC)      │
                                                  └──────────────────┘
```

### STM32F407 as Data Acquisition Controller
- Handles ADC timing and triggering
- Performs 12/16-bit ADC sampling at up to 24 MSPS
- Buffers data and streams via USB 2.0 High-Speed (FSMC) or Ethernet
- Runs on 3.3V logic, interfaces directly with most AD9361/AD9363 modules

### SBC as Image Formation Engine
- Receives raw data from STM32 or directly from ADC via high-speed interface
- Runs SAR imaging algorithms (Range-Doppler, Back-Projection)
- Stores final images and intermediate results
- Can run real-time display of processing progress

### PC as Heavy Processing (Optional)
- Offload final image enhancement, geocoding, filtering
- Use MATLAB, Python (NumPy/SciPy), or C++ with FFTW/OpenCV
- Store large datasets

### Data Flow Options

| Option | Bandwidth | Complexity | Latency |
|--------|-----------|------------|---------|
| STM32 → SBC via USB 2.0 | ~35 MB/s | Low | Medium |
| STM32 → SBC via Ethernet | ~100 MB/s | Medium | Low |
| Direct ADC → SBC (via LVDS) | ~800 MB/s | High | Very Low |
| STM32 → PC via USB 3.0 | ~200 MB/s | Medium | Medium |

---

## 5. Data Storage Considerations

### 5.1 Storage Requirements for SAR

For a typical stripmap SAR with 10 MHz IF bandwidth, 16-bit samples:

| Acquisition Duration | Sample Rate | Raw Data Size |
|----------------------|-------------|---------------|
| 1 second             | 20 MSPS     | 40 MB         |
| 10 seconds           | 20 MSPS     | 400 MB        |
| 1 minute             | 20 MSPS     | 2.4 GB        |
| 10 minutes           | 20 MSPS     | 24 GB         |

**Recommendation**: Budget for at least 32-64 GB storage for practical experiments.

### 5.2 Storage Options

| Storage Type | Read/Write Speed | Capacity | Cost (64GB) | Suitability |
|--------------|------------------|----------|-------------|-------------|
| **microSD (UHS-I)** | ~100 MB/s | Up to 1TB | $15-25      | Only for final images, too slow for raw streaming |
| **USB 3.0 Flash Drive** | ~200 MB/s | Up to 512GB | $20-40     | Marginal for raw data |
| **USB 3.0 SSD (External)** | ~500 MB/s | 500GB-2TB | $50-80     | ✅ Good for raw data and processing |
| **NVMe via USB-C** | ~1000 MB/s | 256GB-2TB | $60-120    | ✅✅ Best performance |
| **SATA SSD (USB 3.0)** | ~550 MB/s | 256GB-2TB | $40-70     | ✅ Good balance |

**Recommendation**: A **USB 3.0 external SSD** (500GB ~$50) is the best balance of cost, speed, and capacity for DIY SAR. If budget allows, NVMe via USB-C provides headroom for higher sample rates.

### 5.3 microSD Limitations

microSD cards (even UHS-I/UHS-II) are **not suitable** for raw SAR data acquisition because:

1. **Sequential write speed**: Marketing claims ~100 MB/s, real-world ~60-80 MB/s
2. **Sustained write**: Cards throttle after ~1-2 GB to ~20-30 MB/s
3. **Latency**: High and unpredictable (bad for streaming)
4. **Life expectancy**: Heavy writes degrade cards quickly

**Only use microSD** for: OS boot, final image storage, small datasets

---

## 6. GPS/IMU Requirements for Motion Compensation

Motion compensation (Mocomp) is **critical** for SAR image quality. Even 1mm position errors over a 1m aperture cause significant image degradation at X-band.

### 6.1 GPS Requirements

| Parameter | Requirement | Recommended |
|-----------|-------------|-------------|
| Position Accuracy | <1 cm (10 mm) | RTK GPS or DGPS |
| Update Rate | ≥10 Hz (20+ Hz preferred) | 10-20 Hz |
| Raw Data Output | Required for post-processing | NMEA + proprietary |

**Options**:

| GPS Type | Accuracy | Cost | Notes |
|----------|----------|------|-------|
| **u-blox ZED-F9P** | 1-2 cm (RTK) | ~$150 (module) | Best hobbyist option, dual antenna capable |
| **u-blox M8P** | 10-20 cm | ~$60 | Good, but not RTK |
| **u-blox NEO-M8N** | 2-3 m | ~$20 | Too coarse for SAR |
| **u-blox CAM-M8Q** | 2.5 m | ~$15 | Not suitable |

**Recommendation**: **u-blox ZED-F9P** is the minimum for serious SAR. It supports RTK with base station or NTRIP caster for <2 cm accuracy.

### 6.2 IMU Requirements

IMU is needed for **attitude (pitch, roll, yaw)** and **short-term position** updates between GPS fixes.

| Parameter | Requirement | Notes |
|-----------|-------------|-------|
| Gyroscope | ±450°/s min, <0.1°/s resolution | For attitude |
| Accelerometer | ±16g min | For vibration isolation |
| Update Rate | ≥100 Hz | For interpolation |
| Allan Variance | <10°/h (gyro) | Determines drift rate |

**Options**:

| IMU Type | Gyro Range | Noise Density | Cost | Notes |
|----------|------------|---------------|------|-------|
| **BMI088** | ±2000°/s | 0.014°/s/√Hz | ~$15 | Good hobbyist choice |
| **ICM-20948** | ±250°/s | 0.005°/s/√Hz | ~$10 | Excellent, 9-axis |
| **ADXL345 + L3GD20** | Various | Moderate | ~$5 | Budget option, separate chips |
| **Nav2 (ESP32)** | ±450°/s | 0.01°/s/√Hz | ~$30 | Integrated, easy interface |

**Recommendation**: **ICM-20948** (~$10) or **BMI088** (~$15) for serious SAR. The integrated Nav2 modules are convenient but more expensive.

### 6.3 Motion Compensation Data Fusion

**Extended Kalman Filter (EKF)** is typically used to combine:
- GPS position (accurate, low-rate: 10-20 Hz)
- IMU (high-rate: 100-1000 Hz, but drifts)

**Processing Requirements**:
- EKF update: ~1000 FLOPS per sample
- Update rate: 100 Hz
- **Total**: ~1 MFLOPS (trivial for any SBC)

### 6.4 Minimum Viable Motion Sensing

For a **bare-bones hobbyist SAR**:
- **GPS**: u-blox ZED-F9P (RTK) — $150
- **IMU**: ICM-20948 — $10

If budget is extremely tight:
- **GPS**: Skip real-time, use GCPs (Ground Control Points) for post-processing
- **IMU**: BMI088 for attitude only

---

## 7. Complete SBC Recommendations Under $100

### 7.1 Best Overall: Raspberry Pi 4 Model B (4GB)

| Item | Cost |
|------|------|
| Raspberry Pi 4B (4GB) | $55-65 |
| 32GB microSD (OS only) | $10 |
| 5V 3A USB-C power | $10 |
| **Total** | **~$75-85** |

**Pros**:
- Excellent community support
- Good floating-point performance with NEON
- Low power (~5-8W)
- Full Linux toolchain
- Can run Python/NumPy for algorithm development

**Cons**:
- No native ADC or high-speed data interface
- USB bus can saturate during heavy I/O

**SAR Performance**: 256×1024 Range-Doppler in ~80-150 ms (multi-threaded)

### 7.2 Best for Storage-Heavy Applications: Rock Pi 4B

| Item | Cost |
|------|------|
| Rock Pi 4B (4GB) | $65-75 |
| M.2 NVMe adapter | $10 |
| NVMe SSD 256GB | $40 |
| 32GB microSD (boot) | $10 |
| **Total** | **~$125-135** |

**Pros**:
- M.2 slot enables fast NVMe storage
- Higher sustained throughput for data acquisition
- USB 3.0 and Gigabit Ethernet

**Cons**:
- Slightly more expensive
- Smaller community than Pi

**Note**: Exceeds $100 if including NVMe, but still cheaper than Jetson Nano.

### 7.3 Best Value for Fixed-Point Processing: BeagleBone AI-64

| Item | Cost |
|------|------|
| BeagleBone AI-64 (4GB) | $70-80 |
| 32GB microSD | $10 |
| 5V 3A USB-C | $10 |
| **Total** | **~$90-100** |

**Pros**:
- Built-in C66x DSP (1-2 TFLOPS fixed-point)
- Excellent for embedded signal processing
- Programmable via MATLAB/Simulink or C
- Two PRU-ICSS for real-time I/O

**Cons**:
- Newer platform, less community support
- ARM cores less powerful than Cortex-A72 per MHz

### 7.4 Budget Pick: Orange Pi 5

| Item | Cost |
|------|------|
| Orange Pi 5 (4GB) | $55-65 |
| 64GB eMMC module | $15 |
| 5V 3A USB-C | $10 |
| **Total** | **~$80-90** |

**Pros**:
- RK3588 with 4×A76 cores (faster than Pi 4)
- 8K video output capability (overkill but shows power)
- Good value for floating-point

**Cons**:
- Less mature software ecosystem
- Some board variability

---

## 8. Summary Recommendations by Budget Tier

### Budget Tier 1: Under $100 Total System

**Architecture**: STM32F407 (data acquisition) + Raspberry Pi 4B (processing) + External USB SSD + ZED-F9P GPS + ICM-20948 IMU

| Component | Approximate Cost |
|-----------|-----------------|
| STM32F407 (already have) | $0 |
| Raspberry Pi 4B (4GB) | $60 |
| External SSD 500GB | $50 |
| ZED-F9P GPS | $150 |
| ICM-20948 IMU | $10 |
| **Total (excluding GPS)** | **~$120** |

**Note**: GPS is the most expensive component. If GPS must be under $50, use M8N and accept lower accuracy.

### Budget Tier 2: All-in-One SBC (No STM32)

If STM32 is not required and SBC can handle acquisition:

**Architecture**: BeagleBone AI-64 (handles PRU-based acquisition) + Single board handles everything

| Component | Cost |
|-----------|------|
| BeagleBone AI-64 | $75 |
| 64GB eMMC + microSD | $20 |
| External SSD 500GB | $50 |
| ZED-F9P GPS | $150 |
| ICM-20948 IMU | $10 |
| **Total (excluding GPS)** | **~$155** |

### Budget Tier 3: PC-Assist (If PC Available)

If a PC/laptop is available for final processing:

| Component | Cost |
|-----------|------|
| Raspberry Pi CM4 + Carrier | $60-80 |
| External SSD 500GB | $50 |
| M8N GPS (budget) | $40 |
| ICM-20948 IMU | $10 |
| **Total** | **~$160-190** |

PC handles the heavy Range-Doppler or back-projection; Pi handles acquisition and pre-processing.

---

## 9. Conclusions

1. **STM32F407 is NOT suitable for SAR image formation** but excels as a data acquisition controller for streaming raw ADC data.

2. **Raspberry Pi 4B (4GB) is the best single-board computer under $100** for SAR processing, offering a balance of performance, community support, and cost.

3. **PC-assist architecture is highly recommended** for hobbyists: STM32 for acquisition → SBC for preprocessing → PC for final image formation.

4. **Minimum motion compensation** requires ZED-F9P (or better) GPS and ICM-20948 (or better) IMU.

5. **Data storage** must be USB 3.0 SSD (not microSD) for raw data acquisition.

6. **For real-time SAR**, FPGA offers the best performance but requires significant expertise. Start with CM4 + PC assist before considering FPGA.

---

## References and Further Reading

- Carrara, W. G., et al. *Spotlight Synthetic Aperture Radar: Signal Processing Algorithms* (1995)
- Cumming, I. G., & Wong, F. H. *Digital Processing of Synthetic Aperture Radar Data* (2005)
- MIT OpenCourseWare: *Radar Systems Analysis and Design Using MATLAB*
- IEEE Xplore: "A Survey of Synthetic Aperture Radar Image Formation Algorithms"
- u-blox ZED-F9P Integration Manual
- STM32F4 DSP documentation (AN3998)

---

*Document Version: 1.0*  
*Last Updated: April 2026*  
*Research Level: Hobbyist/Maker*

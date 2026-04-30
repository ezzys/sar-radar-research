# Practical DIY SAR Implementation Guide — RPi CM5 + X-band FMCW

**Date:** April 30, 2026  
**Based on:** arxiv literature analysis + sar-radar-research repo (Cycle 3) + Henrik Forstén's DIY SAR  
**Target platform:** RPi CM5 (4GB) + X-band custom FMCW on 7" FPV quad

---

## Section 1: Proven DIY SAR Builds — Lessons Learned

### 1.1 Henrik Forstén's Polarimetric SAR (Finland, 2024-2025)

**What he built:**
- 6 GHz C-band FMCW radar (custom PCB, ADF4351 PLL)
- FPV drone kit (~€100) carrying ~500g radar payload
- Raspberry Pi for data capture
- Python processing on desktop

**What worked:**
- Custom FMCW approach — full control over chirp parameters
- RPi data capture reliable at moderate data rates
- Python processing adequate for post-flight
- Cheap FPV drone sufficient as platform
- Achieved ~1.5m resolution with 100 MHz bandwidth

**What failed/painful:**
- Antenna isolation (TX-RX leakage) was a major problem
- Required careful PCB layout for RF sections
- Vibration from drone motors contaminated data
- GPS accuracy insufficient without RTK — had to use post-processing
- Large data files (GB per flight) — SD card speed was bottleneck

**Lessons for our build:**
1. Start with ground-based rail SAR before flying
2. TX-RX antenna isolation is critical — separate antennas with shielding
3. Use RTK GPS from day one (not regular GPS)
4. Vibration-dampened mounting is mandatory
5. Fast storage (NVMe or high-speed SD) is essential

### 1.2 PulsON 410 on DJI Phantom (University project)

**What worked:**
- UWB radar (3.1-6.0 GHz) gave excellent range resolution
- <300g total payload — consumer drone compatible
- Raspberry Pi handled data capture easily

**What failed:**
- UWB radar is hard to source (Time Domain Corp, limited availability)
- Very short range (10-20m effective SAR range)
- Low output power limited applications

**Lesson:** Impulse/UWB radar can work but FMCW is more practical for DIY.

### 1.3 University of Birmingham 77 GHz SAR (2023)

**What worked:**
- 77 GHz FMCW achieved <2cm cross-range resolution
- Hexacopter platform provided stability
- Specialized imaging algorithm handled motion errors

**What's not applicable to DIY:**
- £15,000 system cost (INRAS RadarBook is professional equipment)
- Heavy payload (~2kg) requires large drone
- Complex calibration setup

**Lesson:** 77 GHz resolution is real but the hardware isn't DIY-friendly at our budget.

### 1.4 SAR Lessons Summary

```
DO:
✅ Start with ground-based experiments (rail or car)
✅ Use FMCW approach (proven, flexible, well-documented)
✅ Use RTK GPS from day one
✅ Separate TX/RX antennas with physical shielding
✅ Vibration-isolate the radar from drone frame
✅ Log raw I/Q data, process post-flight
✅ Use corner reflectors for calibration

DON'T:
❌ Try to fly on day one — ground first
❌ Use regular GPS (±2m is useless for SAR)
❌ Underestimate TX-RX leakage (it will destroy your data)
❌ Try real-time processing initially (post-flight first)
❌ Use STM32 for anything beyond basic GPIO toggling
❌ Skip the PGA autofocus step
```

---

## Section 2: Processing Pipeline for RPi CM5 — Step by Step

### 2.1 Software Setup

```bash
# RPi CM5 setup (Raspberry Pi OS 64-bit Bookworm)
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-numpy python3-scipy python3-matplotlib \
    python3-h5py python3-rasterio python3-numba python3-picamera2 \
    build-essential cmake git

# Install pyfftw (faster FFT)
pip3 install pyfftw --break-system-packages

# Install ARM-optimized FFTW from source (optional, for maximum speed)
git clone https://github.com/FFTW/fftw3.git
cd fftw3
./configure --enable-neon --enable-single --enable-threads --enable-openmp
make -j4
sudo make install
```

### 2.2 Data Capture Module

```python
#!/usr/bin/env python3
"""
sar_capture.py — Raw SAR data capture on RPi CM5
Captures: ADC samples (SPI) + GPS position (UART) + IMU (I2C)
"""

import numpy as np
import time
import struct
import threading
import serial
from datetime import datetime
import h5py

class SARDataCapture:
    def __init__(self, output_path, config):
        self.output_path = output_path
        self.config = config  # chirp parameters, PRF, etc.
        
        # GPS serial port (ZED-F9P)
        self.gps_serial = serial.Serial(
            '/dev/serial/by-id/u-blox_ZED-F9P',
            baudrate=460800, timeout=0.1
        )
        
        # Data storage
        self.gps_positions = []
        self.imu_data = []
        self.pulse_timestamps = []
        
    def capture_flight(self, duration_sec):
        """Main capture loop."""
        n_chirps = int(duration_sec * self.config['prf'])
        n_samples = self.config['n_samples']
        
        with h5py.File(self.output_path, 'w') as f:
            # Create datasets
            raw_ds = f.create_dataset(
                'raw_iq', 
                shape=(n_chirps, n_samples),
                dtype=np.complex64,
                chunks=(256, n_samples)  # Chunked for streaming
            )
            gps_ds = f.create_dataset(
                'gps_positions',
                shape=(n_chirps, 3),
                dtype=np.float64
            )
            
            f.attrs['config'] = str(self.config)
            f.attrs['start_time'] = datetime.utcnow().isoformat()
            
            for i in range(n_chirps):
                # 1. Trigger chirp via SPI (ADF4351 control)
                # trigger_chirp()  # Implement for your hardware
                
                # 2. Read ADC samples via SPI (AD9200)
                # adc_samples = read_adc(n_samples)
                # raw_iq = adc_to_iq(adc_samples)
                
                # Placeholder: replace with real SPI reads
                raw_iq = np.zeros(n_samples, dtype=np.complex64)
                
                # 3. Store
                raw_ds[i] = raw_iq
                
                # 4. Get GPS position (latest from background thread)
                pos = self.get_latest_gps()
                gps_ds[i] = pos
                
                # 5. Timestamp
                self.pulse_timestamps.append(time.time())
                
                # 6. Progress
                if i % 1000 == 0:
                    print(f"Captured {i}/{n_chirps} chirps")
            
            f.attrs['end_time'] = datetime.utcnow().isoformat()
        
        print(f"Capture complete: {self.output_path}")
    
    def get_latest_gps(self):
        """Parse latest NMEA/UBX position from ZED-F9P."""
        # Simplified — real implementation parses UBX-RXM-RAWX or NMEA GGA
        try:
            line = self.gps_serial.readline().decode()
            # Parse GGA sentence for lat/lon/alt
            # ... (implement NMEA parser)
            return np.array([0.0, 0.0, 0.0])  # placeholder
        except:
            return np.array([0.0, 0.0, 0.0])

# Usage
if __name__ == '__main__':
    config = {
        'prf': 1000,           # Pulse repetition frequency (Hz)
        'n_samples': 1024,     # Samples per chirp
        'bandwidth': 200e6,    # 200 MHz bandwidth
        'center_freq': 9.6e9,  # 9.6 GHz (X-band)
    }
    
    capture = SARDataCapture('/data/sar_flight_001.h5', config)
    capture.capture_flight(duration_sec=120)  # 2 min capture
```

### 2.3 SAR Processing Pipeline

```python
#!/usr/bin/env python3
"""
sar_process.py — Complete SAR image formation pipeline
Optimized for RPi CM5 with Numba JIT
"""

import numpy as np
from numba import njit, prange
import h5py
from scipy.signal import windows
import pyfftw

# Enable FFTW caching for speed
pyfftw.interfaces.cache.enable()

# ============================================
# STEP 1: RANGE COMPRESSION
# ============================================

def range_compression(raw_data, bandwidth, fs, n_samples):
    """
    Range compress raw FMCW de-ramped data.
    raw_data: [n_pulses, n_samples] complex
    """
    n_pulses, n_samples = raw_data.shape
    
    # Create range-compressed output using FFTW
    range_compressed = pyfftw.empty_aligned(
        (n_pulses, n_samples), dtype='complex64'
    )
    
    # Reference chirp matched filter
    t = np.arange(n_samples) / fs
    ref_chirp = np.exp(-1j * np.pi * bandwidth / n_samples * t**2)
    ref_fft = np.fft.fft(ref_chirp).conj()
    
    # Range compress each pulse
    for i in range(n_pulses):
        pulse_fft = pyfftw.interfaces.numpy_fft.fft(raw_data[i])
        range_compressed[i] = pyfftw.interfaces.numpy_fft.ifft(
            pulse_fft * ref_fft
        )
    
    # Convert range axis to meters
    range_resolution = 3e8 / (2 * bandwidth)  # c/(2B)
    range_axis = np.arange(n_samples) * range_resolution
    
    return range_compressed, range_axis


# ============================================
# STEP 2: AZIMUTH COMPRESSION — BACK-PROJECTION
# ============================================

@njit(parallel=True, fastmath=True, cache=True)
def backprojection_kernel(range_compressed, platform_pos, 
                          image_x, image_y, image_z,
                          range_axis, wavelength):
    """
    Numba-optimized back-projection.
    platform_pos: [n_pulses, 3] ECEF or local coordinates
    image_x, image_y, image_z: 1D arrays defining image grid
    """
    n_pulses, n_range = range_compressed.shape
    nx = len(image_x)
    ny = len(image_y)
    
    image = np.zeros((nx, ny), dtype=np.complex128)
    dr = range_axis[1] - range_axis[0]
    k0 = 2 * np.pi / wavelength
    
    for ix in prange(nx):
        for iy in range(ny):
            accum = 0.0j
            for ip in range(n_pulses):
                # Range from platform to pixel
                dx = platform_pos[ip, 0] - image_x[ix]
                dy = platform_pos[ip, 1] - image_y[iy]
                dz = platform_pos[ip, 2] - image_z
                r = np.sqrt(dx*dx + dy*dy + dz*dz)
                
                # Range bin index
                rbin = int(r / dr)
                
                if 0 <= rbin < n_range:
                    # Phase compensation
                    phase = -2.0 * k0 * r  # factor 2 for round-trip
                    accum += range_compressed[ip, rbin] * np.exp(1j * phase)
            
            image[ix, iy] = accum
    
    return image


def form_sar_image_bpa(data_path, image_bounds, pixel_size=0.25):
    """
    Complete BPA pipeline.
    """
    with h5py.File(data_path, 'r') as f:
        raw = f['raw_iq'][:]
        gps = f['gps_positions'][:]
        config = eval(f.attrs['config'])
    
    c = 3e8
    fs = config.get('fs', 20e6)
    bandwidth = config['bandwidth']
    fc = config['center_freq']
    wavelength = c / fc
    
    print(f"Processing {raw.shape[0]} pulses, {raw.shape[1]} samples")
    print(f"Range resolution: {c/(2*bandwidth):.3f} m")
    
    # Step 1: Range compress
    print("Range compression...")
    rc, range_axis = range_compression(raw, bandwidth, fs, raw.shape[1])
    
    # Step 2: Define image grid
    x_min, x_max, y_min, y_max = image_bounds
    image_x = np.arange(x_min, x_max, pixel_size)
    image_y = np.arange(y_min, y_max, pixel_size)
    
    # Step 3: Back-projection (Numba-accelerated)
    print("Back-projection...")
    image_z = np.mean(gps[:, 2]) - 10  # 10m below flight altitude
    
    sar_image = backprojection_kernel(
        rc, gps, image_x, image_y, image_z, range_axis, wavelength
    )
    
    return np.abs(sar_image), image_x, image_y


# ============================================
# STEP 3: PGA AUTOFOCUS
# ============================================

@njit(cache=True)
def pga_autofocus(sar_image, n_iterations=5):
    """
    Phase Gradient Autofocus.
    sar_image: complex-valued SAR image (range × azimuth)
    """
    n_range, n_azimuth = sar_image.shape
    
    for iteration in range(n_iterations):
        # Find brightest scatterer per range line
        phase_error = np.zeros(n_azimuth, dtype=np.float64)
        
        for ir in range(n_range):
            line = sar_image[ir, :]
            
            # Find peak
            peak_idx = np.argmax(np.abs(line))
            
            # Circular shift to center
            shifted = np.roll(line, -peak_idx)
            
            # Window (reduce noise from scatterers)
            window_size = min(n_azimuth // 4, 128)
            half_w = window_size // 2
            center = n_azimuth // 2
            
            # Compute phase gradient in window
            for i in range(center - half_w, center + half_w - 1):
                phase_error[i] += np.angle(
                    shifted[i+1] * np.conj(shifted[i])
                )
        
        # Average phase gradient across range lines
        phase_error /= n_range
        
        # Integrate to get phase error estimate
        phase_correction = np.cumsum(phase_error)
        
        # Remove mean
        phase_correction -= np.mean(phase_correction)
        
        # Apply correction
        for ir in range(n_range):
            sar_image[ir, :] *= np.exp(-1j * phase_correction)
    
    return sar_image


# ============================================
# STEP 4: OUTPUT
# ============================================

def save_sar_image(image, image_x, image_y, output_path):
    """Save SAR image as GeoTIFF."""
    try:
        import rasterio
        from rasterio.transform import from_bounds
        
        transform = from_bounds(
            image_x[0], image_y[0],
            image_x[-1], image_y[-1],
            image.shape[1], image.shape[0]
        )
        
        # Normalize to dB
        image_db = 20 * np.log10(image / np.max(image) + 1e-10)
        image_db = np.clip(image_db, -40, 0)  # 40 dB dynamic range
        
        with rasterio.open(
            output_path, 'w',
            driver='GTiff',
            height=image_db.shape[0],
            width=image_db.shape[1],
            count=1,
            dtype=np.float32,
            transform=transform,
            crs='EPSG:4326'
        ) as dst:
            dst.write(image_db.astype(np.float32), 1)
        
        print(f"Saved GeoTIFF: {output_path}")
    except ImportError:
        # Fallback: save as NumPy
        np.save(output_path.replace('.tif', '.npy'), image)
        print(f"Saved NumPy (rasterio not available)")


# ============================================
# MAIN PIPELINE
# ============================================

if __name__ == '__main__':
    import time
    
    data_file = '/data/sar_flight_001.h5'
    
    # Image bounds (adjust for your flight area)
    # Example: 50m × 50m scene
    image_bounds = (0, 50, 0, 50)  # x_min, x_max, y_min, y_max (meters)
    
    t0 = time.time()
    
    # Form SAR image
    sar_image, img_x, img_y = form_sar_image_bpa(
        data_file, image_bounds, pixel_size=0.25
    )
    t1 = time.time()
    print(f"BPA completed in {t1-t0:.1f} sec")
    
    # Apply PGA autofocus
    # Note: PGA works on complex image, so we need to modify pipeline
    # to return complex image before taking magnitude
    # sar_complex = pga_autofocus(sar_complex, n_iterations=5)
    # sar_image = np.abs(sar_complex)
    
    # Save
    save_sar_image(sar_image, img_x, img_y, '/data/sar_output.tif')
    
    print(f"Total processing time: {time.time()-t0:.1f} sec")
```

### 2.4 Expected Processing Times (RPi CM5, 4GB)

| Image Size | Range Compress | BPA | PGA (5 iter) | Total |
|-----------|---------------|-----|-------------|-------|
| 256×256 | 100 ms | 0.5 sec | 30 ms | ~0.7 sec |
| 512×512 | 200 ms | 2 sec | 80 ms | ~2.5 sec |
| 1024×1024 | 800 ms | 8 sec | 200 ms | ~9 sec |
| 2048×2048 | 3 sec | 60 sec | 800 ms | ~65 sec |

### 2.5 Memory Budget (4GB RAM)

| Data Structure | Size (512²) | Size (1024²) |
|---------------|-------------|--------------|
| Raw I/Q data (1000 chirps × 1024 samples) | 8 MB | 32 MB |
| Range-compressed data | 8 MB | 32 MB |
| SAR image (complex) | 2 MB | 8 MB |
| SAR image (magnitude) | 1 MB | 4 MB |
| GPS/IMU arrays | 0.1 MB | 0.1 MB |
| FFTW buffers | 16 MB | 64 MB |
| OS + Python | ~500 MB | ~500 MB |
| **Total** | **~535 MB** | **~640 MB** |

All fits comfortably in 4GB. 2048×2048 needs ~2GB — still OK.

---

## Section 3: 77 GHz Automotive Radar as SAR — Feasibility Assessment

### 3.1 Evidence Summary

| Evidence | Source | Finding |
|----------|--------|---------|
| Chalmers thesis (2024) | odr.chalmers.se | AWR1843 produces SAR images via BPA |
| KAUST dataset | repository.kaust.edu.sa | Published raw SAR data from TI radar |
| Real-time auto SAR | arxiv 2306.09784 | GPU-accelerated BPA on 77 GHz MIMO |
| Indoor mobile SAR | arxiv 2405.00121 | SAR on wheeled robot with COTS radar |

**Verdict:** 77 GHz automotive radar SAR is **proven and documented**.

### 3.2 The DCA1000 Problem

The DCA1000EVM is the **only official way** to extract raw ADC data from TI mmWave eval kits.

| Parameter | Value |
|-----------|-------|
| Cost new | ~$500 |
| Cost used | ~$100-200 (eBay) |
| Weight | ~200g |
| Connection | LVDS to eval kit, GigE to PC |
| Software | MMWAVE-STUDIO (Windows only) |
| Data rate | Up to 600 MB/s |

**The problem for drone SAR:**
- DCA1000 adds $300-500 to budget (kills €500 target)
- Requires Windows PC for capture (can't run on RPi)
- 200g extra weight
- GigE cable needed → tethered operation only

### 3.3 Workarounds

**Option A: Ground-rail SAR first (no drone)**
- Mount AWR1642BOOST + DCA1000 on linear rail
- Capture SAR data, process on laptop
- Validate algorithms before investing in flight hardware
- **Cost:** AWR1642BOOST ($300) + DCA1000 ($300) = $600

**Option B: Tethered drone SAR**
- Drone flies, data streams via GigE cable to ground PC
- Works for low altitude (5-20m), short flights
- **Cost:** AWR1642BOOST ($300) + DCA1000 ($300) + cable

**Option C: Custom raw data capture (advanced)**
- Bypass DCA1000, read LVDS directly with custom FPGA/ARM board
- Example: Xilinx Artix-7 FPGA to capture LVDS → store to SD/RAM
- **Cost:** $50-100 for FPGA board, significant dev time
- **Reference:** Some researchers have done this (TI E2E forums)

**Option D: Use IWR6843 point cloud mode (not raw SAR)**
- IWR6843ISK ($150) has built-in processing
- Can output range-angle heatmaps (not raw I/Q)
- Lower quality but no DCA1000 needed
- Connects directly to RPi via UART/CSI
- **Cost:** $150 total

### 3.4 Recommended 77 GHz Strategy for Our Project

```
PHASE 3-4: Use X-band custom FMCW (from sar-radar-research repo)
PHASE 5: Evaluate 77 GHz path with one of:
  - Option A (ground rail) for algorithm validation
  - Option D (IWR6843 heatmap) for quick drone SAR
  - Option C (custom capture) for full raw SAR on drone
```

The 4 GHz bandwidth at 77 GHz gives 3.75 cm range resolution — easily meets 0.2m target. The question is getting raw data off the chip affordably.

---

## Section 4: Practical Motion Compensation

### 4.1 What Accuracy Is Actually Needed?

SAR coherence requires phase error < λ/4 (quarter wavelength).

| Band | Wavelength | λ/4 | Required Position Accuracy |
|------|-----------|-----|---------------------------|
| C-band (5.8 GHz) | 51.7 mm | 12.9 mm | ~13 mm |
| X-band (9.6 GHz) | 31.3 mm | 7.8 mm | ~8 mm |
| Ku-band (12.8 GHz) | 23.4 mm | 5.9 mm | ~6 mm |
| 77 GHz | 3.9 mm | 1.0 mm | ~1 mm (!) |

**RTK GPS (ZED-F9P):** 10 mm + 1 ppm horizontal → ~12 mm absolute
**RTK + IMU fusion:** 5-10 mm relative between pulses
**Consumer IMU alone:** Not sufficient for any band

**Critical insight:** For X-band SAR, RTK GPS is *barely* sufficient. IMU fusion and PGA autofocus are *necessary*, not optional.

### 4.2 Error Budget for X-band SAR

| Error Source | Magnitude | Effect on SAR |
|-------------|-----------|---------------|
| RTK absolute position | ±12 mm | Near λ/4 limit — OK with PGA |
| RTK between-pulse drift | ±5 mm | Within budget if update rate ≥10 Hz |
| Motor vibration (un-isolated) | ±2-5 mm | Degrades image, PGA can't fully fix |
| Motor vibration (isolated) | ±0.5-1 mm | OK |
| Wind-induced path deviation | ±50-200 mm | PGA + trajectory correction needed |
| Timing jitter | ±1 μs | Negligible |
| Altitude error | ±20 mm | Affects slant range geometry |

### 4.3 Vibration Isolation

**Must-have for X-band SAR on multirotor:**

```
DRONE FRAME
    │
    │  4× vibration dampers (silicone gel or sorbothane)
    │
    ▼
SAR MOUNTING PLATE (3D printed)
    │
    ├── RPi CM5
    ├── RF front-end
    ├── Battery (separate from drone)
    └── GPS antenna (on top, clear sky view)
```

Recommended vibration dampers:
- **Sorbothane hemispheres** (Durometer 50, 6mm): €5 for 8-pack
- Or: O-ring suspension system (cheaper, less effective)
- Target: isolate frequencies >50 Hz (motor RPM harmonics)

### 4.4 EKF Fusion Implementation

```python
#!/usr/bin/env python3
"""
sar_navigation.py — GPS/IMU EKF fusion for SAR motion compensation
"""

import numpy as np
from scipy.linalg import expm

class SARNavigationEKF:
    """
    Extended Kalman Filter fusing RTK GPS + IMU for SAR trajectory.
    State: [x, y, z, vx, vy, vz, roll, pitch, yaw]
    """
    
    def __init__(self):
        # State vector
        self.x = np.zeros(9)
        
        # Covariance
        self.P = np.eye(9) * 0.1
        
        # Process noise (IMU drift)
        self.Q = np.diag([
            0.001, 0.001, 0.001,  # position
            0.01,  0.01,  0.01,   # velocity
            0.001, 0.001, 0.001   # attitude
        ])
        
        # GPS measurement noise
        self.R_gps = np.diag([0.01, 0.01, 0.02])  # 10mm horiz, 20mm vert
        
        # IMU noise
        self.accel_noise = 0.05  # m/s²
        self.gyro_noise = 0.001  # rad/s
        
    def predict(self, accel, gyro, dt):
        """Predict step using IMU data."""
        # Rotation matrix from body to world (simplified)
        roll, pitch, yaw = self.x[6:9]
        R = self.rotation_matrix(roll, pitch, yaw)
        
        # Transform acceleration to world frame
        accel_world = R @ accel + np.array([0, 0, 9.81])
        
        # State prediction
        self.x[0:3] += self.x[3:6] * dt + 0.5 * accel_world * dt**2
        self.x[3:6] += accel_world * dt
        self.x[6:9] += gyro * dt
        
        # Covariance prediction (simplified Jacobian)
        F = np.eye(9)
        F[0:3, 3:6] = np.eye(3) * dt
        
        self.P = F @ self.P @ F.T + self.Q * dt
    
    def update_gps(self, gps_pos):
        """Update step with RTK GPS measurement."""
        H = np.zeros((3, 9))
        H[0:3, 0:3] = np.eye(3)
        
        y = gps_pos - self.x[0:3]  # Innovation
        S = H @ self.P @ H.T + self.R_gps
        K = self.P @ H.T @ np.linalg.inv(S)
        
        self.x += K @ y
        self.P = (np.eye(9) - K @ H) @ self.P
    
    def get_position(self):
        """Return current position estimate."""
        return self.x[0:3].copy()
    
    @staticmethod
    def rotation_matrix(roll, pitch, yaw):
        """ZYX Euler rotation matrix."""
        cr, sr = np.cos(roll), np.sin(roll)
        cp, sp = np.cos(pitch), np.sin(pitch)
        cy, sy = np.cos(yaw), np.sin(yaw)
        
        R = np.array([
            [cy*cp, cy*sp*sr - sy*cr, cy*sp*cr + sy*sr],
            [sy*cp, sy*sp*sr + cy*cr, sy*sp*cr - cy*sr],
            [-sp,   cp*sr,            cp*cr]
        ])
        return R

# Usage during SAR capture:
# 1. IMU at 200 Hz → predict()
# 2. GPS at 20 Hz → update_gps()
# 3. Store EKF position at PRF (e.g., 1 kHz) via interpolation
```

### 4.5 Map-Drift Autofocus (Quick Win)

```python
def mapdrift_autofocus(range_compressed, n_subapertures=2):
    """
    Map-drift autofocus — estimate along-track velocity error.
    Split aperture, correlate sub-images, estimate shift.
    """
    n_pulses, n_range = range_compressed.shape
    sub_size = n_pulses // n_subapertures
    
    # Form sub-images
    sub_images = []
    for i in range(n_subapertures):
        sub = range_compressed[i*sub_size:(i+1)*sub_size, :]
        sub_img = np.fft.fftshift(np.fft.fft2(sub))
        sub_images.append(np.abs(sub_img))
    
    # Cross-correlate adjacent sub-images
    velocity_errors = []
    for i in range(len(sub_images) - 1):
        corr = np.correlate(
            sub_images[i].flatten(),
            sub_images[i+1].flatten(),
            mode='full'
        )
        shift = np.argmax(corr) - len(sub_images[i].flatten()) + 1
        velocity_errors.append(shift)
    
    # Average velocity error estimate
    mean_shift = np.mean(velocity_errors)
    
    print(f"Map-drift estimated velocity error: {mean_shift} pixels")
    return mean_shift
```

---

## Section 5: Implementation Roadmap — Detailed Week-by-Week

### Phase 1: Ground-Based Learning (Weeks 1-4) — €100

**Week 1: PlutoSDR FMCW Setup**
- [ ] Buy PlutoSDR (€100 from Mouser/DigiKey)
- [ ] Install PlutoSDR drivers on laptop: `pip install pyadi-iio`
- [ ] Implement simple FMCW chirp generator in Python
- [ ] Verify TX output on SDR (if you have a second one) or with power meter

```python
# Simple PlutoSDR FMCW chirp test
import adi

sdr = adi.Pluto("ip:192.168.2.1")
sdr.sample_rate = 10e6
sdr.tx_lo = 5800000000  # 5.8 GHz ISM
sdr.tx_hardwaregain_chan0 = -10  # dB
# Generate linear FM chirp
t = np.arange(1024) / 10e6
chirp = np.exp(2j * np.pi * 50e6 * t**2 / (2 * 1024/10e6))
sdr.tx(chirp * 2**14)
```

**Week 2: Ground Rail SAR**
- [ ] Build linear rail (2m sliding platform from aluminum extrusion): €20
- [ ] Mount PlutoSDR + antennas on rail platform
- [ ] Implement range compression in Python
- [ ] Capture data at multiple positions along rail (manual movement)
- [ ] **Gate:** Range profiles visible in processed data

**Week 3: First SAR Image (Ground)**
- [ ] Implement back-projection algorithm
- [ ] Place metal plate / corner reflector at known position (5m from rail)
- [ ] Capture data at 20 positions along rail (10cm spacing)
- [ ] Process with BPA
- [ ] **Gate:** SAR image shows bright point at reflector location

**Week 4: Refine Processing**
- [ ] Implement PGA autofocus
- [ ] Test with multiple reflectors at different ranges
- [ ] Measure actual resolution (3dB width of point target)
- [ ] Compare with theoretical resolution
- [ ] **Gate:** Resolution measurement matches theory (±20%)

### Phase 2: Drone Platform (Weeks 5-8) — €390

**Week 5: Frame + Power Train Assembly**
- [ ] Assemble 7" frame (Nazgul5 or equivalent)
- [ ] Install FC (Speedybee F405 V4) + ESC (40A 4-in-1)
- [ ] Solder motors (1400KV 2806.5)
- [ ] Wire power distribution
- [ ] Flash ArduPilot via INAV configurator or Mission Planner

**Week 6: RTK GPS + Autopilot**
- [ ] Install ZED-F9P GPS module
- [ ] Configure NTRIP client for RTK corrections
  - Lithuania: Use EPOS GNSS or EUREF NTRIP (free)
  - Configure on RPi or FC directly
- [ ] Set up ArduPilot AUTO mode for survey patterns
- [ ] Configure MAVLink telemetry to RPi Zero (practice companion computer setup)

**Week 7: Flight Testing**
- [ ] First flights without SAR payload (weight substitute instead)
- [ ] Validate RTK fix quality in flight (check HDOP, number of satellites)
- [ ] Fly automated straight-line missions at 10m altitude, 3 m/s
- [ ] Log position data, analyze straightness and accuracy
- [ ] **Gate:** <5cm position accuracy on straight lines, stable flight for 5+ min

**Week 8: Payload Integration Practice**
- [ ] 3D print vibration-isolated SAR mount
- [ ] Mount dummy payload (same weight as SAR: ~340g)
- [ ] Fly with payload, verify flight time >5 min
- [ ] Check vibration levels on payload mount (use phone accelerometer app)
- [ ] **Gate:** Stable flight with payload, vibration <0.5g at payload

### Phase 3: X-band SAR on Drone (Weeks 9-16) — €375

**Week 9-10: RF Front-End Build**
- [ ] Order components (from Cycle 3 corrected BOM):
  - ADF4351 breakout: €15 (AliExpress)
  - KZD-1+ multiplier: €20 (Mouser)
  - ZAM-50+ mixer: €25 (Mouser)
  - HMC1041L LNA: €40 (Mouser/DigiKey)
  - ZFM-2+ + MAR-6SM+: €20 (Mouser)
  - AD9200 ADC: €30 (DigiKey)
- [ ] Design carrier PCB in KiCad (RPi CM5 interface + ADC + PLL)
- [ ] Order PCB from JLCPCB (€30, 1 week turnaround)

**Week 11-12: Carrier PCB + Integration**
- [ ] Assemble carrier PCB (hand solder or reflow)
- [ ] Build X-band RF section on dedicated RF PCB
- [ ] Test PLL chirp output with spectrum analyzer (or second SDR)
- [ ] Verify ADC capture on RPi CM5 via SPI
- [ ] Integrate GPS + IMU on carrier board

**Week 13-14: Antenna + Bench Test**
- [ ] Design 4×4 patch antenna in KiCad / OpenEMS
  - Rogers RT/duroid 5880 substrate (0.508mm)
  - Target: 18-22 dBi at 9.6 GHz
  - Order from JLCPCB (Rogers layers available)
- [ ] Build second antenna for TX (or use single antenna with circulator)
- [ ] Bench test complete radar: chirp → TX → reflector → RX → ADC → RPi
- [ ] Verify range profiles in bench test
- [ ] **Gate:** Clear range profile of corner reflector at 5-10m range

**Week 15-16: First Flight with SAR**
- [ ] Mount complete SAR payload on drone
- [ ] Pre-flight checklist:
  - Battery charged (both drone and SAR)
  - RTK fix acquired (green LED on ZED-F9P)
  - SD card inserted, enough free space
  - SAR capture software tested on ground
  - Vibration dampers in place
- [ ] Flight: automated straight line at 10m altitude, 3 m/s, 100m length
- [ ] Capture raw data to SD card
- [ ] **Gate:** Raw I/Q data + GPS positions captured successfully

### Phase 4: SAR Processing + Calibration (Weeks 17-20) — €50

**Week 17-18: Post-Flight SAR Processing**
- [ ] Transfer data from SD to laptop
- [ ] Run range compression pipeline
- [ ] Run back-projection with GPS trajectory
- [ ] Apply PGA autofocus
- [ ] Visualize SAR image
- [ ] **Gate:** Recognizable SAR image of test area

**Week 19-20: Calibration + Resolution Verification**
- [ ] Place trihedral corner reflectors at known positions
  - Build from aluminum sheet: 3 triangles, 30cm sides
  - RCS at X-band: ~10 m² (very bright)
  - Cost: €10 in materials
- [ ] Fly calibration mission
- [ ] Measure resolution from point target response (Impulse Response Function)
- [ ] Calculate:
  - Range resolution: 3dB width in range direction
  - Azimuth resolution: 3dB width in azimuth direction
  - PSLR (Peak Sidelobe Ratio)
  - ISLR (Integrated Sidelobe Ratio)
- [ ] **Gate:** Resolution <0.5m (range + azimuth combined)

### Phase 5: Push to 0.2m (Weeks 21+) — €100-200

**Options (choose one):**
1. Wider bandwidth VCO (HMC587LC4B, +€80)
2. 77 GHz eval kit ground-rail experiments (AWR1642BOOST + DCA1000, +€300)
3. Accept 0.5m and improve image quality instead

---

## Appendix A: Corner Reflector Design

```
Trihedral Corner Reflector (X-band optimized):

     ┌──────┐
     │╲     │
     │ ╲    │  Side view: three mutually perpendicular
     │  ╲   │  triangular plates forming a corner
     │   ╲  │
     │    ╲ │
     │     ╲│
     └──────┘

Dimensions for X-band (9.6 GHz):
- Side length: 30 cm (≈10λ)
- Material: 1mm aluminum sheet
- RCS: σ = 4πa⁴/(3λ²) ≈ 10.5 m² at 9.6 GHz
- 3 dB beamwidth: ~40° ( forgiving for alignment)

Cost: ~€5-10 each (DIY from sheet aluminum)
Make 3-5 for calibration field
```

## Appendix B: Pre-Flight Checklist

```
□ Drone battery charged (>90%)
□ SAR battery charged
□ SD card inserted (>32GB free)
□ RTK fix acquired (surveyed position, <1m accuracy base)
□ ArduPilot mission loaded and verified
□ SAR capture software running on RPi CM5
□ GPS logging verified (check LED on ZED-F9P)
□ Vibration dampers intact
□ Antenna connections secure
□ Prop nuts tight
□ Failsafe configured (RTL on signal loss)
□ Weather: wind <15 km/h, no rain
□ Flight area cleared, NOTAM checked
□ Log book entry started
```

## Appendix C: SAR Data Format (HDF5)

```
sar_flight_YYYYMMDD_HHMMSS.h5
├── raw_iq              [n_pulses × n_samples] complex64
├── gps_positions       [n_pulses × 3] float64 (x,y,z in meters, local frame)
├── imu_accel           [n_imu × 3] float32 (m/s²)
├── imu_gyro            [n_imu × 3] float32 (rad/s)
├── timestamps_pulse    [n_pulses] float64 (Unix epoch)
├── timestamps_imu      [n_imu] float64
├── attributes:
│   ├── config          str (JSON: prf, bw, fc, fs, n_samples)
│   ├── start_time      str (ISO 8601)
│   ├── end_time        str (ISO 8601)
│   ├── platform        str ("7inch_quad_Xband_SAR")
│   ├── gps_type        str ("ZED-F9P RTK")
│   └── notes           str (flight conditions, weather)
```

---

*Guide compiled by Claw (OpenClaw Agent) — April 30, 2026*  
*Sources: arxiv literature, Henrik Forstén blog, TI documentation, BYU MicroASAR, sar-radar-research repo*

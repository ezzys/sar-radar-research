# DIY Synthetic Aperture Radar (SAR) — Hobby Project Research

## Project Overview
**Goal:** Build a functional Synthetic Aperture Radar (SAR) as a DIY/hobby project.
**Constraints:**
- Very limited budget (cost-sensitive)
- STM32F407VG available (ARM Cortex-M4, 180MHz, 192KB RAM, 1MB Flash)
- Not for commercial use
- Hobby/research only

## Target Applications
1. On-ground (static) SAR imaging
2. Drone-mounted SAR (UAV)
3. Both:isar principles are similar; drone adds motion compensation complexity

## Research Questions (to be answered)
1. **Antennas**: What type, size, frequency, bandwidth for X-band or C-band DIY SAR?
2. **RF Frontend**: Trade-offs between direct sampling vs. heterodyne, required dynamic range, ADC specs
3. **Processor**: Can STM32F407 handle SAR processing? If not, what's the minimum viable alternative?
4. **Signal Processing**: Core algorithms (Range-Doppler, back-projection), computational load, PC-assist strategy
5. **Hobby References**: What have other hobbyists actually built? What worked/what failed?
6. **Budget Breakdown**: Low-cost component selection, what's skippable vs. critical
7. **Legal/Frequency**: ISM bands, FCC Part 15 vs. amateur radio licensing

## Initial Hardware Candidates
- STM32F407VG (on-hand)
- Raspberry Pi CM4 / Jetson Nano (for heavier processing)
- AD9361 (SDR) — likely too expensive for budget
- LMX2572 + LMS7002 — budget RF chains

## Deliverable
A practical design document with:
- Hardware BOM with cost estimates
- Antenna design specs
- Processor selection rationale
- Processing pipeline architecture
- Timeline / phased build approach

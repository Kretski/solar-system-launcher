Priority document published on Zenodo.
DOI: 10.5281/zenodo.18898270
AETHER-NT v4.4: Aerospace-Grade Integrated Pipeline

KEY RESULTS:
• Detection Rate: 100% across all scenarios
• PMD: 0%
• SEU Robustness: 0 crashes after 6 NaN/Inf/bit-flip injections
• Mode Thrashing: 0 transitions (fully stabilized)
• Noise Robustness: FPR < 40% at σ=0.30
• Temperature Drift: Adaptive thresholds track rising noise floor

COMPONENTS:
• AZURO Compass: environment model with 3 feedback loops
• N(t) v6 Detector: N=k·(G/G₀)·(t/τ)² with persistence filter
• Aether Optimizer: stabilized with hysteresis and dwell time
• Three feedback loops: N(t)↔AZURO↔Aether

STATUS: Ready for hardware validation and publication.

 AETHER-NT: An Adaptive Error Detection and Record Reduction Pipeline for Onboard AI with Limited Emission

**DOI:** 10.5281/zenodo.18898270 (original record)
**Version:** v4.3 – March 2026
**License:** CC-BY-4.0 (documentation and benchmarks); source code and algorithms – patent pending, available upon request for validation
**Authors:** Dimitar Kretski
**Contact:** kretski1@gmail.com | LinkedIn: dimitar-kretski-071118b6

## Overview

AETHER-NT is a proof-of-concept pipeline for **autonomous control of AI on board deep space probes**, where real-time control from Earth is physically impossible (signal delay up to 4+ hours at Neptune).

The system integrates four components:

1. **Solar Launch Optimizer** – trajectory with minimum ΔV (educational tool)

2. **AZURO Compass** – real-time assessment of the structural environment of the Solar System (ρ(r,τ), Φ)

3. **N(t) Instability Detector** – adaptive anomaly detector from IMU data

4. **Aether Space-Grade Optimizer** – environment-sensitive optimizer to reduce memory writes under thermal, radiation and energy stress

The goal is **autonomous protection and continuation of training/inference** under extreme conditions, without human intervention.

*
AZURO Compass, N(t) Detector, Ether Optimizer – algorithms and formulas are proprietary. Simulations and benchmark codes are provided for validation upon request only.

## Key Results (v4.3)

**Aether Optimizer** – 675 trainings (ResNet18 + SmallCNN, CIFAR-10):

| Scenario | Record Reduction | Accuracy Price | Note |
|---------------------|-----------------|---------------|-----------------------------|
| THERMAL (87°C) | 35.6% | −0.51% | p > 0.05 (Welch t-test) |
| COMBINED (heat + radiation + power) | 27.7% | −0.53% | p > 0.05 |

**Fault Detection and Isolation (FDIR-class tests)**:

- Detection Rate (DR): **100%** in all scenarios (Standard, Avalanche-15, Rapid Fire, High Noise, Worst Case)
- Missed Detection (PMD): **0%**
- False Positive Rate (FPR) at low noise (σ ≤ 0.10): **≤ 1.8%**
- SEU/Radiation Resistance: **0 crashes** at 6 NaN/Inf/SEU injections

**TEMPERATURE DRIFT/AGEING OF MEMS IMU** (σ: 0.01 → 0.12):

- Phase 1–2 (σ ≤ 0.08): FP = **0%**
- Phase 3 (σ 0.08–0.12): FP = **14.5%** (acceptable with aging/thermal run)
- Real Defect (σ ≈ 0.09): **detected** at step ~400
<img width="2101" height="1354" alt="aether_nt_v43_tempdrift" src="https://github.com/user-attachments/assets/67456ad5-c2cf-4ac8-9a03-c18e6b1888c4" />

The adaptive threshold follows the increasing noise floor and minimizes the FP to the limit of normal noise.

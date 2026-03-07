Priority document published on Zenodo.
DOI: 10.5281/zenodo.18898270

AZURO Compass + N(t) Detector + AIDSP pipeline.
# AIDSP — Autonomous Intelligent Deep-Space Probe
### *Adaptive AI pipeline for autonomous deep-space missions*

> At Neptune, a signal takes ~4 hours to reach Earth.
> Real-time ground control is physically impossible.
> The probe must read its environment and protect its own intelligence — alone.

---

## Four integrated components

**1. Solar Launch Optimizer** — *The Path*
Calculates optimal transfer trajectories with minimum ΔV.
Portable `.exe` — download from Releases.

> ⚠️ Educational tool. For real missions use SPICE / GMAT / STK.

---

**2. AZURO Compass** — *The Eyes*
A coordinate system that describes the structural environment
of the solar system as a function of position and time.
Outputs a real-time environmental index to the AI core.

*Patent pending. Technical details available on request.*

---

**3. N(t) Instability Detector** — *The Alarm*
A lightweight adaptive index computed from onboard IMU data.
Detects anomalies in real-time and triggers immediate
protective response in the AI core.
Designed for FPGA / embedded deployment.

*Patent pending. Technical details available on request.*

---

**4. Aether Space-Grade Optimizer** — *The Brain*
Sparse runtime optimizer for onboard AI in extreme environments.
Five autonomous operational modes governed by environmental triggers.
Reduces memory write activity proportionally to available resources —
automatically, bidirectionally, with full autonomous recovery.

*Patent pending. Technical details available on request.*

---

## Validated results

Aether optimizer — **675 training runs**, ResNet18 + SmallCNN, CIFAR-10:

| Scenario | Write Reduction | Accuracy Cost |
|----------|-----------------|---------------|
| THERMAL (87°C) | **35.6%** | -0.51% |
| COMBINED (heat + radiation + power) | **27.7%** | -0.53% |

> Statistical validation: Welch's t-test, p>0.05 for primary scenarios.
> Full technical report available on request.

---<img width="2000" height="1501" alt="aidsp_v3_protected" src="https://github.com/user-attachments/assets/0e02fe39-0b38-45e9-b039-3b6b2bc5913a" />


## Current status

```
Aether optimizer      → validated in laboratory (675 runs)
AZURO + N(t) + AIDSP  → validated in simulation
Hardware (FPGA)       → next step
Flight-tested         → not yet
```

> We do not claim superiority over existing flight systems.
> The contribution is the integrated adaptive pipeline concept.

---

## Patent status

> 🔒 **Patent Pending** — Bulgarian National Patent, filed 05.12.2025
> PCT filing planned before 05.12.2026

Algorithms, formulas, and source code are **not public**.
Simulation and benchmark scripts available **on request only**
for validation purposes.

---

## Contact & Partnerships

Open for partnerships in **Aerospace AI**, **Edge Computing**, and **On-Orbit Learning**.

- **Dimitar Kretski**
- Email: kretski1@gmail.com
- LinkedIn: [dimitar-kretski-071118b6](https://linkedin.com/in/dimitar-kretski-071118b6)

*Technical report v3.0 and full pipeline documentation available on request.*


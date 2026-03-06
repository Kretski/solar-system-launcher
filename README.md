# Aether Space-Grade Optimizer

**Thermal & Radiation-Aware ML Optimizer for Space Deployment**

> Runtime training optimizer for AI systems operating under degraded space conditions —
> heat, radiation, and power failure. No comparable open system exists.

---

## 1. Solar Launch Optimizer *(The Navigator)*

Windows desktop application for calculating optimal Hohmann transfer windows
from Earth to: Mercury, Venus, Mars, Jupiter, Saturn, Uranus, Neptune.

**Key features:**
- Minimum ΔV launch windows
- 4 perturbation modes: TCM, Solar Pressure, Pioneer Anomaly, Penalty for bad windows
- Orbit visualization + ΔV timeline
- Portable `.exe` — no installation required

> ⚠️ Educational tool. For real missions use SPICE / GMAT / STK.

→ Download from **Releases** → `SolarLaunchOptimizer.exe`

---

## 2. Aether Space-Grade Optimizer *(The Brain and the Shield)*

Sparse runtime optimizer for onboard AI in extreme environments.
Validated across **675 training runs** on two architectures, five mission scenarios,
and three random seeds.

### How it works

Five operational modes governed by three independent environmental triggers
(temperature, power, radiation). A sparse update gate **σ** reduces memory
write activity proportionally to available resources — automatically,
bidirectionally, with full recovery to NORMAL mode.

```
NORMAL   σ=0.00  T<50°C   P>88%   R<5/ep    → full training
SAFE     σ=0.10  T≥50°C   P≤88%   R≥5/ep    → ~9% write reduction
REDUCED  σ=0.30  T≥65°C   P≤70%   R≥15/ep   → ~27% write reduction
POWER    σ=0.50  T≥75°C   P≤50%   R≥30/ep   → ~45% write reduction
SURVIVAL σ=0.70  T≥85°C   P≤28%   R≥60/ep   → ~60% write reduction
```

### Validated results

**ResNet18 — 11.2M parameters, CIFAR-10, 3 seeds × 25 epochs**

| Scenario | Accuracy | Write Reduction | vs Baseline | Std |
|----------|----------|-----------------|-------------|-----|
| NOMINAL | 91.27% | 0.4% | — | ±0.07 |
| **THERMAL** (87°C SURVIVAL) | **90.76%** | **35.6%** | **-0.51%** | **±0.09** |
| **COMBINED** (heat + radiation + power) | **90.74%** | **27.7%** | **-0.53%** | **±0.04** |

**SmallCNN — 5-scenario validation, 3 seeds × 25 epochs**

| Scenario | Write Reduction | vs Baseline | p-value |
|----------|-----------------|-------------|---------|
| THERMAL | 28.8% | -0.07% | 0.713 ✅ ns |
| ECLIPSE | 18.3% | +0.40% | 0.137 ✅ ns |
| COMBINED | 25.1% | -0.76% | 0.031 * |

> ✅ THERMAL and ECLIPSE: no statistically significant accuracy degradation (Welch's t-test)
> 🟡 COMBINED: significant but <1% accuracy cost — acceptable for space deployment

**Scaling confirmed:** larger model → stronger write reduction effect
(ResNet18: 35.6% vs SmallCNN: 28.8% in THERMAL scenario)

**COMBINED worst-case timeline (ResNet18, seed=42):**
13 mode transitions including 1 radiation bit-flip absorbed mid-training —
full autonomous recovery to NORMAL by epoch 25.

### Why this matters

Standard optimizers write to memory at full rate regardless of thermal budget,
power availability, or radiation environment.
Aether reduces write activity proportionally to available resources —
enabling ML training to continue under degraded conditions where
standard optimizers would exhaust power or thermal budgets.

```
Inference optimizers (TensorRT, ONNX):  compress model before deployment
Aether:                                 keeps training viable during deployment
```

---

## Patent status

> 🔒 Patent Pending — Bulgarian National Patent, filed 05.12.2025
> PCT filing planned before 05.12.2026

Algorithms and source code are not public.
Benchmark scripts available on request for validation purposes.

---

## Synergy

```
Solar Launch Optimizer   →  path of least resistance (minimum ΔV)
Aether Optimizer         →  intelligence retention (maximum accuracy at minimum power)
```

---

## Contact & Partnerships

Open for partnerships in **Aerospace AI**, **Edge Computing**, and **On-Orbit Learning**.

- **Dimitar Kretski**
- Email: kretski1@gmail.com
- LinkedIn: [dimitar-kretski-071118b6](https://linkedin.com/in/dimitar-kretski-071118b6)

*Technical report v3.0 available on request.*





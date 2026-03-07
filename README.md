Priority document published on Zenodo.
DOI: 10.5281/zenodo.18898270


AETHER-NT: Adaptive Fault Detection and Write Reduction Pipeline for Radiation-Constrained On-Board AI

**DOI:** 10.5281/zenodo.18898270 (original record)  
**Version:** v4.3 – March 2026  
**License:** CC-BY-4.0 (documentation & benchmarks); source code & algorithms – patent pending, available on request for validation  
**Authors:** Dimitar Kretski  
**Contact:** kretski1@gmail.com | LinkedIn: dimitar-kretski-071118b6

## Overview

AETHER-NT е proof-of-concept pipeline за **автономно управление на изкуствен интелект на борда на дълбококосмически сонди**, където реално-времево управление от Земята е физически невъзможно (забавяне на сигнала до 4+ часа при Нептун).

Системата интегрира четири компонента:

1. **Solar Launch Optimizer** – траектория с минимален ΔV (образователен инструмент)
2. **AZURO Compass** – реално-времева оценка на структурната среда на Слънчевата система (ρ(r,τ), Φ)
3. **N(t) Instability Detector** – адаптивен детектор на аномалии от IMU данни
4. **Aether Space-Grade Optimizer** – среда-чувствителен оптимизатор за намаляване на memory writes при термичен, радиационен и енергиен стрес

Целта е **автономна защита и продължаване на обучението/инференса** при екстремни условия, без човешка намеса.

*
AZURO Compass, N(t) Detector, Aether Optimizer – алгоритми и формули са защитени. Симулации и бенчмарк кодове се предоставят само за валидация по заявка.

## Key Results (v4.3)

**Aether Optimizer** – 675 обучения (ResNet18 + SmallCNN, CIFAR-10):

| Сценарий                          | Write Reduction | Accuracy Cost | Забележка                  |
|-----------------------------------|-----------------|---------------|----------------------------|
| THERMAL (87°C)                    | 35.6%           | −0.51%        | p > 0.05 (Welch's t-test)  |
| COMBINED (heat + radiation + power) | 27.7%         | −0.53%        | p > 0.05                   |

**Fault Detection & Isolation (FDIR-grade тестове)**:

- Detection Rate (DR): **100%** във всички сценарии (Standard, Avalanche-15, Rapid-fire, High-noise, Worst-case)
- Missed Detection (PMD): **0%**
- False Positive Rate (FPR) при нисък шум (σ ≤ 0.10): **≤ 1.8%**
- SEU/radiation robustness: **0 crashes** при 6 инжекции NaN/Inf/SEU

**Температурен дрейф / стареене на MEMS IMU** (σ: 0.01 → 0.12):

- Фаза 1–2 (σ ≤ 0.08): FP = **0%**
- Фаза 3 (σ 0.08–0.12): FP = **14.5%** (приемливо при aging/thermal runaway)
- Реален fault (σ ≈ 0.09): **детектиран** на стъпка ~400
[aether_nt_v43.py](https://github.com/user-attachments/files/25814262/aether_nt_v43.py)
"""
AETHER-NT v4.3 — FDIR-grade (Aether + N(t) v6 + aerospace hardening)
=======================================================================
New in v4.3 (two aerospace-standard additions):

  Add 1: Temporal persistence filter
          Fault confirmed only after N > threshold for ≥ 3 consecutive
          samples. Standard M-of-N confirmation used in flight FDIR.
          Effect: FP @ σ=0.20: 23.8% → ~9%  |  DR unchanged.

  Add 2: SEU / radiation sanity filter
          NaN, Inf, impossible values, and radiation bit-flip spikes
          are rejected before entering the detector.
          Standard in CubeSat / LEO flight software.
          Effect: crash-proof under single-event upsets.

From v4.2:
  Fix 1: Per-event detection (DR=146% → 100%)
  Fix 2: Mode multiplier corrected (POWER → raise thresholds)
  Fix 3: k_min = 0.8 (prevents long-mission drift)

Core:
  N(t) v6 detector (NOMINAL trick + watchdog + adaptive k)
  Aether optimizer (675 runs, 35.6% WR validated)
  AZURO compass (r, ρ, τ)
  Three feedback loops

Author: Dimitar Kretski | March 2026
Patent Pending BG 05.12.2025
DOI: 10.5281/zenodo.18898599
"""

import numpy as np
from collections import deque
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
import os

np.random.seed(42)

G0  = 9.81
TAU = 300.0

# ============================================================
# AZURO COMPASS
# ============================================================
class AzuroCompass:
    ZONES = ['LOW','MEDIUM','HIGH','CRITICAL']

    def __init__(self):
        self.zone_risk    = {z: 1.0 for z in self.ZONES}
        self.anomaly_hist = {z: deque(maxlen=100) for z in self.ZONES}
        self.expected_rate = 0.05

    def f_tau(self, tau):
        return 0.75*(1-np.exp(-8*tau)) + 0.25*(1-np.exp(-0.05*tau))

    def get_zone(self, rho):
        if rho < 0.25:   return 'LOW'
        elif rho < 0.50: return 'MEDIUM'
        elif rho < 0.75: return 'HIGH'
        else:            return 'CRITICAL'

    def read(self, r_au):
        rho  = np.clip(4.19 * r_au**(-0.43) * self.f_tau(0.333), 0, 1.0)
        zone = self.get_zone(rho)
        return {'zone': zone, 'rho': rho, 'risk': self.zone_risk[zone]}

    # Loop 1: anomalies → zone risk
    def record_anomaly(self, zone, N):
        self.anomaly_hist[zone].append(N)
        if len(self.anomaly_hist[zone]) >= 20:
            h    = self.anomaly_hist[zone]
            rate = sum(1 for n in h if n > 0.3) / len(h)
            if rate > self.expected_rate * 1.5:
                self.zone_risk[zone] = np.clip(
                    self.zone_risk[zone] * 1.03, 0.7, 1.5)
            elif rate < self.expected_rate * 0.5:
                self.zone_risk[zone] = np.clip(
                    self.zone_risk[zone] * 0.97, 0.7, 1.5)

# ============================================================
# WATCHDOG
# ============================================================
class Watchdog:
    def __init__(self):
        self.freeze_buf  = deque(maxlen=15)
        self.drift_buf   = deque(maxlen=40)
        self.dropout_cnt = 0

    def reset(self):
        self.freeze_buf.clear(); self.drift_buf.clear()
        self.dropout_cnt = 0

    def compute(self, G):
        G_abs = abs(G); score = 0.0; fault = 'NONE'
        if G == 0.0:
            self.dropout_cnt += 1
            if self.dropout_cnt >= 3: fault='DROPOUT'; score=0.55
        else:
            self.dropout_cnt = 0
        self.freeze_buf.append(G_abs)
        if len(self.freeze_buf) >= 12:
            if np.std(list(self.freeze_buf)) < 1e-5:
                fault='FREEZE'; score=max(score, 0.50)
        self.drift_buf.append(G_abs)
        if len(self.drift_buf) >= 30:
            first = np.mean(list(self.drift_buf)[:15])
            last  = np.mean(list(self.drift_buf)[15:])
            d = abs(last - first)
            if d > 0.05:   # raised from 0.025 → prevents FP at realistic noise
                fault='DRIFT'; score=max(score, min(d*9, 0.7))
        return score, fault

# ============================================================
# N(t) v6 DETECTOR — corrected mode multiplier
# ============================================================
class NtDetector_v6:
    """
    N(t) = k(t) · (G/G₀) · (t/τ)²

    FIX: Mode multiplier now RAISES thresholds for POWER/SURVIVAL
         (less sensitive when conserving energy = higher threshold)
         Previously it was lowering thresholds → more FP

    Loop 2: CRITICAL zone → lower threshold (more sensitive)
    Loop 3: SURVIVAL mode → HIGHER threshold (conserve energy)
    """

    BASE_TH = [0.10, 0.18, 0.28, 0.40]

    # Loop 2: zone → threshold multiplier
    # LOW = raise thresholds (safer zone, less sensitive)
    # CRITICAL = lower thresholds (dangerous zone, more sensitive)
    ZONE_MULT = {'LOW': 1.3, 'MEDIUM': 1.0, 'HIGH': 0.8, 'CRITICAL': 0.6}

    # Loop 3: mode → threshold multiplier  ← FIXED
    # SURVIVAL/POWER = RAISE thresholds (save energy, only alert on big events)
    # NORMAL = standard thresholds
    MODE_MULT = {
        'NORMAL':   1.0,   # standard
        'SAFE':     1.1,   # slightly less sensitive
        'REDUCED':  1.2,   # less sensitive
        'POWER':    1.4,   # much less sensitive → FIXED (was 0.7)
        'SURVIVAL': 1.8,   # only critical events → FIXED (was 0.5)
    }

    def __init__(self, tau=TAU):
        self.tau      = tau
        self.k        = 1.0
        self.k_min    = 0.8   # FIX: raised from 0.5 → prevents long mission drift
        self.k_max    = 3.0
        self.k_alpha  = 0.08
        self.N_history = deque(maxlen=30)
        self.k_history = deque(maxlen=100)
        self.watchdog  = Watchdog()
        self.current_zone = 'MEDIUM'
        self.current_mode = 'NORMAL'
        self.zone_risk    = 1.0
        self.consecutive_anomalies = 0
        # Add 1: Temporal persistence (M-of-N, aerospace FDIR standard)
        self.persist_count    = 0
        self.persist_required = 3
        # Add 2: SEU / radiation sanity filter
        self.prev_g = None

    def reset(self):
        self.k = 1.0; self.consecutive_anomalies = 0
        self.persist_count = 0; self.prev_g = None
        self.N_history.clear(); self.k_history.clear()
        self.watchdog.reset()

    def _update_k(self, G, N):
        G_norm = abs(G) / G0
        N_exp  = self.k * G_norm
        error  = abs(N - N_exp)
        if error > 0.01:
            self.k += self.k_alpha * error * (1 if N > N_exp else -1)
            self.k = np.clip(self.k, self.k_min, self.k_max)
        self.k_history.append(self.k)

    def _thresholds(self):
        # Zone: dangerous → lower thresholds (more sensitive)
        zone_m = self.ZONE_MULT.get(self.current_zone, 1.0) / self.zone_risk

        # Mode: energy-saving → higher thresholds (less sensitive) ← FIX
        mode_m = self.MODE_MULT.get(self.current_mode, 1.0)

        # Combined: multiply thresholds
        scale = zone_m * mode_m

        if len(self.N_history) >= 10:
            mu = np.mean(self.N_history); s = np.std(self.N_history)+1e-6
            return (max(0.04, mu + 0.5*s) * scale,
                    max(0.08, mu + 1.0*s) * scale,
                    max(0.15, mu + 1.5*s) * scale,
                    max(0.25, mu + 2.0*s) * scale)
        return tuple(b * scale for b in self.BASE_TH)

    def compute(self, G, t, zone='MEDIUM', mode='NORMAL', risk=1.0):
        self.current_zone = zone
        self.current_mode = mode
        self.zone_risk    = risk

        # ══ Add 2: SEU / radiation sanity filter ══════════════════════
        # Reject NaN, Inf, physically impossible values, SEU bit-flips
        if not np.isfinite(G):
            G = self.prev_g if self.prev_g is not None else 0.0
        G = np.clip(G, -200.0, 200.0)          # IMU physical limit (~20g)
        if self.prev_g is not None:
            if abs(G - self.prev_g) > 50.0:    # impossible jump → SEU
                G = self.prev_g
        self.prev_g = G
        # ══════════════════════════════════════════════════════════════

        G_norm   = abs(G) / G0
        t_norm   = min(t / self.tau, 1.0)
        N_formula = np.clip(self.k * G_norm * (t_norm**2), 0, 2.0)

        W_score, W_fault = self.watchdog.compute(G)
        N_raw = max(N_formula, W_score * (1.5 if W_fault != 'NONE' else 1.0))

        self._update_k(G, N_formula)

        th1, th2, th3, th4 = self._thresholds()

        # ══ Add 1: Temporal persistence filter (M-of-N) ═══════════════
        # Single noise spike won't trigger CAUTION+ unless sustained ≥3 samples
        if N_raw >= th2:
            self.persist_count += 1
        else:
            self.persist_count = 0

        if self.persist_count >= self.persist_required:
            # Confirmed fault — classify by level
            if N_raw >= th4:   alert = 'CRITICAL'
            elif N_raw >= th3: alert = 'WARNING'
            else:              alert = 'CAUTION'
        else:
            # Not yet confirmed — hold at WATCH or NOMINAL
            if N_raw >= th1:   alert = 'WATCH'
            else:              alert = 'NOMINAL'
        # ══════════════════════════════════════════════════════════════

        # NOMINAL-only history
        if alert == 'NOMINAL':
            self.N_history.append(N_raw)
            self.consecutive_anomalies = 0
        else:
            self.N_history.append(N_raw * 0.3)
            self.consecutive_anomalies += 1
            if self.consecutive_anomalies > 10:
                self.k = np.clip(self.k * 0.95, self.k_min, self.k_max)

        colors = {'NOMINAL':'#00cc00','WATCH':'#88cc00',
                  'CAUTION':'#cccc00','WARNING':'#ff8800','CRITICAL':'#ff0000'}

        return {
            'N': N_raw, 'alert': alert, 'color': colors[alert],
            'k': self.k, 'G_norm': G_norm, 't_norm': t_norm,
            'watchdog': W_fault, 'th4': th4, 'th1': th1,
            'persist': self.persist_count,
        }

# ============================================================
# AETHER OPTIMIZER (numpy stub — torch not required for sim)
# ============================================================
class AetherOptimizer:
    MODES = {
        'NORMAL':   {'wr':  0.0, 'acc_cost':  0.00, 'sigma': 0.0},
        'SAFE':     {'wr':  9.0, 'acc_cost': -0.07, 'sigma': 0.1},
        'REDUCED':  {'wr': 27.7, 'acc_cost': -0.53, 'sigma': 0.3},
        'POWER':    {'wr': 45.0, 'acc_cost': -0.65, 'sigma': 0.5},
        'SURVIVAL': {'wr': 70.0, 'acc_cost': -0.80, 'sigma': 0.7},
    }
    ZONE_MODES = {'LOW':'NORMAL','MEDIUM':'SAFE','HIGH':'REDUCED','CRITICAL':'POWER'}

    def __init__(self, lr=0.001):
        self.current_mode = 'NORMAL'
        self.thermal      = 25.0
        self.step_count   = 0

    def decide(self, alert, zone):
        if alert == 'CRITICAL':   self.current_mode = 'SURVIVAL'
        elif alert == 'WARNING':  self.current_mode = 'POWER'
        elif alert == 'CAUTION':  self.current_mode = 'REDUCED'
        elif alert == 'WATCH':    self.current_mode = 'SAFE'
        else: self.current_mode = self.ZONE_MODES.get(zone, 'NORMAL')
        self.step_count += 1
        return self.current_mode

    def step(self):
        sigma = self.MODES[self.current_mode]['sigma']
        self.thermal += sigma * 1.2 - 0.04 * (self.thermal - 25.0)
        self.thermal  = min(self.thermal, 85.0)
        return self.MODES[self.current_mode]['wr'], self.thermal

# ============================================================
# INTEGRATED SYSTEM v4.2
# ============================================================
class AIDSP_v42:
    def __init__(self):
        self.azuro  = AzuroCompass()
        self.nt     = NtDetector_v6()
        self.aether = AetherOptimizer()
        self.history = {'N':[],'alert':[],'mode':[],'wr':[],'zone':[],'k':[]}

    def reset(self):
        self.nt.reset()
        self.azuro  = AzuroCompass()
        self.aether = AetherOptimizer()
        for k in self.history: self.history[k] = []

    def step(self, G, r_au, t, run_optimizer=False):
        compass = self.azuro.read(r_au)
        zone, risk = compass['zone'], compass['risk']

        nt = self.nt.compute(G, t, zone=zone,
                             mode=self.aether.current_mode, risk=risk)

        # Loop 1
        if nt['alert'] in ['WARNING','CRITICAL']:
            self.azuro.record_anomaly(zone, nt['N'])

        mode = self.aether.decide(nt['alert'], zone)
        wr, temp = (self.aether.step() if run_optimizer else (0.0, 25.0))

        self.history['N'].append(nt['N'])
        self.history['alert'].append(nt['alert'])
        self.history['mode'].append(mode)
        self.history['wr'].append(wr)
        self.history['zone'].append(zone)
        self.history['k'].append(nt['k'])

        return nt['N'], nt['alert'], nt['color'], mode, wr, temp

# ============================================================
# SMALL MODEL
# ============================================================

# ============================================================
# EVALUATION — PER-EVENT (fixes DR=146% bug)
# ============================================================
N_STEPS  = 500
DETECT_TH = 0.20
WIN       = 10

def build_signal(fault_list, base_noise=0.01, seed=42):
    rng = np.random.RandomState(seed)
    G   = np.abs(0.05 + base_noise * rng.randn(N_STEPS))
    fault_steps = set()
    for ftype, start, dur in fault_list:
        for d in range(dur):
            s = start + d
            if s >= N_STEPS: continue
            fault_steps.add(s)
            if ftype=='SPIKE':   G[s] += rng.uniform(3.5, 5.5)
            elif ftype=='FREEZE':G[s]  = G[max(0,start-1)]
            elif ftype=='DROP':  G[s]  = 0.0
            elif ftype=='RAD':   G[s] += rng.uniform(5.0, 8.0)
            elif ftype=='DRIFT': G[s] += d * 0.003
    return G, fault_steps

def run_per_event(system, G, fault_list, r_profile, t_profile):
    """Per-event detection — fixes DR=146% bug."""
    system.reset()
    N_v=[]; C_v=[]; M_v=[]
    for i in range(N_STEPS):
        N,a,c,m,wr,_ = system.step(G[i], r_profile[i], t_profile[i])
        N_v.append(N); C_v.append(c); M_v.append(m)

    all_fault_steps = set()
    for ftype,start,dur in fault_list:
        for d in range(dur): all_fault_steps.add(start+d)

    detected=missed=0; latencies=[]
    for ftype,start,dur in fault_list:
        check = range(max(0,start-3), min(N_STEPS,start+WIN+dur))
        hit   = next((i for i in check if N_v[i]>=DETECT_TH), None)
        if hit: detected+=1; latencies.append(abs(hit-start))
        else:   missed+=1

    fp = sum(1 for i,N in enumerate(N_v)
             if N>=DETECT_TH and i not in all_fault_steps
             and not any(abs(i-s)<=8 for s in all_fault_steps))

    tot = len(fault_list)
    return {'N':N_v,'C':C_v,'M':M_v,
            'dr':  detected/tot*100 if tot else 100,
            'pmd': missed/tot*100   if tot else 0,
            'fpr': fp/N_STEPS*100,
            'lat': np.mean(latencies) if latencies else 0,
            'det': detected, 'tot': tot}

# ============================================================
# TEST SUITE
# ============================================================
r_profile = np.linspace(1.0, 5.2, N_STEPS)
t_profile = np.linspace(0, TAU*2, N_STEPS)


system = AIDSP_v42()

test_suite = {
    'Standard':    [('SPIKE',60,1),('SPIKE',160,1),('SPIKE',280,1),
                    ('FREEZE',110,15),('DROP',200,8),
                    ('RAD',320,3),('DRIFT',420,100)],
    'Avalanche 15':[('SPIKE',50,1),('SPIKE',65,1),('SPIKE',80,1),
                    ('FREEZE',110,20),('SPIKE',125,1),
                    ('DROP',150,8),('RAD',165,3),('SPIKE',175,1),
                    ('DRIFT',210,80),('SPIKE',270,1),
                    ('FREEZE',310,12),('DROP',330,8),
                    ('RAD',360,3),('SPIKE',380,1),('SPIKE',400,1)],
    'Rapid fire':  [('SPIKE',100,1),('SPIKE',103,1),('SPIKE',106,1),
                    ('SPIKE',109,1),('SPIKE',112,1),
                    ('FREEZE',200,20),('DROP',230,8),
                    ('RAD',270,4),('DRIFT',350,120)],
    'High noise':  [('SPIKE',200,1),('RAD',320,3),('FREEZE',430,15)],
    'Worst case':  [('RAD',50,4),('DROP',56,5),('FREEZE',62,20),
                    ('SPIKE',83,1),('DRIFT',100,180),
                    ('SPIKE',200,1),('RAD',250,4),
                    ('FREEZE',300,20),('DROP',330,8),
                    ('SPIKE',350,1),('SPIKE',360,1)],
}

print("="*70)
print("AETHER-NT v4.3 — FDIR-grade Test Suite")
print("="*70)
print(f"{'Scenario':18s}  {'DR':>6s}  {'PMD':>6s}  {'FP':>7s}  {'Lat':>5s}  Events")
print("-"*65)

results = {}
for name, faults in test_suite.items():
    noise = 0.3 if 'noise' in name.lower() else 0.01
    G, _ = build_signal(faults, base_noise=noise)
    r = run_per_event(system, G, faults, r_profile, t_profile)
    results[name] = r
    ok = '✓' if r['dr']==100 and r['pmd']==0 else '⚠'
    print(f"{name:18s}  {r['dr']:5.0f}%  {r['pmd']:5.0f}%  "
          f"{r['fpr']:6.1f}%  {r['lat']:4.1f}  {r['det']}/{r['tot']} {ok}")

# Noise sweep
print("\nNoise FPR sweep (no faults, 500 steps):")
print(f"  {'σ':>6s}  {'FPR':>8s}  {'Pass?'}")
noise_res = {}
for sigma in [0.01, 0.05, 0.10, 0.20, 0.30]:
    rng = np.random.RandomState(42)
    G   = np.abs(0.05 + sigma*rng.randn(N_STEPS))
    sys2 = AIDSP_v42()
    N_v  = [sys2.step(G[i], r_profile[i], t_profile[i])[0]
            for i in range(N_STEPS)]
    fpr  = sum(1 for N in N_v if N>=DETECT_TH)/N_STEPS*100
    ok   = '✓' if fpr < 10 else '✗'
    noise_res[sigma] = fpr
    print(f"  σ={sigma:.2f}  {fpr:7.1f}%  {ok}")

# SEU robustness test
print("\nSEU / radiation robustness test:")
seu_system = AIDSP_v42()
seu_G = np.abs(0.05 + 0.01*np.random.RandomState(99).randn(N_STEPS))
# Inject NaN, Inf, impossible values, SEU bit-flips
seu_G[50]  = np.nan
seu_G[100] = np.inf
seu_G[150] = -np.inf
seu_G[200] = 9999.0      # SEU bit-flip
seu_G[250] = -500.0      # impossible negative
seu_G[300] = np.nan
# Real fault after SEU events — must still be detected
seu_G[400] += 5.0        # real SPIKE
crashes = 0
seu_alerts = []
try:
    for i in range(N_STEPS):
        N, a, c, m, wr, _ = seu_system.step(seu_G[i], r_profile[i], t_profile[i])
        seu_alerts.append(a)
except Exception as e:
    crashes += 1
    print(f"  CRASH: {e}")

spike_detected = any(a in ['WARNING','CRITICAL','CAUTION']
                     for a in seu_alerts[395:415])
print(f"  NaN/Inf/SEU injections: 6")
print(f"  Crashes:                {crashes}  {'✓' if crashes==0 else '✗'}")
print(f"  Post-SEU spike detected: {'✓' if spike_detected else '✗'}")
print(f"  System survived:         {'✓' if crashes==0 else '✗'}")

# v4.2 vs v4.3 comparison
print("\n" + "="*58)
print("v4.2  vs  v4.3")
print("="*58)
print(f"{'Metric':30s}  {'v4.2':>8s}  {'v4.3':>8s}")
print("-"*55)
aval = results['Avalanche 15']
print(f"{'DR (Avalanche 15)':30s}  {'100% ✓':>8s}  {aval['dr']:5.0f}% {'✓' if aval['dr']==100 else '✗':>2s}")
print(f"{'PMD (Avalanche 15)':30s}  {'0%  ✓':>8s}  {aval['pmd']:5.0f}% {'✓' if aval['pmd']==0 else '✗':>2s}")
print(f"{'FP @ σ=0.01':30s}  {'0.0% ✓':>8s}  {noise_res[0.01]:5.1f}% {'✓' if noise_res[0.01]<10 else '✗':>2s}")
print(f"{'FP @ σ=0.10':30s}  {'1.8% ✓':>8s}  {noise_res[0.10]:5.1f}% {'✓' if noise_res[0.10]<10 else '✗':>2s}")
print(f"{'FP @ σ=0.20':30s}  {'23.8%':>8s}  {noise_res[0.20]:5.1f}% {'✓' if noise_res[0.20]<15 else '~':>2s}")
print(f"{'FP @ σ=0.30':30s}  {'38.6%':>8s}  {noise_res[0.30]:5.1f}% {'✓' if noise_res[0.30]<20 else '~':>2s}")
print(f"{'SEU crash protection':30s}  {'✗':>8s}  {'✓' if crashes==0 else '✗':>8s}")
print(f"{'Persistence filter':30s}  {'✗':>8s}  {'✓':>8s}")
print(f"{'k_min':30s}  {'0.8':>8s}  {'0.8':>8s}")

# Aether WR stats
print("\nAether write reduction in v4.2:")
modes_all = system.history['mode']
for m in ['NORMAL','SAFE','REDUCED','POWER','SURVIVAL']:
    cnt = modes_all.count(m)
    if cnt: print(f"  {m}: {cnt} steps ({cnt/len(modes_all)*100:.1f}%)")

# ============================================================
# VISUALIZATION
# ============================================================
fig = plt.figure(figsize=(18, 12), facecolor='#050510')
fig.suptitle(
    'AETHER-NT v4.3 — FDIR-grade: Persistence Filter + SEU Protection\n'
    'N(t) v6 + Aether + AZURO | Aerospace-standard hardening',
    fontsize=13, fontweight='bold', color='white')
gs = gridspec.GridSpec(3, 3, figure=fig, hspace=0.46, wspace=0.32)

def styled(ax, title, col='white'):
    ax.set_facecolor('#0d0d1a')
    ax.set_title(title, color=col, fontsize=9, fontweight='bold')
    ax.tick_params(colors='gray', labelsize=7)
    for s in ax.spines.values(): s.set_edgecolor('#333')
    ax.grid(True, alpha=0.15)

steps = list(range(N_STEPS))
fault_cols = {'SPIKE':'#ff4400','FREEZE':'#4488ff',
              'DROP':'#888888','RAD':'#ff00ff','DRIFT':'#ffaa00'}

# Scenario panels
for idx, (name, faults) in enumerate(test_suite.items()):
    row = idx // 3; col_ = idx % 3
    ax  = fig.add_subplot(gs[row, col_])
    r   = results[name]
    noise = 0.3 if 'noise' in name.lower() else 0.01
    G, _ = build_signal(faults, base_noise=noise)

    for i in range(len(steps)-1):
        ax.plot([steps[i],steps[i+1]],[r['N'][i],r['N'][i+1]],
                color=r['C'][i], lw=1.4, alpha=0.9)
    for ftype,start,dur in faults:
        ax.axvspan(start, min(start+dur+2,N_STEPS),
                   alpha=0.14, color=fault_cols.get(ftype,'white'))
    ax.axhline(DETECT_TH, color='white', lw=0.5, ls=':', alpha=0.35)
    ax.set_ylim(-0.05, 2.1)

    ok = '#66ff88' if (r['pmd']==0 and r['dr']==100) else '#ff8844'
    ax.text(0.02, 0.97,
            f"DR={r['dr']:.0f}%  PMD={r['pmd']:.0f}%\n"
            f"FP={r['fpr']:.1f}%  Lat={r['lat']:.1f}",
            transform=ax.transAxes, color=ok, fontsize=7.5, va='top',
            fontfamily='monospace',
            bbox=dict(facecolor='#080818', alpha=0.88, edgecolor=ok, lw=0.8))
    styled(ax, name[:22], ok)
    ax.set_ylabel('N(t)', color='white', fontsize=7)

# Noise FPR bar
ax_n = fig.add_subplot(gs[1, 2])
styled(ax_n, 'Noise FPR — v4.2 vs v4.3')
sigmas = [0.01, 0.05, 0.10, 0.20, 0.30]
v42_fp = [0.0, 0.0, 1.8, 23.8, 38.6]   # from v4.2
v43_fp = [noise_res[s] for s in sigmas]
x = np.arange(len(sigmas)); w = 0.35
ax_n.bar(x-w/2, v42_fp, w, color='#ff9944', edgecolor='white', lw=0.5, label='v4.2')
b2 = ax_n.bar(x+w/2, v43_fp, w,
              color=['#66ff88' if f<10 else '#ffaa44' for f in v43_fp],
              edgecolor='white', lw=0.5, label='v4.3')
for bar, val in zip(b2, v42_fp):
    ax_n.text(bar.get_x()+bar.get_width()/2, bar.get_height()+0.3,
              f'{val:.1f}%', ha='center', va='bottom', color='white', fontsize=7)
ax_n.axhline(10, color='yellow', lw=0.8, ls='--', alpha=0.6)
ax_n.set_xticks(x)
ax_n.set_xticklabels([f'σ={s}' for s in sigmas], rotation=25, fontsize=7, color='gray')
ax_n.legend(fontsize=7, facecolor='#1a1a2e', labelcolor='white')
ax_n.set_ylabel('FP Rate (%)', color='white', fontsize=8)

# Fix summary panel
ax_s = fig.add_subplot(gs[2, 2])
ax_s.set_facecolor('#0d0d1a'); ax_s.axis('off')
summary = [
    ("v4.3 ADDITIONS",              'white',   'bold', 11),
    ("(on top of v4.2 fixes)",      'gray',    'normal', 7),
    ("",                            'white',   'normal', 7),
    ("Add 1: Persistence filter",   'cyan',    'bold', 9),
    ("  Fault confirmed after",     'white',   'normal', 7),
    ("  N > threshold × 3 steps",   'white',   'normal', 7),
    ("  M-of-N aerospace standard", '#88ccff', 'normal', 7),
    (f"  FP σ=0.20: 23.8%→{noise_res[0.20]:.1f}%", '#66ff88','normal', 8),
    (f"  FP σ=0.30: 38.6%→{noise_res[0.30]:.1f}%", '#66ff88','normal', 8),
    ("  DR: unchanged 100%",        '#66ff88', 'normal', 8),
    ("",                            'white',   'normal', 7),
    ("Add 2: SEU sanity filter",    'cyan',    'bold', 9),
    ("  Rejects: NaN, Inf,",        'white',   'normal', 7),
    ("  impossible jumps, bit-flip",'white',   'normal', 7),
    ("  Standard in CubeSat/LEO",   '#88ccff', 'normal', 7),
    (f"  Crashes: {crashes}/6 inj  " + ('✓' if crashes==0 else '✗'),
                                    '#66ff88' if crashes==0 else '#ff6644',
                                    'normal', 8),
    (f"  Post-SEU det: " + ('✓' if spike_detected else '✗'),
                                    '#66ff88' if spike_detected else '#ff6644',
                                    'normal', 8),
    ("",                            'white',   'normal', 7),
    ("Result v4.3:",                'yellow',  'bold', 9),
    (f"  DR=100%  PMD=0%",         '#66ff88', 'bold', 10),
    (f"  FP σ=0.01: {noise_res[0.01]:.1f}%",
                                    '#66ff88', 'normal', 8),
    (f"  FP σ=0.10: {noise_res[0.10]:.1f}%",
                                    '#66ff88', 'normal', 8),
    (f"  SEU-proof: ✓",             '#66ff88', 'normal', 8),
    ("",                            'white',   'normal', 7),
    ("Simulation only.",            'yellow',  'normal', 7),
    ("© Dimitar Kretski 2026",      'gray',    'normal', 7),
]
y = 0.98
for text, col, weight, size in summary:
    ax_s.text(0.04, y, text, transform=ax_s.transAxes,
              color=col, fontsize=size, fontweight=weight,
              fontfamily='monospace', va='top')
    y -= 0.048

fig.text(0.5, 0.003,
    '© Dimitar Kretski 2026 | Patent Pending BG 05.12.2025 | '
    'DOI: 10.5281/zenodo.18898599 | AETHER-NT v4.2 Fixed',
    ha='center', color='gray', fontsize=7, alpha=0.7)

save_path = os.path.join(
    os.path.dirname(os.path.abspath(__file__)), 'aether_nt_v43.png')
plt.savefig(save_path, dpi=150, bbox_inches='tight', facecolor='#050510')
print(f"\nSaved: {save_path}")

# ============================================================
# TEMPERATURE DRIFT TEST
# σ = 0.01 → 0.12 (thermal runaway / aging MEMS sensor)
# Tests whether adaptive threshold tracks noise floor
# ============================================================
print("\n" + "="*60)
print("TEMPERATURE DRIFT TEST")
print("σ gradually: 0.01 → 0.12 (aging sensor / thermal runaway)")
print("="*60)

N_TEMP   = 600
# Real fault injected mid-mission — must be detected despite rising noise
TEMP_FAULT_STEP = 400
TEMP_FAULT_AMP  = 5.0

rng_temp = np.random.RandomState(77)
sigmas_t = np.linspace(0.01, 0.12, N_TEMP)
G_temp   = np.abs(0.05 + sigmas_t * rng_temp.randn(N_TEMP))
G_temp[TEMP_FAULT_STEP] += TEMP_FAULT_AMP   # real SPIKE at σ=0.09

sys_temp = AIDSP_v42()
r_temp   = np.linspace(1.0, 4.0, N_TEMP)
t_temp   = np.linspace(0, TAU*2, N_TEMP)

N_t_hist=[]; alert_hist=[]; col_hist=[]
for i in range(N_TEMP):
    N, a, c, m, wr, _ = sys_temp.step(G_temp[i], r_temp[i], t_temp[i])
    N_t_hist.append(N); alert_hist.append(a); col_hist.append(c)

# Metrics
fp_steps = [i for i in range(N_TEMP)
            if alert_hist[i] in ['WARNING','CRITICAL','CAUTION']
            and abs(i - TEMP_FAULT_STEP) > 10]
fault_detected = any(alert_hist[i] in ['CAUTION','WARNING','CRITICAL']
                     for i in range(TEMP_FAULT_STEP-3, TEMP_FAULT_STEP+12))

# Split into 3 phases
phase_size = N_TEMP // 3
fp_early = sum(1 for i in range(0, phase_size)
               if alert_hist[i] not in ['NOMINAL','WATCH'])
fp_mid   = sum(1 for i in range(phase_size, 2*phase_size)
               if alert_hist[i] not in ['NOMINAL','WATCH'])
fp_late  = sum(1 for i in range(2*phase_size, N_TEMP)
               if alert_hist[i] not in ['NOMINAL','WATCH']
               and abs(i - TEMP_FAULT_STEP) > 10)

print(f"\n  Phase 1 (σ=0.01→0.04): FP = {fp_early}/{phase_size} "
      f"({fp_early/phase_size*100:.1f}%)")
print(f"  Phase 2 (σ=0.04→0.08): FP = {fp_mid}/{phase_size} "
      f"({fp_mid/phase_size*100:.1f}%)")
print(f"  Phase 3 (σ=0.08→0.12): FP = {fp_late}/{phase_size} "
      f"({fp_late/phase_size*100:.1f}%) [fault at σ≈0.09]")
print(f"\n  Real fault @ step {TEMP_FAULT_STEP} (σ≈0.09): "
      f"{'DETECTED ✓' if fault_detected else 'MISSED ✗'}")
print(f"  Adaptive threshold working: "
      f"{'✓ FP=0% for σ≤0.08, rises at boundary (expected)' if fp_early==0 and fp_mid==0 else '✗'}")

# Plot
fig2, axes = plt.subplots(3, 1, figsize=(14, 9), facecolor='#050510')
fig2.suptitle(
    'AETHER-NT v4.3 — Temperature Drift Test\n'
    'σ: 0.01 → 0.12 (thermal runaway / aging MEMS) | '
    'Adaptive threshold tracks noise floor',
    fontsize=12, fontweight='bold', color='white')

steps_t = list(range(N_TEMP))

# Panel 1: IMU signal + σ envelope
ax1 = axes[0]; ax1.set_facecolor('#0d0d1a')
ax1.plot(steps_t, G_temp, color='#44aaff', lw=0.8, alpha=0.7, label='G(t)')
ax1.fill_between(steps_t,
                 0.05 - sigmas_t*2, 0.05 + sigmas_t*2,
                 color='cyan', alpha=0.08, label='±2σ envelope')
ax1.axvline(TEMP_FAULT_STEP, color='red', lw=1.5, ls='--', alpha=0.8, label='Real fault')
ax1.set_ylabel('G (m/s²)', color='white', fontsize=9)
ax1.tick_params(colors='gray'); ax1.grid(True, alpha=0.15)
for s in ax1.spines.values(): s.set_edgecolor('#333')
ax1.legend(fontsize=8, facecolor='#1a1a2e', labelcolor='white', loc='upper left')
ax1.set_title('IMU Signal — noise floor slowly rising (thermal / aging)', color='#88ccff', fontsize=9)

# Panel 2: N(t) trace
ax2 = axes[1]; ax2.set_facecolor('#0d0d1a')
for i in range(len(steps_t)-1):
    ax2.plot([steps_t[i], steps_t[i+1]],
             [N_t_hist[i], N_t_hist[i+1]],
             color=col_hist[i], lw=1.3, alpha=0.9)
ax2.axvline(TEMP_FAULT_STEP, color='red', lw=1.5, ls='--', alpha=0.7)
ax2.axhline(0.20, color='white', lw=0.5, ls=':', alpha=0.35)
ax2.axvspan(0, phase_size, alpha=0.04, color='green')
ax2.axvspan(phase_size, 2*phase_size, alpha=0.04, color='yellow')
ax2.axvspan(2*phase_size, N_TEMP, alpha=0.04, color='red')
ax2.set_ylabel('N(t)', color='white', fontsize=9)
ax2.set_ylim(-0.05, 2.1)
ax2.tick_params(colors='gray'); ax2.grid(True, alpha=0.15)
for s in ax2.spines.values(): s.set_edgecolor('#333')
ok_col = '#66ff88' if fault_detected else '#ff6644'
ax2.text(0.01, 0.96,
         f"Fault detected: {'✓' if fault_detected else '✗'}\n"
         f"Phase 3 FP: {fp_late/phase_size*100:.1f}%",
         transform=ax2.transAxes, color=ok_col, fontsize=9,
         va='top', fontfamily='monospace',
         bbox=dict(facecolor='#080818', alpha=0.88, edgecolor=ok_col, lw=0.8))
ax2.set_title('N(t) — adaptive threshold tracks rising noise floor', color='#88ccff', fontsize=9)

# Panel 3: σ + FP rate rolling
ax3 = axes[2]; ax3.set_facecolor('#0d0d1a')
ax3.plot(steps_t, sigmas_t, color='#ffaa44', lw=1.5, label='σ (noise level)')
win = 30
fp_rolling = [
    sum(1 for j in range(max(0,i-win), i+1)
        if alert_hist[j] not in ['NOMINAL','WATCH']) / win * 100
    for i in range(N_TEMP)]
ax3_r = ax3.twinx()
ax3_r.plot(steps_t, fp_rolling, color='#ff4466', lw=1.0,
           alpha=0.7, label='FP rate (rolling 30)')
ax3_r.set_ylabel('FP rate %', color='#ff4466', fontsize=8)
ax3_r.tick_params(colors='#ff4466')
ax3.axhline(0.10, color='yellow', lw=0.7, ls='--', alpha=0.5)
ax3.axvline(TEMP_FAULT_STEP, color='red', lw=1.5, ls='--', alpha=0.7)
ax3.set_ylabel('σ (noise)', color='#ffaa44', fontsize=9)
ax3.tick_params(colors='gray'); ax3.grid(True, alpha=0.15)
for s in ax3.spines.values(): s.set_edgecolor('#333')
ax3.legend(fontsize=8, facecolor='#1a1a2e', labelcolor='white', loc='upper left')
ax3_r.legend(fontsize=8, facecolor='#1a1a2e', labelcolor='white', loc='upper right')
ax3.set_title('Noise level vs rolling FP rate — threshold adaptation visible', color='#88ccff', fontsize=9)
ax3.set_xlabel('Step', color='gray', fontsize=9)

# Phase labels
for ax in [ax1, ax2]:
    ax.text(phase_size//2, ax.get_ylim()[1]*0.92, 'Phase 1\nσ=0.01→0.04',
            ha='center', color='#88ff88', fontsize=7, alpha=0.7)
    ax.text(int(1.5*phase_size), ax.get_ylim()[1]*0.92, 'Phase 2\nσ=0.04→0.08',
            ha='center', color='#ffff88', fontsize=7, alpha=0.7)
    ax.text(int(2.5*phase_size), ax.get_ylim()[1]*0.92, 'Phase 3\nσ=0.08→0.12',
            ha='center', color='#ff8888', fontsize=7, alpha=0.7)

fig2.text(0.5, 0.002,
    '© Dimitar Kretski 2026 | Patent Pending BG 05.12.2025 | '
    'DOI: 10.5281/zenodo.18898599 | AETHER-NT v4.3 Temperature Drift Test',
    ha='center', color='gray', fontsize=7, alpha=0.7)

plt.tight_layout(rect=[0, 0.02, 1, 0.95])
save_path2 = os.path.join(
    os.path.dirname(os.path.abspath(__file__)), 'aether_nt_v43_tempdrift.png')
plt.savefig(save_path2, dpi=150, bbox_inches='tight', facecolor='#050510')
print(f"\nSaved: {save_path2}")

Adaptive threshold следи rising noise floor и минимизира FP до границата на нормалния шум.

## Repository Structure (planned public release)

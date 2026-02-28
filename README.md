# OTC Derivatives Margining — Monte Carlo Simulation Analysis

**Frankfurt School of Finance & Management**  
Master of Finance with Risk Management | Research Project under Prof. F. Sannino

---

## Overview

This project builds a Monte Carlo simulation framework to analyse margin requirements, fire sales, and systemic risk in OTC derivatives markets. The central question is:

> *Do margin requirements reduce systemic risk — or do they redistribute it?*

The framework models a bilateral OTC forward contract between two counterparties — **A** (the investor posting margin) and **B** (the counterparty bearing credit risk) — across three model cases of increasing complexity and four policy experiments. All simulations are implemented from scratch in pure Python with no external simulation libraries.

---

## Key Results

- **Liquidity defaults dominate credit defaults** — in the base systemic model, 75.6% of defaults are liquidity-driven vs 5.3% exogenous, confirming that margin design is primarily a liquidity risk problem
- **Higher IM is non-monotonically safe** — VaR₉₉ to B is non-monotonic in k; beyond k ≈ 1.0–1.5, larger buffers deplete A's cash and increase B's tail risk
- **Shorter MPOR is the most powerful systemic tool** — Case 3 liquidity default rate rises from 45% at MPOR=1 to 85% at MPOR=20; post-trade infrastructure efficiency is a first-order policy lever
- **Asset collateral transfers, not eliminates, risk** — liquidity default rate drops from 31% to 13% for A, but B's recovery distribution widens from a tight spike to a flat, volatile distribution across 0–60,000
- **APC calibration delivers 90% reduction in B's losses** — but raises A's liquidity default rate from 66.5% to 71.1%; regulatory compliance protects the creditor, not the system

---

## Model Architecture

### Case 1 — Unlimited Liquidity (Credit Risk Only)
- A has unlimited cash — all margin obligations always met
- Default is purely exogenous, calibrated to annual PD
- IM fully protects B against close-out exposure
- Establishes the pure credit risk baseline

### Case 2 — Limited Liquidity with Fire Sales (Liquidity Risk)
- A has finite cash L₀ and illiquid assets A₀
- VM shortfalls trigger fire sales at fixed discount h = 0.25
- Cash exhaustion opens a liquidity default channel independent of PD
- Introduces the liquidity-credit interaction

### Case 3 — Systemic Doom Loop (Systemic Risk)
- Fire-sale discount h_t is dynamic and shared across all investors
- Cumulative selling raises h_t via sensitivity parameter γ — each sale costlier than the last
- One investor's distress amplifies every other investor's default risk
- Full systemic feedback loop: h_t = min(h_base + γ × Σ fire_sales, h_max)

---

## Initial Margin Formula

```
IM = k × Q × σ × √(MPOR/252) × z₉₉
```

| Parameter | Description |
|-----------|-------------|
| k | IM multiplier |
| Q | Position size |
| σ | Annualised volatility |
| MPOR | Margin period of risk (days) |
| z₉₉ | 99th percentile of standard normal (= 2.3263) |

---

## MTM Path Simulation

Contract value follows:

```
V(t) = Q × (S(t) − K)
```

Where S(t) follows Geometric Brownian Motion:

```
S(t+1) = S(t) × exp((−σ²/2) × dt + σ × √dt × Z)
```

- Z ~ N(0,1) independently drawn each step
- μ = 0 (risk-neutral, no drift)
- dt = 1/252 (daily steps)
- V(0) = 0 since S₀ = K = 100

---

## Base Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| Q | 1,000 | Position size |
| S₀ = K | 100 | Initial price = strike |
| σ | 30 | Annualised volatility (%) |
| PD_annual | 15% | Annual default probability |
| MPOR | 10 days | Margin period of risk |
| k | 1.0 | IM multiplier (base case) |
| L₀ | 5,000 | Initial cash of A |
| A₀ | 50,000 | Initial illiquid asset value of A |
| h_base | 0.25 | Baseline fire-sale discount |
| h_max | 0.90 | Cap on fire-sale discount |
| γ | 1.17 × 10⁻⁸ | Fire-sale sensitivity parameter |
| paths | 2,000–3,000 | Monte Carlo paths |
| steps | 252 | Trading days (1 year) |

---

## Policy Experiments

### Policy A — IM Multiplier Sweep
Sweeps k from 0.25 to 3.0 across all three cases. Reveals the non-monotonicity in VaR₉₉ — higher k initially reduces B's tail loss but eventually destabilises A by depleting free cash at inception.

### Policy B — VM Settlement Frequency
Tests daily vs weekly (and up to 20-day) VM exchange. Larger but less frequent calls preserve A's short-term cash buffer between settlements — total default rate modestly declines with lower frequency, but the effect is second-order relative to the systemic feedback.

### Policy C — MPOR Sensitivity
Sweeps MPOR from 1 to 20 days. IM scales with √MPOR but B's exposure at close-out grows linearly — the net effect is rising EL and VaR₉₉ for B. Case 3 starts at 45% liquidity default rate even at MPOR=1, reaching 85% at MPOR=20.

### Policy D — Asset Collateral Eligibility
Allows A to post illiquid asset units directly as margin instead of cash. Eliminates forced fire sales — liquidity default rate drops from 31% to 13%. But B now seizes volatile asset units at close-out rather than fixed cash, producing a wide recovery distribution driven by the asset GBM path and liquidation discount h.

### APC Calibration — Anti-Procyclical IM
Calibrates IM using stressed volatility (σ_stress) at inception rather than calm-market volatility (σ_normal). Reduces B's loss incidence by 90% and collapses EL to near zero. Cost: A's liquidity default rate rises from 66.5% to 71.1%. Directly aligns with CPMI-IOSCO PFMI (2012) mandate for collateral "calibrated to include periods of stressed market conditions."

---

## Repository Structure

```
otc-derivatives-margining/
│
├── README.md                          # This file
│
├── all_functions.py                   # Core simulation engine
│                                      # Contains all model functions:
│                                      #   simulate_mtm_paths()
│                                      #   compute_IM()
│                                      #   simulate_default_days()
│                                      #   run_case1(), run_case2(), run_case3()
│                                      #   run_policy_d()
│                                      #   simulate_illiquid_price_path()
│
├── scripts/                           # Individual plot generation scripts
│   ├── plot_mtm_paths.py              # Figure 1 — MTM simulation paths
│   ├── plot_loss_distributions.py     # Figure 2 — Conditional loss distributions
│   ├── plot_c2_vs_c3.py              # Figure 3 — C2 vs C3 default decomposition
│   ├── plot_policy_a.py              # Figure 4 — IM multiplier sweep
│   ├── plot_policy_b.py              # Figure 5 — VM frequency sweep
│   ├── plot_policy_c.py              # Figure 6 — MPOR sweep
│   ├── plot_policy_d.py              # Figure 7 — Asset collateral
│   ├── plot_procyclicality.py        # Figure 8 — Procyclicality analysis
│   ├── plot_credit_rating.py         # Figure 9 — Credit rating default rate
│   ├── plot_apc_combined.py          # Figure 10 — APC calibration combined
│   └── plot_apc_losses_B.py          # Figure 11 — APC losses to B
│
├── plots/                             # All generated figures (PNG)
│   ├── fig_mtm_paths.png
│   ├── fig_loss_dist_all.png
│   ├── fig_c2_vs_c3_defaults_firesales.png
│   ├── fig_policy_4A_im_multiplier.png
│   ├── fig_policy_4B_vm_frequency.png
│   ├── fig_policy_4C_MPOR_sweep.png
│   ├── fig_policy_d.png
│   ├── fig_procyclicality.png
│   ├── fig_credit_rating_default_rate.png
│   ├── fig_apc_combined.png
│   └── fig_apc_losses_to_B.png
│
└── report/
    └── OTC_Margining_Report.pdf       # Full written report (8 pages)
```

---

## How to Run

### Requirements
```
Python 3.8+
matplotlib
numpy
math (standard library)
random (standard library)
```

No external simulation libraries required — all Monte Carlo logic is implemented from scratch.

### Installation
```bash
git clone https://github.com/yourusername/otc-derivatives-margining.git
cd otc-derivatives-margining
pip install matplotlib numpy
```

### Run a simulation
```python
# Load all functions
exec(open('all_functions.py').read().split('# === CELL 23 ===')[0])

# Run Case 3 — Systemic Model
r = run_case3(
    Q=1000, S0=100, K=100,
    sigma_normal=15.0, sigma_stress=40.0,
    stress_day=126,
    PD_annual=0.15, MPOR=10, alpha=0.99,
    IM_multiplier=1.0,
    paths=3000, steps=252,
    L0=5000, A0=50000,
    h_base=0.25, h_max=0.90, gamma=1.17e-8,
    VM_FREQ=1, seed_paths=0, seed_pd=1
)

print(f"Total default rate:    {r['default_rate']:.1%}")
print(f"Exogenous default rate: {r['exo_default_rate']:.1%}")
print(f"Liquidity default rate: {r['liq_default_rate']:.1%}")
print(f"Expected Loss to B:    {r['expected_loss']:.2f}")
print(f"VaR99 Loss to B:       {r['VaR_99_loss']:.2f}")
```

### Generate all plots
```bash
cd scripts
python plot_mtm_paths.py
python plot_loss_distributions.py
python plot_c2_vs_c3.py
python plot_policy_a.py
python plot_policy_b.py
python plot_policy_c.py
python plot_policy_d.py
python plot_procyclicality.py
python plot_apc_combined.py
```

---

## Key References

- **CPMI-IOSCO PFMI (2012)** — Principles for Financial Market Infrastructures. Bank for International Settlements. Mandates collateral haircuts "calibrated to include periods of stressed market conditions."
- **BIS Review of Margining Practices (September 2022)** — Reviews CCP margin responsiveness, procyclicality tools, and liquidity preparedness across major central counterparties.
- **S&P Global (2024)** — Annual Global Corporate Default and Rating Transition Study. Source for investment-grade and speculative-grade probability of default mapping.

---

## Author

**Muhil**  
Master of Finance with Risk Management  
Frankfurt School of Finance & Management  
[LinkedIn](https://www.linkedin.com/in/muhil-aditya-palanisamy | [Email](mailto:muhiladitya@outlook.com)

---

## License

This project is for academic research purposes. All simulation code is original work.

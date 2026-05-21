# ForgeWM — Sheaf-Cohomology-Guided Physical Law Discovery

[![Paper](https://img.shields.io/badge/paper-JMLR%20submission-blue)](paper/ForgeWM_Paper1_JMLR.pdf)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](https://python.org)
[![Colab](https://img.shields.io/badge/experiments-Colab%20T4-orange)](https://colab.research.google.com)

> **"Sheaf-Cohomology-Guided Physical Anomaly Discovery: Topological Identification of Missing Constitutive Laws"**
> Submitted to Journal of Machine Learning Research (JMLR), 2026.

---

## What this is

ForgeWM identifies where existing physical axiom systems are incomplete — specifically, where a regime transition exists in simulation data that no current governing equation explains. The framework uses sheaf cohomology to formalise this gap as an algebraic object, persistent homology to locate it from finite FEM samples, and symbolic regression (PySR) to recover the missing equation.

**Three results from the paper:**
- Regime transition threshold recovered with **0.07% error** against known ground truth (Experiment 1)
- Topology-guided sampling requires **1.84× fewer FEM evaluations** than random sampling, p = 0.00036 (Experiment 3)
- Framework correctly **diagnoses information-limited recovery** when the feature ceiling precludes reliable symbolic extraction (Experiment 2)

---

## Repository structure

```
forgwm/
├── paper/                  LaTeX source + compiled PDF
├── experiments/            Colab notebooks (T4 GPU, self-contained)
├── figures/                All publication figures
└── spec/                   Full system specification and theoretical notes
```

---

## Reproducing the experiments

All three experiments run on **Google Colab T4** (free tier). No local installation required.

### Experiment 1 — Synthetic transition recovery (~4 hours)

Open `experiments/ForgeWM_Experiment1_Synthetic_Validation.ipynb` in Colab.
Set runtime to T4 GPU. Run all cells top to bottom.

**Expected output:** `Recovery error: 0.07%  Theorem 1: PASS`

### Experiment 3 — Ablation study (~6–8 hours)

Open `experiments/ForgeWM_Experiment3_Ablation.ipynb` in Colab.
Set runtime to T4 GPU. Run all cells top to bottom.

**Expected output:** `Efficiency ratio: 1.84×  p-value: 0.00036  Significant: true`

### Experiment 2 — Active matter constitutive recovery (~18–24 hours, multi-session)

Open `experiments/ForgeWM_Experiment2_CLEAN.ipynb` in Colab.
Requires Google Drive mount (`MyDrive/ForgeWM_E2/` created automatically).
Checkpoints every 100 samples — safe to disconnect and resume.

**Expected output:** Feature ceiling R²=0.703, ForgeWM R²~0.52, diagnostic finding reported.

---

## Dependencies

All installed automatically in the notebooks. For reference:

```bash
pip install ripser persim pysr fenics-dolfinx the_well \
            scikit-learn scipy matplotlib numpy pandas tqdm
```

---

## Theoretical background

The framework rests on three theorems proved in the paper:

| Theorem | Content |
|---|---|
| 1 | Regime transitions ↔ non-trivial $H^0(X, \mathcal{F})$ generators |
| 2 | Persistent cohomology converges with $n = \mathcal{O}(\varepsilon^{-d} \log(1/\delta))$ FEM samples |
| 3 | Discovery gradient exists and is computable via Hodge decomposition |

Full proofs in `paper/ForgeWM_Paper1_JMLR.pdf`.

---

## Citation

If you use this work, please cite:

```bibtex
@article{forgwm2026,
  title   = {Sheaf-Cohomology-Guided Physical Anomaly Discovery:
             Topological Identification of Missing Constitutive Laws},
  author  = {Blackl1stV35},
  journal = {Journal of Machine Learning Research},
  year    = {2026},
  note    = {Under review}
}
```

---

## This paper is part of a series

- **Paper 1** (this repository): sheaf cohomology + topology-guided discovery
- **Paper 2** (in preparation): causal certification via exact FEM interventions
- **Paper 3** (in preparation): PSPACE-completeness of physical discovery and irreducible boundary

---

## Contact

Open an issue for questions about the code or experiments.
For correspondence about the theoretical results, use the email in the paper submission system.

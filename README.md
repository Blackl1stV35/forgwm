# ForgeWM — Autonomous Physical Law Discovery

[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](https://python.org)
[![Colab](https://img.shields.io/badge/experiments-Colab%20T4-orange)](https://colab.research.google.com)
[![Paper 1](https://img.shields.io/badge/Paper%201-JMLR%20under%20review-blue)](paper/paper1/)
[![Paper 2](https://img.shields.io/badge/Paper%202-JMLR%20under%20review-blue)](paper/paper2/)
[![Paper 3](https://img.shields.io/badge/Paper%203-JMLR%20under%20review-blue)](paper/paper3/)

> **A three-paper series on the mathematics of discovering physical laws that lie outside any given axiom system — and the precise boundary where such discovery ends.**

---

## The question

Given a physical simulation and an existing set of governing equations, how do we know what is *missing* — and how do we prove that the missing part is genuinely new rather than something we simply failed to derive?

This series constructs a complete mathematical answer, building from topology through causality to computability, validated experimentally on FEM material systems running on a single GPU.

---

## The series

### Part 1 — Where the axiom system fails
**"Sheaf-Cohomology-Guided Physical Anomaly Discovery"**
*Under review, JMLR 2026*

Formalises physical regime transitions as non-trivial generators of H⁰(X, F) of a physical observation sheaf. Proves persistent cohomology on finite FEM samples converges to true sheaf cohomology. Demonstrates topology-guided sampling reduces FEM evaluation budget by 1.84× (p = 0.00036).

→ [`paper/paper1/`](paper/paper1/)

### Part 2 — Proving the discovery is real
**"Exact Non-Derivability Certification via Causal Intervention"**
*Under review, JMLR 2026*

Shows FEM Dirichlet boundary conditions constitute perfect do-calculus operations. The Causal Lifting Theorem issues a formal A ⊬ h* proof requiring two FEM evaluations. Validated at p = 8.1×10⁻²⁷ on a thermal softening mechanism unknown to the axiom system.

→ [`paper/paper2/`](paper/paper2/)

### Part 3 — The boundary of what is discoverable
**"PSPACE-Completeness of Physical Law Discovery and the Irreducible Boundary"**
*Under review, JMLR 2026*

Proves physical discovery is PSPACE-complete and terminates in ≤ cd(X) ≤ 40 iterations. Identifies the exact irreducible boundary — causally decoupled variables — as the physical analogue of Gödel incompleteness, proved sharp. Minimal vocabulary extensions computable in polynomial time.

→ [`paper/paper3/`](paper/paper3/)

---

## Key results across the series

| Result | Paper | Value |
|---|---|---|
| Regime transition recovery error | 1 | **0.07%** |
| TDA sampling efficiency gain | 1 | **1.84×** (p=0.00036) |
| Causal certificate p-value | 2 | **8.1×10⁻²⁷** |
| DAG recovery F1 | 2 | **1.0** |
| PC convergence threshold | 2 | **n≈530 FEM samples** |
| Faithfulness violations | 2 | **0/20 configs** |
| Termination bound | 3 | **≤ cd(X) ≤ 40 iterations** |
| Discovery complexity | 3 | **PSPACE-complete** |
| Vocabulary extension | 3 | **Polynomial time** |
| Damage variable r correlation | 3 | **r = 1.0** |
| Irreducible boundary | 3 | **γ = 0 (causally decoupled)** |

---

## Repository structure

```
forgwm/
├── paper/
│   ├── paper1/                 Part 1: topology-guided discovery
│   │   ├── README.md
│   │   ├── ForgeWM_Paper1_JMLR.pdf
│   │   └── ForgeWM_Paper1_JMLR.tex
│   ├── paper2/                 Part 2: causal certification
│   │   ├── README.md
│   │   ├── ForgeWM_Paper2_JMLR.pdf
│   │   └── ForgeWM_Paper2_JMLR.tex
│   ├── paper3/                 Part 3: computability limits
│   │   ├── README.md
│   │   ├── ForgeWM_Paper3_JMLR.pdf
│   │   └── ForgeWM_Paper3_JMLR.tex
│   ├── forgwm_refs.bib         Shared bibliography (40 entries)
│   └── jmlr2e.sty
├── experiments/
│   ├── paper1/                 E1: synthetic transition
│   │   ├── Experiment1_Synthetic_Validation.ipynb
│   │   ├── Experiment2_Constitutive_Recovery.ipynb
│   │   └── Experiment3_Ablation.ipynb
│   ├── paper2/                 E4–E6: causal certification
│   │   ├── Experiment4_Faithfulness.ipynb
│   │   ├── Experiment5_CausalCertification.ipynb
│   │   └── Experiment6_SampleComplexity.ipynb
│   └── paper3/                 E7–E9: PSPACE + boundary
│       ├── Experiment7_PSPACE_Verification.ipynb
│       ├── Experiment8_BoundaryCharacterisation.ipynb
│       └── Experiment9_Termination.ipynb
├── spec/
│   ├── ForgeWM_System_Spec.md      Full engineering specification
│   └── ForgeWM_Research_Paper.md   Complete theoretical framework
├── README.md
├── LICENSE
└── .gitignore
```

---

## Reproducing all experiments

Every experiment runs on **Google Colab T4** (free tier). No local installation needed.

| Experiment | Paper | Runtime | Notes |
|---|---|---|---|
| E1: Synthetic transition | 1 | ~4h | single session |
| E2: Constitutive recovery | 1 | ~24h | multi-session, Drive checkpoint |
| E3: Ablation | 1 | ~8h | single session |
| E4: Faithfulness | 2 | ~2h | single session |
| E5: Causal certification | 2 | ~4h | single session — core result |
| E6: Sample complexity | 2 | ~4h | single session |
| E7: PSPACE verification | 3 | ~4h | single session |
| E8: Boundary characterisation | 3 | ~4h | single session |
| E9: Termination | 3 | ~4h | single session |

---

## The mathematical arc

```
Part 1: WHERE is the axiom system incomplete?
  H⁰(X,F) ≠ 0  ←→  missing law at regime boundary
  Persistent cohomology + residual guidance locates it

         ↓

Part 2: IS the discovery genuinely new?
  FEM do(X=x) = exact Pearl do-calculus
  P(Y|do(x₁)) ≠ P(Y|do(x₂))  →  A ⊬ (X causes Y)
  Certificate: p = 8.1×10⁻²⁷

         ↓

Part 3: HOW FAR and WHERE does it end?
  Physical discovery ∈ PSPACE-complete  (decidable)
  Terminates in ≤ cd(X) ≤ 40 iterations
  Irreducible limit: causally decoupled variables
  = exact Gödel horizon of computable physics
```

---

## Submission status

| Paper | Journal | Status |
|---|---|---|
| Part 1 | JMLR | Under review |
| Part 2 | JMLR | Under review |
| Part 3 | JMLR | Under review |

---

## Citation

```bibtex
@article{forgwm2026part1,
  title   = {Sheaf-Cohomology-Guided Physical Anomaly Discovery},
  author  = {Anonymous},
  journal = {Journal of Machine Learning Research},
  year    = {2026},
  note    = {Under review. Part 1 of the ForgeWM series}
}

@article{forgwm2026part2,
  title   = {Exact Non-Derivability Certification via Causal Intervention},
  author  = {Anonymous},
  journal = {Journal of Machine Learning Research},
  year    = {2026},
  note    = {Under review. Part 2 of the ForgeWM series}
}

@article{forgwm2026part3,
  title   = {PSPACE-Completeness of Physical Law Discovery
             and the Irreducible Boundary},
  author  = {Anonymous},
  journal = {Journal of Machine Learning Research},
  year    = {2026},
  note    = {Under review. Part 3 of the ForgeWM series}
}
```

---

## Contact

Open an issue for questions about code or experiments.
For theoretical correspondence use the email in the JMLR submission system.

*All theorems original. All experiments reproducible on free hardware. All three papers under review.*
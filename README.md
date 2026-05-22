# ForgeWM — Autonomous Physical Law Discovery

[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](https://python.org)
[![Colab](https://img.shields.io/badge/experiments-Colab%20T4-orange)](https://colab.research.google.com)
[![Series](https://img.shields.io/badge/papers-3%20part%20series-purple)](#the-series)

> **A three-paper series on the mathematics of discovering physical laws that lie outside any given axiom system — and the precise boundary where such discovery ends.**

---

## The question

Given a physical simulation and an existing set of governing equations, how do we know what is *missing* — and how do we prove that the missing part is genuinely new rather than something we simply failed to derive?

This series constructs a complete mathematical answer, building from topology through causality to computability, and validates each result experimentally on FEM material systems running on a single GPU.

---

## The series

### Part 1 — Where the axiom system fails
**"Sheaf-Cohomology-Guided Physical Anomaly Discovery"**

Formalises physical regime transitions as non-trivial generators of the cohomology group H⁰(X, F) of a physical observation sheaf. Proves persistent cohomology on finite FEM samples converges to the true sheaf cohomology. Demonstrates topology-guided sampling reduces FEM evaluation budget by 1.84× (p = 0.00036).

→ [`paper/paper1/`](paper/paper1/)

### Part 2 — Proving the discovery is real
**"Exact Non-Derivability Certification via Causal Intervention"**

Shows that FEM Dirichlet boundary conditions constitute perfect do-calculus operations, converting causal discovery from a statistical argument into an exact certificate. The Causal Lifting Theorem issues a formal A ⊬ h* proof requiring two FEM evaluations. Validated at p = 8.1×10⁻²⁷ on a thermal softening mechanism unknown to the axiom system.

→ [`paper/paper2/`](paper/paper2/)

### Part 3 — The boundary of what is discoverable
**"PSPACE-Completeness of Physical Law Discovery and the Irreducible Boundary"** *(in preparation)*

Proves the physical discovery problem is PSPACE-complete and identifies the exact irreducible boundary: variables causally decoupled from all observables are formally undiscoverable. This is the physical analogue of Gödel incompleteness, proved sharp.

→ [`paper/paper3/`](paper/paper3/) *(coming)*

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
| Discovery complexity | 3 | **PSPACE-complete** |
| Irreducible boundary | 3 | Causally decoupled variables |

---

## Repository structure

```
forgwm/
├── paper/
│   ├── paper1/                 Part 1: topology-guided discovery
│   ├── paper2/                 Part 2: causal certification
│   ├── paper3/                 Part 3: computability limits (in prep)
│   ├── forgwm_refs.bib         Shared bibliography
│   └── jmlr2e.sty
├── experiments/
│   ├── paper1/                 Experiments 1–3
│   ├── paper2/                 Experiments 4–6
│   └── paper3/                 Experiments 7–9 (in prep)
├── spec/
│   ├── ForgeWM_System_Spec.md
│   └── ForgeWM_Research_Paper.md
├── README.md
├── LICENSE
└── .gitignore
```

---

## Reproducing all experiments

Every experiment runs on **Google Colab T4** (free tier).

| Experiment | Paper | Runtime | Session |
|---|---|---|---|
| E1: Synthetic transition | 1 | ~4h | single |
| E2: Constitutive recovery | 1 | ~24h | multi + Drive |
| E3: Ablation | 1 | ~8h | single |
| E4: Faithfulness | 2 | ~2h | single |
| E5: Causal certification | 2 | ~4h | single |
| E6: Sample complexity | 2 | ~4h | single |
| E7–E9 | 3 | ~12h | single |

---

## The mathematical arc

```
Part 1: WHERE is the axiom system incomplete?
  H⁰(X,F) ≠ 0  ←→  missing law at regime boundary
  Persistent cohomology locates it from finite FEM samples

         ↓

Part 2: IS the discovery genuinely new?
  FEM do(X=x)  =  exact Pearl do-calculus
  P(Y|do(x₁)) ≠ P(Y|do(x₂))  →  A ⊬ (X causes Y)

         ↓

Part 3: HOW FAR can this go?
  Physical discovery ∈ PSPACE-complete
  Irreducible limit = causally decoupled variables
  = Gödel horizon of computable physics
```

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
```

---

## Contact

Open an issue for questions. For theoretical correspondence, use the email in the JMLR submission system.

*All theorems original. All experiments reproducible on free hardware.*
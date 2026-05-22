# Part 1 — Sheaf-Cohomology-Guided Physical Anomaly Discovery

**"Sheaf-Cohomology-Guided Physical Anomaly Discovery: Topological Identification of Missing Constitutive Laws"**
*Submitted to JMLR, 2026.*

---

## What this paper proves

Three theorems, all experimentally validated:

**Theorem 1** — Physical regime transitions correspond to non-trivial generators of H⁰(X, F), the zeroth cohomology group of the physical observation sheaf. Each generator is exactly one missing law.

**Theorem 2** — Persistent cohomology on n finite FEM samples converges to the true H⁰ in bottleneck distance, with sample complexity n = O(ε⁻ᵈ log(1/δ)).

**Theorem 3** — A computable gradient toward H⁰ void regions exists via the Hodge decomposition, enabling topology-guided sampling.

## Results

| Experiment | Claim | Result |
|---|---|---|
| E1: Elastic-plastic transition | H⁰ encodes missing law | λ* recovery error **0.07%** |
| E2: Active matter recovery | Residual guidance identifies anomaly region | Feature ceiling diagnosis correct |
| E3: Ablation study | TDA guidance reduces FEM budget | **1.84×** efficiency, p=0.00036 |

## Running the experiments

All notebooks in `experiments/paper1/`. Runtime on Colab T4:
- **E1**: ~4 hours, single session
- **E2**: ~24 hours, multi-session with Google Drive checkpointing
- **E3**: ~8 hours, single session

## Files

```
paper1/
├── ForgeWM_Paper1_JMLR.pdf     Compiled paper
├── ForgeWM_Paper1_JMLR.tex     LaTeX source
└── figures/
    ├── fig1.png                Experiment 1: E-normalised recovery
    ├── fig2.png                Experiment 2: Constitutive recovery
    └── fig3.png                Experiment 3: Ablation
```

## Recompile

```bash
cd paper/paper1
pdflatex ForgeWM_Paper1_JMLR.tex
bibtex ForgeWM_Paper1_JMLR
pdflatex ForgeWM_Paper1_JMLR.tex
pdflatex ForgeWM_Paper1_JMLR.tex
```

Requires: `jmlr2e.sty` (in `paper/`), `forgwm_refs.bib` (in `paper/`), figure PNGs in same directory.

## Key honest finding

The correct homology degree for a constitutive regime boundary is H⁰ (connected component separation), not H¹ (loops). Naive H¹ application produced 13,486 noise features. E-normalisation + H⁰ breakpoint detection gave 0.07% error. This self-correction is reported in Remark 1 of the paper.
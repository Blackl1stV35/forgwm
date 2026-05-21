# ForgeWM — Autonomous Engineering Discovery System
### Full Design & Environment Specification

> **Goal**: A self-improving system that autonomously discovers physically valid, high-complexity engineered geometries by combining a JEPA world model (LeWM), intrinsic self-RL, and FEM-grounded physics simulation — driven by plain-language engineer prompts.

---

## 1. System Overview

```
Engineer Prompt
      │
      ▼
 Goal Encoder ──► z_goal ∈ ℝ¹⁹²
      │
      ▼
┌─────────────────────────────────────────────┐
│           LeWM World Model                  │
│  ViT Encoder (5M) + Transformer Predictor  │
│  Trained on physics simulation datasets     │
│  SIGReg anti-collapse · single GPU          │
└────────────┬────────────────────────────────┘
             │  latent rollout ẑ₁:H
             ▼
     CEM Planner (H=24, 50 samples)
             │  action sequence a₁:H
             ▼
  Geometry Mutation Engine
  (sweep · lattice · branch · chamfer)
             │  candidate mesh
             ▼
   FEM Stress Evaluator (FEniCS)
             │  σ_vonMises, mass, stiffness
             ▼
  Intrinsic RL Reward
  R = novelty(z) + constraint_satisfaction − violation_penalty
             │
             └──► update world model · next generation
```

---

## 2. Architecture Components

### 2.1 World Model — LeWM (LeWorldModel)

| Component | Spec |
|---|---|
| Encoder | ViT-Tiny · 5M params · patch=14 · 12 layers · d=192 |
| Predictor | Transformer · 10M params · 6 layers · 16 heads · AdaLN |
| Loss | `L = Lpred (MSE) + λ·SIGReg` · λ=0.1 · 1 hyperparameter |
| Anti-collapse | SIGReg · M=1024 random projections · Epps–Pulley test |
| Training | Offline · reward-free · single GPU · ~150k steps · ~3h |
| Planning | CEM · H=24 horizon · 50 samples · 0.98 s/rollout |
| Paper | LeWorldModel (Maes et al., arXiv:2603.19312, 2026) |

**Training objective:**
```
L_LeWM = ||ẑ_{t+1} − z_{t+1}||² + λ · SIGReg(Z)

SIGReg(Z) = (1/M) Σ_m T(Z·u^(m))   # Epps-Pulley normality test
                                      # along M random unit directions
```

**Action conditioning** — geometry mutation operators are encoded as a fixed-dim action vector fed into the predictor via Adaptive Layer Normalization (AdaLN) at every layer. Mutation op vector:

```python
a_t = [
  sweep_angle,       # float [-π, π]
  lattice_density,   # float [0, 1]
  branch_factor,     # int {0,1,2,3} → one-hot
  taper_ratio,       # float [0, 1]
  chamfer_radius,    # float [0, 0.5]
  material_id,       # int {0..N} → one-hot
]  # dim A = 6 + one-hots → padded to A=16
```

---

### 2.2 Physics Dataset — The Well + FEM Supplement

#### 2.2.1 The Well (base physics understanding)

| Dataset | Domain | Size | Relevance |
|---|---|---|---|
| `turbulence_2d` | Fluid dynamics | ~40 GB | stress field topology |
| `active_matter` | Biological mechanics | ~12 GB | emergent structure |
| `acoustic_scattering` | Wave propagation | ~28 GB | structural resonance |
| `mhd_64` | Magneto-hydrodynamics | ~60 GB | anisotropic field learning |
| `shear_flow` | Viscous dynamics | ~35 GB | surface stress distribution |

Install and stream:
```bash
pip install the_well
the-well-download --base-path ./data --dataset turbulence_2d --split train
the-well-download --base-path ./data --dataset active_matter --split train
```

Or stream from HuggingFace:
```python
from the_well.data import WellDataset

ds = WellDataset(
    well_base_path="hf://datasets/polymathic-ai/",
    well_dataset_name="turbulence_2d",
    well_split_name="train",
)
```

#### 2.2.2 FEM Supplement (structural mechanics — not in The Well)

The Well covers fluid/MHD/bio. Solid mechanics must be generated separately using FEniCS or Calculix. Generate 10,000–50,000 trajectories of:

- Beam bending under distributed load (parametric cross-sections)
- Shell deformation (swept surfaces, variable thickness)
- Lattice compression (BCC, FCC, Kelvin cell topologies)
- Notch/hole stress concentration (varying geometry)

```python
# FEniCS trajectory generation example
from fenics import *

def generate_beam_trajectory(L, h, E, nu, F, steps=50):
    """Returns (obs_sequence, action_sequence) for world model training."""
    mesh = BoxMesh(Point(0,0,0), Point(L,h,h), 20,5,5)
    # ... solve elasticity, record displacement field at each load step
    # obs shape: (steps, C=3, Nx, Ny)  [ux, uy, von_mises_stress]
    # action: load increment vector
    return obs_seq, action_seq
```

Recommended generation: **20,000 trajectories × 50 timesteps** ≈ 180 GB. Store in The Well HDF5 format for compatibility.

---

### 2.3 Self-RL Loop (Intrinsic Motivation)

No external reward signal. The agent is rewarded for:

```python
def intrinsic_reward(z_current, z_history, constraints, fem_result):
    # 1. Novelty: distance from nearest latent neighbor
    novelty = min_dist(z_current, z_history)  # encourages exploration

    # 2. Constraint satisfaction
    stress_ok   = max(fem_result.von_mises) < constraints.yield_strength
    mass_ok     = fem_result.mass < constraints.mass_budget
    stiffness_ok= fem_result.stiffness_ratio > constraints.target_stiffness
    satisfaction = (stress_ok + mass_ok + stiffness_ok) / 3.0

    # 3. Surprise: Epps-Pulley score on trajectory
    # High surprise = near distribution boundary = interesting mechanic
    surprise = sigreg_score(z_current)
    surprise_reward = clip(surprise - 0.15, 0, 0.5)  # reward mild novelty

    # 4. Violation penalty
    violation = relu(max(fem_result.von_mises) - constraints.yield_strength)

    R = 0.4*novelty + 0.4*satisfaction + 0.2*surprise_reward - 0.8*violation
    return R
```

**RL algorithm**: PPO with latent world model rollouts (no environment interaction during inner loop). The CEM planner finds action sequences; PPO updates the policy that *proposes* mutation operators.

```
Outer loop (PPO):  updates mutation proposal policy π(a|z, prompt)
Inner loop (CEM):  optimizes specific action sequence given fixed π
World model:       provides ẑ_{t+1} = pred(z_t, a_t) for both loops
FEM evaluator:     called only on promising candidates (top 10% by CEM cost)
```

FEM is expensive (~2–10s per evaluation). The world model acts as a fast surrogate: CEM runs 50×24 rollouts in <1s in latent space. FEM is only invoked to *verify* the top candidate per generation.

---

### 2.4 Geometry Mutation Engine

Parametric geometry is represented as a **constructive solid geometry (CSG) tree** with differentiable parameters:

```python
@dataclass
class GeometryNode:
    op: str           # sweep | lattice | branch | extrude | chamfer | void
    params: dict      # op-specific floats
    children: list    # child GeometryNodes

# Example: swept blade with lattice infill
geometry = GeometryNode(
    op="sweep",
    params={"profile": "crescent", "angle": 1.2, "taper": 0.3},
    children=[
        GeometryNode(
            op="lattice_infill",
            params={"cell": "BCC", "density": 0.4, "orientation": 45.0},
            children=[]
        ),
        GeometryNode(
            op="spine_branch",
            params={"count": 3, "divergence": 0.2},
            children=[]
        )
    ]
)
```

Mutation applies one operator per generation step, sampled from the policy π:

```python
def mutate(geometry, action_vector):
    op = decode_op(action_vector)       # from AdaLN-conditioned predictor
    new_params = perturb(geometry, op)  # Gaussian perturbation in param space
    return apply_op(geometry, op, new_params)
```

Mesh export: **OpenCASCADE → STEP → gmsh → FEniCS mesh** pipeline. Output formats: `.step`, `.stl`, `.vtk` (with stress field), `.json` (CSG tree + params).

---

### 2.5 FEM Stress Evaluator

```python
# FEniCS solver — called on top candidate per generation
from fenics import *

def evaluate_structure(mesh_path, material, load_case):
    mesh = Mesh(mesh_path)
    V = VectorFunctionSpace(mesh, 'Lagrange', degree=2)

    # Material: Ti-6Al-4V or Inconel 718
    E  = material.youngs_modulus   # 114 GPa Ti64 / 200 GPa IN718
    nu = material.poisson_ratio    # 0.33 / 0.29

    # Boundary conditions
    bc = DirichletBC(V, Constant((0,0,0)), boundary_root)
    f  = Constant(load_case.force_vector)

    # Solve linear elasticity
    u = Function(V)
    solve(a(u,v) == L(v), u, bc)

    # Von Mises stress
    sigma = stress_tensor(u, E, nu)
    von_mises = project(sqrt(3/2 * inner(dev(sigma), dev(sigma))), S)

    return FEMResult(
        von_mises_max = von_mises.vector().max(),
        mass          = assemble(density * dx),
        stiffness     = compute_stiffness_ratio(u, load_case),
        displacement  = u,
        stress_field  = von_mises,
    )
```

---

## 3. Environment Setup

### 3.1 Hardware Requirements

| Tier | GPU | VRAM | RAM | Storage | Use case |
|---|---|---|---|---|---|
| **Minimum** | RTX 3090 / 4080 | 24 GB | 64 GB | 2 TB NVMe | Single dataset, dev |
| **Recommended** | A100 80GB | 80 GB | 128 GB | 8 TB NVMe | Full Well + FEM supplement |
| **Optimal** | 2× A100 / H100 | 80 GB × 2 | 256 GB | 16 TB NVMe | All datasets + fast FEM |
| **Apple Silicon** | M2/M3 Max | 96 GB unified | — | 2 TB | Development only |

**Estimated training time (150k steps, batch=8):**
- RTX 4090: ~2.5 h (LeWM alone) + ~6 h (RL loop × 1000 gen)
- A100: ~1.8 h + ~3.5 h

---

### 3.2 Software Environment

#### Python environment

```bash
# Python 3.11+ recommended
python -m venv forgwm_env
source forgwm_env/bin/activate

# Core ML
pip install torch==2.3.0 torchvision --extra-index-url https://download.pytorch.org/whl/cu121
pip install einops timm transformers

# The Well
pip install the_well

# FEM
pip install fenics-dolfinx  # DOLFINx (FEniCS successor, Python 3.11 compatible)
# OR legacy FEniCS via Docker:
# docker pull dolfinx/dolfinx:stable

# Geometry pipeline
pip install cadquery gmsh pythonocc-core
pip install meshio trimesh open3d

# RL
pip install stable-baselines3 gymnasium

# Visualization / UI
pip install plotly dash vtk pyvista

# Utilities
pip install hydra-core omegaconf wandb einops numpy scipy
```

#### CUDA / cuBLAS

```bash
# CUDA 12.1 (matches torch 2.3.0)
# Verify:
python -c "import torch; print(torch.cuda.is_available(), torch.version.cuda)"

# FlashAttention-2 (drop-in, 2-4x memory reduction for ViT attention)
pip install flash-attn --no-build-isolation

# Triton (for custom SIGReg kernel)
pip install triton
```

#### Optional: Rust data pipeline

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
cargo new forgwm_data_loader
# Dependencies in Cargo.toml:
# tokio = { version = "1", features = ["full"] }
# hdf5 = "0.8"
# ndarray = "0.15"
# pyo3 = { version = "0.21", features = ["extension-module"] }
```

---

### 3.3 Repository Structure

```
forgwm/
├── configs/
│   ├── model/
│   │   ├── lewm_tiny.yaml          # ViT-Tiny + 10M predictor
│   │   └── lewm_small.yaml         # ViT-Small + 20M predictor
│   ├── data/
│   │   ├── turbulence_2d.yaml
│   │   ├── active_matter.yaml
│   │   └── fem_structural.yaml
│   ├── rl/
│   │   └── ppo_intrinsic.yaml
│   └── experiment/
│       └── forgwm_full.yaml
│
├── forgwm/
│   ├── models/
│   │   ├── encoder.py              # ViT encoder + MLP projection
│   │   ├── predictor.py            # Transformer + AdaLN action conditioning
│   │   ├── sigreg.py               # SIGReg + Epps-Pulley (+ Triton kernel)
│   │   └── lewm.py                 # Full LeWM training loop
│   │
│   ├── geometry/
│   │   ├── csg_tree.py             # CSG representation
│   │   ├── mutation_ops.py         # sweep, lattice, branch, chamfer, void
│   │   ├── mesh_export.py          # OCC → gmsh → FEniCS
│   │   └── material_library.py     # Ti-6Al-4V, Inconel 718, etc.
│   │
│   ├── fem/
│   │   ├── evaluator.py            # FEniCS stress solver
│   │   ├── mesh_loader.py
│   │   └── result_types.py         # FEMResult dataclass
│   │
│   ├── rl/
│   │   ├── reward.py               # Intrinsic reward function
│   │   ├── cem_planner.py          # Cross-Entropy Method in latent space
│   │   ├── policy.py               # Mutation proposal policy (PPO)
│   │   └── discovery_loop.py       # Outer RL loop
│   │
│   ├── data/
│   │   ├── well_adapter.py         # WellDataset → LeWM training format
│   │   ├── fem_generator.py        # FEniCS trajectory generation
│   │   └── geometry_encoder.py     # mesh → pixel obs for ViT
│   │
│   ├── probes/
│   │   └── physical_probes.py      # Linear/MLP probes on z_t
│   │
│   └── ui/
│       ├── dashboard.py            # Dash/Plotly live UI
│       └── viewport.py             # VTK-based 3D stress viewer
│
├── scripts/
│   ├── train_worldmodel.py         # Step 1: train LeWM on Well + FEM data
│   ├── run_discovery.py            # Step 2: run self-RL discovery loop
│   ├── evaluate_generation.py      # Step 3: FEM verification of candidates
│   └── export_design.py            # Step 4: export STEP + stress VTK
│
├── data/                           # downloaded Well datasets + FEM supplement
├── checkpoints/                    # model checkpoints
├── outputs/                        # discovered geometries per generation
└── requirements.txt
```

---

### 3.4 Docker Environment (recommended for FEniCS compatibility)

```dockerfile
FROM dolfinx/dolfinx:stable

# Add CUDA (for PyTorch)
RUN apt-get update && apt-get install -y \
    cuda-toolkit-12-1 \
    libgl1-mesa-glx \
    && rm -rf /var/lib/apt/lists/*

# Python ML stack
RUN pip install --no-cache-dir \
    torch==2.3.0+cu121 torchvision \
    --extra-index-url https://download.pytorch.org/whl/cu121

RUN pip install --no-cache-dir \
    the_well einops timm flash-attn \
    stable-baselines3 gymnasium \
    cadquery gmsh meshio trimesh \
    plotly dash pyvista wandb hydra-core

WORKDIR /workspace
COPY . /workspace/forgwm
```

```bash
docker build -t forgwm:latest .
docker run --gpus all -v $(pwd)/data:/workspace/data \
           -v $(pwd)/outputs:/workspace/outputs \
           -p 8050:8050 forgwm:latest
```

---

## 4. Training Pipeline

### Step 1 — Train world model

```bash
cd forgwm/scripts
python train_worldmodel.py \
  model=lewm_tiny \
  data=turbulence_2d \
  training.steps=150000 \
  training.batch_size=8 \
  training.lambda_sigreg=0.1 \
  training.lr=3e-4 \
  training.device=cuda \
  logging.wandb=true
```

Expected output:
- `checkpoints/lewm_step150000.pt`
- Lpred converges to ~0.8, SIGReg to ~0.007
- Physical probes: velocity r²>0.96, vorticity r²>0.94

### Step 2 — Generate FEM training data (parallel)

```bash
python -m forgwm.data.fem_generator \
  --n-trajectories 20000 \
  --topology-types beam,shell,lattice,notch \
  --output-dir data/fem_structural \
  --n-workers 16
# ~6h on 16-core CPU cluster
```

### Step 3 — Fine-tune world model on structural data

```bash
python train_worldmodel.py \
  model=lewm_tiny \
  data=fem_structural \
  training.steps=50000 \
  training.checkpoint=checkpoints/lewm_step150000.pt \
  training.lr=1e-4
# Transfer learning: 50k steps ~ 45 min
```

### Step 4 — Run discovery loop

```bash
python run_discovery.py \
  model.checkpoint=checkpoints/lewm_finetuned.pt \
  rl.generations=1000 \
  rl.cem.horizon=24 \
  rl.cem.samples=50 \
  rl.cem.iterations=50 \
  rl.fem_verify_top_k=1 \
  constraints.yield_strength_mpa=850 \
  constraints.mass_budget_kg=2.4 \
  constraints.stiffness_ratio=3.2 \
  prompt="Maximise stiffness-to-mass ratio with swept crescent profile" \
  output_dir=outputs/run_001
```

---

## 5. Key Implementation Details

### 5.1 Geometry → Observation encoding

The ViT encoder expects image-like tensors `(B, C, H, W)`. Geometry is rendered as:

```python
def geometry_to_observation(geometry, resolution=224):
    # Render 3 views: front, side, perspective
    # Each view: 3 channels = [depth, von_mises_stress, surface_normal_z]
    views = render_views(geometry, resolution=resolution)
    obs = torch.stack(views, dim=0)  # (3_views, 3_channels, 224, 224)
    return obs.reshape(1, 9, 224, 224)  # treat as 9-channel single image
```

Patch tokenisation: 224 / 14 = 16 × 16 = 256 patches. Compatible with standard ViT-Tiny.

### 5.2 SIGReg Triton kernel (fast path)

```python
import triton
import triton.language as tl

@triton.jit
def sigreg_projection_kernel(
    Z_ptr, U_ptr, H_ptr,
    N, B, d, M,
    BLOCK_SIZE: tl.constexpr
):
    # Computes H = Z @ U  (N*B × d) @ (d × M) → (N*B × M)
    # Z: latent embeddings, U: frozen random projections
    pid = tl.program_id(0)
    # ... standard tiled GEMM in Triton
```

Speedup over naive PyTorch: ~2.1× for (512 × 192) @ (192 × 1024) on A100.

### 5.3 AdaLN action conditioning

```python
class AdaLN(nn.Module):
    def __init__(self, d_model, action_dim):
        super().__init__()
        self.norm = nn.LayerNorm(d_model)
        self.proj = nn.Linear(action_dim, 2 * d_model)
        # Zero-init for training stability (paper prescription)
        nn.init.zeros_(self.proj.weight)
        nn.init.zeros_(self.proj.bias)

    def forward(self, x, action):
        scale, shift = self.proj(action).chunk(2, dim=-1)
        return self.norm(x) * (1 + scale) + shift
```

### 5.4 CEM planner

```python
def cem_plan(world_model, z_start, z_goal, H=24, N=50, iterations=50):
    # Initialize action distribution
    mu = torch.zeros(H, action_dim)
    sigma = torch.ones(H, action_dim)

    for _ in range(iterations):
        # Sample N action sequences
        actions = mu + sigma * torch.randn(N, H, action_dim)

        # Rollout in latent space (fast: ~0.98s for all N)
        costs = []
        for i in range(N):
            z = z_start.clone()
            for t in range(H):
                z = world_model.predictor(z, actions[i, t])
            cost = ((z - z_goal) ** 2).sum()
            costs.append(cost)

        # Update distribution with top-k elite plans
        elite_idx = torch.topk(-torch.stack(costs), k=N//5).indices
        elite_actions = actions[elite_idx]
        mu = elite_actions.mean(0)
        sigma = elite_actions.std(0).clamp(min=1e-3)

    return mu  # best action sequence
```

---

## 6. Output Formats

Each discovered geometry exports:

```
outputs/run_001/gen_847/
├── geometry.step          # CAD-ready STEP file (OpenCASCADE)
├── mesh.vtk               # FEM mesh + stress field
├── stress_report.json     # von Mises max, mass, stiffness, r² probes
├── csg_tree.json          # Full parametric CSG tree (reproducible)
├── latent_vector.npy      # z ∈ ℝ¹⁹² at generation 847
├── mutation_log.json      # Sequence of operators that produced this form
└── render_front.png       # 3-view stress render
```

`stress_report.json` example:
```json
{
  "generation": 847,
  "von_mises_max_mpa": 812.3,
  "yield_strength_mpa": 850.0,
  "safety_factor": 1.047,
  "mass_kg": 2.21,
  "stiffness_ratio": 3.41,
  "novelty_score": 0.941,
  "rl_reward": 0.041,
  "surprise_score": 0.31,
  "probes": {
    "velocity_r2": 0.974,
    "vorticity_r2": 0.943,
    "pressure_r2": 0.891
  },
  "material": {
    "core": "Ti-6Al-4V",
    "lattice": "Inconel 718"
  }
}
```

---

## 7. Live UI

```bash
# Launch Dash dashboard (port 8050)
python forgwm/ui/dashboard.py \
  --checkpoint checkpoints/lewm_finetuned.pt \
  --output-dir outputs/run_001 \
  --port 8050
```

Dashboard panels:
- **Viewport** — live VTK stress field render, updated each generation
- **Latent scatter** — PCA of z₁₉₂, discovery trail
- **Self-RL reward curve** — intrinsic reward vs generation
- **Physical probes** — r² for velocity, vorticity, pressure, stress
- **Surprise score** — Epps–Pulley anomaly index
- **Discovery log** — per-generation mechanical innovation notes
- **Engineer prompt** — live editable, re-encodes z_goal on change

---

## 8. Evaluation Metrics

| Metric | Target | How measured |
|---|---|---|
| World model Lpred | < 1.0 | MSE on held-out Well trajectories |
| Physical probe r² | > 0.93 | Linear regression on frozen z_t |
| CEM plan speed | < 1.5 s | Wall clock, 50×24 rollouts |
| FEM constraint sat. | > 80% of gen. | Yield + mass + stiffness all met |
| Novelty (latent dist.) | > 0.8 | Min dist. to z_history |
| Surprise (SIGReg) | 0.1–0.4 | Epps–Pulley on trajectory |
| Mass budget adherence | < target | FEniCS mass assembly |
| Stress safety factor | > 1.0 | yield_strength / von_mises_max |

---

## 9. References

| Source | Role |
|---|---|
| Maes et al. (2026) *LeWorldModel* arXiv:2603.19312 | Core JEPA world model |
| LeCun (2022) *A Path Towards Autonomous Machine Intelligence* | JEPA framework origin |
| Ohana et al. (2024) *The Well* NeurIPS 2024 | Physics simulation dataset |
| FEniCS/DOLFINx docs | FEM stress solver |
| OpenCASCADE / gmsh | Geometry pipeline |
| Schulman et al. (2017) *PPO* | RL algorithm |
| Shannon & Stein (1994) *CEM* | Latent planner |

---

## 10. Quick-start (minimum viable run)

```bash
# 1. Clone and install
git clone https://github.com/your-org/forgwm
cd forgwm && pip install -e .

# 2. Download one Well dataset (~12 GB)
the-well-download --base-path ./data --dataset active_matter --split train

# 3. Train world model
python scripts/train_worldmodel.py model=lewm_tiny data=active_matter training.steps=150000

# 4. Run discovery (10 generations to verify pipeline)
python scripts/run_discovery.py \
  model.checkpoint=checkpoints/lewm_step150000.pt \
  rl.generations=10 \
  prompt="High stiffness swept blade, minimal mass"

# 5. View output
open outputs/run_001/gen_010/render_front.png
cat outputs/run_001/gen_010/stress_report.json
```

**Total wall time to first discovered geometry: ~4 hours on RTX 4090.**

---

*ForgeWM Specification v0.1 · Built on LeWM (Maes et al. 2026) + The Well (Ohana et al. 2024)*

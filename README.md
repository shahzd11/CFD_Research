# Ansys PyFluent — CFD Automation & DMD/SVD Analysis

A fully automated, GUI-free Python framework for running CFD simulations and performing data-driven reduced-order modelling using **Ansys PyFluent**. The framework couples high-fidelity URANS simulations with **Dynamic Mode Decomposition (DMD)** and **Singular Value Decomposition (SVD)** to extract dominant coherent flow structures from time-resolved snapshots — all within a single reproducible Jupyter environment.

---

## What's Included

### 1. `cylinder-flow-pyfluent/` — Transient Cylinder Flow (Primary Case)
The core validation case: transient URANS simulation of laminar vortex shedding over a 2D circular cylinder at Re = 150.

**What it does:**
- Launches Ansys Fluent in headless (no-GUI) mode via PyFluent
- Automatically configures boundary conditions, prism layer meshing, and CFL-based time-step selection
- Runs the transient solver and autosaves flow snapshots at constant time intervals
- Validates drag coefficient (C_D) and Strouhal number (St) against NASA reference data (error < 2%)
- Performs SVD and DMD on the snapshot matrix to extract vortex shedding modes

**Key files:**
```
cylinder-flow-pyfluent/
├── Transient Simulation/
│   └── AnsysCylinderFlowLab - Transient.ipynb   ← main notebook
├── CylinderFlowLab_Mesh.cas.h5                  ← mesh file (download separately, see below)
```

🔗 [Open Transient Simulation Notebook](https://github.com/shahzd11/Ansys_PyFluent/tree/main/cylinder-flow-pyfluent/Transient%20Simulation)

---

### 2. `NACA0012/` — Airfoil Flow Simulation
Extension of the framework to turbulent external aerodynamic flow over the NACA 0012 airfoil.

**What it does:**
- C-mesh topology generation with y⁺ = 1 wall spacing for turbulent Re
- k-ω SST turbulence model
- Transient URANS with snapshot collection after the flow development transient
- DMD modal analysis of trailing-edge shedding dynamics

---

## Environment Setup

### Requirements
- Ansys Fluent (licensed installation) — 2023 R1 or later recommended
- Conda (Miniconda or Anaconda)
- Python 3.9+

### Installation

**1. Clone the repository:**
```bash
git clone https://github.com/shahzd11/Ansys_PyFluent.git
cd Ansys_PyFluent
```

**2. Create and activate the Conda environment:**
```bash
conda env create -f environment.yml
conda activate <env_name>
```
> Replace `<env_name>` with the name listed at the top of `environment.yml`.

**Key dependencies installed:**
- `ansys-fluent-core` — PyFluent API
- `numpy`, `scipy` — numerical computation and SVD/DMD
- `matplotlib` — post-processing plots
- `jupyterlab` / `notebook` — interactive execution
- `h5py` — reading `.dat.h5` snapshot files

---

## Running the Simulations

### ⚠️ Download Mesh Files First
Large mesh and case files (`.cas.h5`) are **not stored in this repository** due to GitHub's file size limits. You must obtain and place them locally before running any simulation notebook.

| File | Place in folder |
|------|----------------|
| `CylinderFlowLab_Mesh.cas.h5` | `cylinder-flow-pyfluent/` |

Or update the file path variable at the top of the relevant notebook to point to your local download location.

---

### Running the Cylinder Transient Simulation

```bash
cd cylinder-flow-pyfluent
jupyter notebook
```

Open **`AnsysCylinderFlowLab - Transient.ipynb`** and run cells sequentially.

The notebook is structured in six self-contained modules:

| Module | What it does |
|--------|-------------|
| 1 | Launch Fluent in headless mode |
| 2 | Load mesh, configure boundary conditions and prism layers |
| 3 | Run steady RANS initialisation, then switch to transient URANS |
| 4 | Execute time-stepping loop with autosave of flow snapshots |
| 5 | Validate C_D and Strouhal number against NASA reference |
| 6 | Build snapshot matrix → SVD → DMD → eigenvalue analysis → flow reconstruction |

---

### Running the NACA 0012 Simulation

```bash
cd NACA0012
jupyter notebook
```

Set the target angle of attack (α) and Reynolds number at the top of the notebook before running. The simulation uses a two-step initialisation: converge at α = 0° first, then continue to the target angle for robust convergence near stall.

---

## Output Files

After running, each simulation produces:

- **`.dat.h5` snapshots** — velocity field data at each autosave time step
- **Force monitor CSVs** — C_D and C_L time history
- **DMD mode plots** — spatial mode contours and eigenvalue spectrum
- **SVD energy plots** — singular value decay and cumulative energy

---

## Author

**Dhvani Shah** · Roll No. 24110325  
Department of Mechanical Engineering  
Indian Institute of Technology Gandhinagar  
📧 [dhvani.shah@iitgn.ac.in](mailto:dhvani.shah@iitgn.ac.in)

---

## References

- Schmid, P. J. (2010). Dynamic mode decomposition of numerical and experimental data. *Journal of Fluid Mechanics*, 656, 5–28.
- Rumsey, C. L. et al. (2004). *NASA/TM-2004-212879* — Cylinder vortex shedding reference data.
- Ladson, C. L. (1988). *NASA TM-4074* — NACA 0012 experimental aerodynamic data.

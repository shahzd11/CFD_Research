# Integrated CFD–ML Framework: PyFluent Automation with POD/DMD Reduced-Order Modelling

A GUI-free Python framework that couples high-fidelity **Ansys Fluent** simulations (automated end-to-end via **PyFluent**) with data-driven **reduced-order modelling** — **Proper Orthogonal Decomposition (POD/SVD)** and **Dynamic Mode Decomposition (DMD)**. The framework takes time-resolved flow snapshots from URANS simulations and extracts the dominant coherent structures, builds low-dimensional surrogate flow-field predictors, and validates every case quantitatively against primary literature.

The project progresses through four cases of increasing physical complexity: a lid-driven cavity (pipeline commissioning), laminar and turbulent flow over a circular cylinder (canonical vortex shedding), and transonic shock buffet on a NACA 0012 airfoil (self-excited unsteady aerodynamics). Every case shares the same automation backbone and the same modal-analysis post-processing, so results are directly comparable across flow regimes.

---

## The Four Cases

### 1. Lid-Driven Cavity — Pipeline Commissioning

The classic Ghia et al. (1982) benchmark, used to verify the PyFluent automation independently of turbulence modelling. A unit-square cavity is defined entirely through the PyFluent API with no Workbench step, meshed on a structured 129×129 Cartesian grid, and solved as a steady laminar SIMPLE case. The primary recirculation vortex and secondary corner eddies are validated against the reference streamfunction and centreline velocity profiles. This case exists to confirm that programmatic setup, moving-wall boundary conditions, solution control, and automated post-processing all work correctly before any of the harder physics is attempted.

### 2. Circular Cylinder — Laminar (Re = 150)

The core validation case. Transient URANS of laminar Kármán vortex shedding over a 2-D circular cylinder. The pipeline launches Fluent headless, configures boundary conditions and prism layers, runs a steady precursor, switches to a second-order transient solve, and autosaves flow-field snapshots at fixed intervals once the wake reaches its limit cycle.

**Validated results:**
- Strouhal number **St ≈ 0.164** (within the Wind-US computed band 0.150–0.183; ~10% below the Roshko 1954 experimental 0.179–0.182)
- Drag coefficient **C_D ≈ 1.25** (consistent with Henderson 1995 incompressible DNS 1.28–1.40)
- **POD** captures ~99% of fluctuation energy in **3 modes**
- **DMD** reconstruction error **4.34%**; out-of-sample kinetic-energy prediction error **~0.002%**

Validated against the NASA NPARC Alliance V&V archive (laminar cylinder, Re = 150), Roshko (1954), and Henderson (1995).

### 3. Circular Cylinder — Turbulent (Re = 3900, k-ω SST)

Extension to a turbulent wake. Same pipeline, k-ω SST closure, second-order URANS. This case documents an important modelling finding rather than a clean textbook result: **vanilla 2-D URANS SST suppresses vortex shedding on a smooth cylinder** — the wake relaxes back to a steady symmetric state (flat lift, over-predicted drag). The fix, identified from the literature, is to enable two sub-options that plain SST omits:

- **Smirnov–Menter curvature correction** — reduces eddy viscosity in the high-curvature regions where vortices roll up
- **Kato–Launder production limiter** — removes the spurious stagnation-point turbulent-kinetic-energy over-production

With these enabled at Re = 3900, the case targets the Parnaudeau et al. (2008) / Norberg (1987) benchmark (experimental **St ≈ 0.215**, **C_D ≈ 0.99**), with 2-D URANS expected to over-predict drag to ~1.2–1.5. The shedding-suppression behaviour is written up as a documented modelling limitation, validating St (a kinematic quantity that remains comparable) while flagging C_D as a 2-D approximation.

Validated against Achenbach (1971), Parnaudeau et al. (2008), and Norberg (1987).

### 4. NACA 0012 Airfoil — Transonic Shock Buffet

The most advanced case: self-sustained **transonic shock buffet** used as a natural unsteady-aerodynamics mechanism, capturing pitching-like limit-cycle behaviour on a **stationary grid** — no dynamic meshing or mesh deformation required. The shock foot oscillates on the suction side and drives a periodic wake response, giving a clean unsteady flow field for modal analysis.

**Validation target — NASA TP-2485 (McDevitt & Okuno, 1985), Table 4, row 3:**
- M∞ = 0.77, α = 4.0°, Re = 10⁷
- Reduced frequency **f̄ = 0.44** (convention f̄ = 2πfc/U from the TP-2485 nomenclature), giving **St ≈ 0.0700**

**Solver configuration:**
- Density-based implicit, k-ω SST with **curvature correction and compressibility effects** (Kato–Launder is cylinder-specific and explicitly excluded here)
- Pressure-far-field boundaries applying Riemann invariants; ideal-gas density
- Steady precursor → transient development; snapshots extracted only after a stationary limit cycle is confirmed
- Time step sized to ~250 steps per buffet cycle

The expected modal picture differs from the cylinder: POD modes 1–2 capture the shock-foot oscillation (streamwise velocity + suction-side pressure, the bulk of the fluctuation energy), modes 3–4 capture the phase-shifted wake/separation response, and the DMD spectrum shows a dominant complex-conjugate pair near the unit circle at the buffet frequency with weaker harmonics.

Additional context references: Bouhadji & Braza (2003, laminar transonic buffet regime), Zauner et al. (2019), Harris NASA TM-81927 (1981, steady transonic Cp/Cl/Cd), Barakos & Drikakis (2000), Crouch et al. (2009), Jacquin et al. (2009).

> **Note on the validation benchmark.** TP-2485's physical model chord is 20.32 cm (8 inches), but the dimensionless physics is chord-independent. Buffet onset at M = 0.76, Re = 10⁷ is approximately α ≈ 3.5°. The paper explicitly recommends Re = 10⁷ for code validation — far-field pressure settings must give this Reynolds number, not a lower value, before quantitative comparison.

---

## The Machine-Learning Part: POD/SVD & DMD Reduced-Order Modelling

Once a case reaches a statistically stationary limit cycle, the same modal-analysis pipeline is applied to every case. The goal is a **surrogate model** that reconstructs — and predicts — the unsteady flow field from a handful of modes instead of millions of cell values.

### Snapshot Matrix

Autosaved flow fields are assembled into a snapshot matrix **X** whose columns are individual time instants and whose rows are the stacked state vector. The state vector stacks the fields together — **[U; V]** for the cylinders, **[U; V; p′]** or **[U; V; M]** for the airfoil — so that the decomposition captures both the velocity structures and the shock/pressure motion in a single coherent set of modes. A **volume-weighted** formulation (weights = √V per cell) makes the SVD energy inner product physically meaningful on non-uniform meshes. The temporal mean is subtracted before decomposition so the modes describe the *fluctuation*, not the base flow.

### POD via SVD

An SVD of the mean-subtracted, volume-weighted snapshot matrix yields the POD modes (spatial structures), their singular values (energy ranking), and temporal coefficients. The singular-value decay and cumulative-energy curve show how few modes are needed to represent the flow — for the laminar cylinder, three modes carry ~99% of the fluctuation energy. POD answers *"what are the dominant energetic structures, and how low-dimensional is this flow?"*

### DMD

DMD extracts modes each with a single complex frequency and growth/decay rate, giving a genuinely predictive linear model of the dynamics. The physically meaningful shedding/buffet mode is the neutrally-stable complex-conjugate pair sitting on the unit circle at the expected Strouhal frequency.

**Correctness details that matter (learned the hard way):**
- **Stack the fields** ([U; V], plus pressure/Mach where relevant) and **subtract the temporal mean** before decomposition.
- **Retain enough modes** (~20) to preserve the complex-conjugate oscillatory pairs — a fixed-cap truncation outperforms energy-based truncation on clean data.
- **Select the dominant mode by neutral stability, not raw amplitude** — picking by amplitude alone selects transient-decay modes instead of the physical shedding mode.
- **Align the initial amplitude vector** by re-projecting the actual flow state onto the DMD modes via least squares, or the prediction phase drifts.
- **Respect Nyquist.** Sampling must give roughly ≥10 snapshots per shedding/buffet cycle; ~2 per cycle aliases the Strouhal frequency and corrupts the mode shapes even when the C_D/C_L histories look fine.
- **Drop the startup transient** (first ~30%) before decomposition; non-monotonic step counts in a CSV reveal concatenated runs from different phases that must be cleaned first.

### Outputs

Each case produces `.dat.h5` / CSV snapshot files, force-monitor histories (C_D, C_L), an FFT spectrum for the Strouhal/buffet frequency, POD singular-value and cumulative-energy plots, DMD spatial-mode contours and an eigenvalue spectrum, and a flow-field reconstruction/prediction comparison. Buffet results are structured to feed directly into the coherent-structures reduction notebook (`coherent_structures_dim_reduction.ipynb`, flowtorch/DMD methods) for the downstream analysis.

---

## Environment & Requirements

- **Ansys Fluent** (licensed) — 2024 R2 recommended
- **PyFluent** (`ansys-fluent-core`) ~0.38
- **Python 3.9+** in a Conda environment
- **Numerics/analysis:** `numpy`, `scipy` (FFT, SVD), `h5py`, `nbformat`
- **Post-processing:** `matplotlib`
- **Interactive:** `jupyterlab` / `notebook`

```bash
git clone <repo-url>
cd <repo>
conda env create -f environment.yml
conda activate <env_name>
```

> **Mesh files** (`.cas.h5`, `.msh`) are large and are **not stored in the repo**. Download them separately and place them in the relevant case folder, or update the mesh-path variable at the top of each notebook.

---

## Running a Case

Each case is a self-contained notebook, structured to mirror the same module sequence for consistency across flow regimes:

| Module | What it does |
|--------|-------------|
| 1 | Launch Fluent headless |
| 2 | Load mesh, configure boundary conditions and prism layers |
| 3 | Steady precursor → switch to transient URANS |
| 4 | Time-stepping loop with autosave of snapshots |
| 5 | Validate C_D / C_L and Strouhal (or buffet) frequency against the reference |
| 6 | Snapshot matrix → SVD/POD → DMD → eigenvalue spectrum → reconstruction/prediction |

Open the notebook for the case you want and run the cells sequentially. Snapshots are only extracted once the force history confirms a stationary limit cycle — this is the single most important gate before any modal analysis.

---

## PyFluent Notes (2024 R2 / ~0.38)

A few API patterns that repeatedly caused friction and are worth knowing:

- `boundary_conditions()` returns BC-*type* category names, not zone names — zone names must be hard-coded.
- Use `bcs.set_zone_type(zone_list=[...], new_type="...")`; wall BC strings need capitalization with spaces (`"No Slip"`, `"Stationary Wall"`).
- Indexed `flow_direction[0]`/`[1]` access (not component-named); `velocity` (not `velocity_magnitude`); `max_iter_per_time_step` (not `max_iterations_per_time_step`).
- Autosave lives at `session.settings.file.auto_save` with a plain integer `data_frequency`; `surfaces_list` must be a plain Python list.
- `"object is not active"` errors for empty BC categories (e.g. `velocity_inlet` when unused) are harmless confirmations, not failures.
- PyFluent's `two_dimensional_meshing()` workflow exports a surface mesh the solver rejects — manual meshing in Workbench/DesignModeler, or a SpaceClaim `.scdoc` + `WorkflowType="2D Meshing"`, is more reliable.
- Wrap solver residual logs in `%%capture` and clear large output cells, or notebooks exceed GitHub's ~1 MB render threshold.

---

## References

**Method**
- Schmid, P. J. (2010). Dynamic mode decomposition of numerical and experimental data. *J. Fluid Mech.*, 656, 5–28.

**Lid-driven cavity**
- Ghia, U., Ghia, K. N., & Shin, C. T. (1982). High-Re solutions for incompressible flow using the Navier–Stokes equations and a multigrid method. *J. Comput. Phys.*, 48, 387–411.

**Cylinder**
- Roshko, A. (1954). On the development of turbulent wakes from vortex streets. NACA TR-1191.
- Achenbach, E. (1971). Influence of surface roughness on the cross-flow around a circular cylinder. *J. Fluid Mech.*, 46(2), 321–335.
- Henderson, R. D. (1995). Details of the drag curve near the onset of vortex shedding. *Phys. Fluids*, 7(9), 2102–2104.
- Parnaudeau, P., Carlier, J., Heitz, D., & Lamballais, E. (2008). Experimental and numerical studies of the flow over a circular cylinder at Re = 3900. *Phys. Fluids*, 20, 085101.
- Norberg, C. (1987). Effects of Reynolds number and low-intensity free-stream turbulence on the flow around a circular cylinder. Chalmers Publication 87/2.
- NASA NPARC Alliance Verification & Validation Archive — laminar cylinder, Re = 150.

**Transonic buffet (NACA 0012)**
- McDevitt, J. B., & Okuno, A. F. (1985). Static and Dynamic Pressure Measurements on a NACA 0012 Airfoil in the Ames High Reynolds Number Facility. NASA TP-2485.
- Harris, C. D. (1981). Two-Dimensional Aerodynamic Characteristics of the NACA 0012 Airfoil. NASA TM-81927.
- Bouhadji, A., & Braza, M. (2003). Organised modes and shock–vortex interaction in unsteady transonic flows around an aerofoil. *Computers & Fluids*, 32, 1233–1260.
- Barakos, G., & Drikakis, D. (2000). Numerical simulation of transonic buffet flows using various turbulence closures. *Int. J. Heat Fluid Flow*, 21, 620–626.
- Crouch, J. D., Garbaruk, A., Magidov, D., & Travin, A. (2009). Origin of transonic buffet on aerofoils. *J. Fluid Mech.*, 628, 357–369.
- Zauner, M., De Tullio, N., & Sandham, N. D. (2019). Direct numerical simulations of transonic flow around an airfoil at moderate Reynolds numbers. *J. Fluid Mech.*, 866.

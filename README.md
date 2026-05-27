# Ansys PyFluent — CFD Automation & DMD/SVD Analysis

A fully automated, GUI-free Python framework for running CFD simulations and performing data-driven reduced-order modelling using **Ansys PyFluent**. The framework couples high-fidelity URANS simulations with **Dynamic Mode Decomposition (DMD)** and **Singular Value Decomposition (SVD)** to extract dominant coherent flow structures from time-resolved snapshots — all within a single reproducible Jupyter environment.

---

## What's Included
This repository contains an integrated Computational Fluid Dynamics (CFD) and Machine Learning (ML) framework developed entirely within a Python environment using the Ansys PyFluent API. 
The project automates the end-to-end simulation pipeline, encompassing geometry and mesh generation, transient solver execution, and the automated extraction of time-resolved flow snapshots. By coupling high-fidelity unsteady Reynolds-Averaged Navier-Stokes (URANS) simulations with Singular Value Decomposition (SVD) and Dynamic Mode Decomposition (DMD), the framework successfully builds predictive Reduced-Order Models (ROMs) capable of extracting and analyzing dominant coherent flow structures. 
The methodology is rigorously validated against established NASA experimental data for laminar vortex shedding over a 2D circular cylinder , and the repository also documents the framework's extension to turbulent flow applications over a NACA 0012 airfoil.  


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

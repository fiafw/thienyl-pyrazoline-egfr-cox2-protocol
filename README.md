# Thienyl-Pyrazoline EGFR/COX-2: Docking, MD and BAR Protocol

A step-by-step computational protocol for structure-based evaluation of small-molecule inhibitors against **EGFR tyrosine kinase** and **COX-2**, combining DFT ligand preparation, AutoDock Vina docking, GROMACS molecular dynamics (MD), and Bennett acceptance ratio (BAR) free-energy estimation.

This repository accompanies the paper below. **It contains the protocol only (parameters, scripts, and instructions). Research data, ligand structures, topologies, trajectories, and results are not included.**

> Wulan, F. F., Wahyuningsih, T. D., Astuti, E., & Prasetyo, N. (2023). Towards targeting EGFR and COX-2 inhibitors: comprehensive computational studies on the role of chlorine group in novel thienyl-pyrazoline derivative. *Journal of Biomolecular Structure and Dynamics*. https://doi.org/10.1080/07391102.2023.2252915

---

## Overview

| Stage | Purpose | Main tools |
|---|---|---|
| 1. Structure preparation | Protein and ligand setup | Discovery Studio Visualizer, GaussView, Gaussian 09W |
| 2. Docking | Predict binding poses and affinities | AutoDock Vina, AutoDock Tools |
| 3. MD simulation | Assess complex stability (180 ns) | GROMACS 2022.4, ATB, GROMOS 54A7 |
| 4. Trajectory analysis | RMSD, RMSF, Rg, SASA, H-bonds, FEL | GROMACS tools, VMD, XMGrace |
| 5. Free energy | Binding free energy by BAR | GROMACS (alchemical), Bennett acceptance ratio |
| 6. Drug-likeness / ADMET | Property prediction | SwissADME, pkCSM |

Targets used in the study (public PDB entries):

- **EGFR tyrosine kinase domain**: PDB `4HJO` (co-crystal ligand: erlotinib)
- **COX-2**: PDB `3LN1` (co-crystal ligand: celecoxib)

---

## Repository structure

```
thienyl-pyrazoline-egfr-cox2-protocol/
├── README.md
├── LICENSE
├── environment.yml
├── 01_structures/
│   ├── download_pdb.sh
│   └── prepare_ligands.md
├── 02_docking/
│   ├── vina_config_template.txt
│   ├── redocking_validation.md
│   └── run_docking.sh
├── 03_md/
│   ├── mdp/                 (em.mdp, nvt.mdp, npt.mdp, md.mdp)
│   ├── topologies/README.md
│   └── run_md.sh
├── 04_analysis/
│   ├── rmsd_rmsf.sh
│   ├── rog_sasa_hbond.sh
│   ├── fel.md
│   └── plots.py
├── 05_bar_free_energy/
│   ├── mdp/                 (lambda templates)
│   └── run_bar.sh
└── example/                 (optional: public demo system)
```

---

## Requirements

- GROMACS 2022.4
- AutoDock Vina and AutoDock Tools (MGLTools)
- Gaussian 09W and GaussView 5.0 (commercial; used for ligand geometry optimization)
- Discovery Studio Visualizer 2017
- VMD, XMGrace
- Python 3 (for plotting; see `environment.yml`)
- Web resources: [ATB server](https://atb.uq.edu.au) (ligand topology), SwissADME, pkCSM, PDBsum

---

## Protocol

### 1. Structure preparation

**Protein**
1. Download `4HJO` and `3LN1` from the RCSB PDB (`01_structures/download_pdb.sh`).
2. Add hydrogens and Kollman charges using Discovery Studio Visualizer 2017.

**Ligands**
1. Build 3D structures in GaussView 5.0.
2. Optimize to the lowest-energy conformer in Gaussian 09W at the **DFT/B3LYP/6-31G** level.
3. Convert the output (`.log`) to **PDBQT** for docking.

See `01_structures/prepare_ligands.md`.

### 2. Molecular docking (AutoDock Vina)

| Parameter | Value |
|---|---|
| Grid box | Cubic, 15 Å, spacing 1.000 Å, centred on the native ligand |
| Grid centre, EGFR (4HJO) | x = 24.407, y = 9.151, z = −0.636 |
| Grid centre, COX-2 (3LN1) | x = 30.597, y = −22.559, z = −15.758 |
| Exhaustiveness | 16 |
| Poses generated | 20 (lowest-binding-affinity pose selected) |

**Validation:** redock the native ligand (erlotinib for EGFR, celecoxib for COX-2) into the active site. A redocking RMSD **< 2 Å** was taken as acceptable. Apply the same box (same size and position) when docking new ligands.

Interactions are visualized in Discovery Studio Visualizer 2017. See `02_docking/`.

### 3. Molecular dynamics (GROMACS 2022.4)

| Parameter | Setting |
|---|---|
| Ligand topology | Automated Topology Builder (ATB) |
| Protein force field | GROMOS 54A7 |
| Water model | SPC/E |
| Box | Cubic, 100 × 100 × 100 Å³, periodic boundary conditions in all directions |
| Neutralization | Na⁺ / Cl⁻ ions |
| Electrostatics | Particle mesh Ewald (PME), 1.2 nm cut-off |
| Energy minimization | 1,000 steps, steepest descent |
| NVT equilibration | 500 ps, 298.15 K, velocity rescaling (V-rescale) thermostat |
| NPT equilibration | 1 ns, 1 bar, Parrinello–Rahman pressure coupling |
| Production | **180 ns** |

Systems simulated per target: apoprotein and protein–ligand complexes of the selected ligands.

Template `.mdp` files and the command sequence are in `03_md/`. Ligand topology generation is described in `03_md/topologies/README.md`.

### 4. Trajectory analysis

Analyses from the production trajectories, using GROMACS tools:

- RMSD (protein backbone and ligand)
- RMSF (per residue)
- Radius of gyration (Rg)
- Solvent-accessible surface area (SASA)
- Number of hydrogen bonds over time
- Free energy landscape (FEL), built from RMSD and Rg
- Secondary structure: Ramachandran plot (PDBsum) and DSSP

Visualization: VMD (structures), XMGrace (plots). 2D FEL contour maps were produced with Wolfram Mathematica; a Python alternative is provided in `04_analysis/plots.py`.

### 5. Binding free energy (BAR)

Binding free energy was estimated with the Bennett acceptance ratio:

```
ΔG_binding = ΔG_complexation − ΔG_solvation
```

- ΔG_solvation: ligand in water
- ΔG_complexation: ligand coupled to the protein–water system
- λ windows: **10** for Coulombic and **20** for non-Coulombic coupling, with Δλ = 0.05
- Sampling: **10 ns** per λ window

Template λ `.mdp` files and scripts are in `05_bar_free_energy/`.

### 6. Drug-likeness and ADMET

- Drug-likeness filters (Lipinski, Ghose, Veber, Egan) and physicochemical descriptors: [SwissADME](http://www.swissadme.ch)
- Absorption, distribution, metabolism, excretion, toxicity: [pkCSM](https://biosig.lab.uq.edu.au/pkcsm/)

---

## Notes and suggested extensions

This protocol reproduces the published workflow. For production-quality studies, consider:

- **Replicas:** run independent replicas (three or more) per system to assess reproducibility.
- **BAR set-up:** document any restraints used for the complexation leg and check convergence per λ window.
- **Force fields:** compare with AMBER (ff14SB/ff19SB with GAFF2/OpenFF) or CHARMM/CGenFF, which are widely used in drug discovery.
- **Experimental validation:** correlate computed values with measured activities (e.g., IC50).

---

## Data availability

No research data are included in this repository. The protein structures are public (RCSB PDB: 4HJO, 3LN1). For questions about the study, please refer to the paper or contact the corresponding author listed in the publication.

---

## How to cite

If you use this protocol, please cite the paper:

```
Wulan, F. F., Wahyuningsih, T. D., Astuti, E., & Prasetyo, N. (2023).
Towards targeting EGFR and COX-2 inhibitors: comprehensive computational studies
on the role of chlorine group in novel thienyl-pyrazoline derivative.
Journal of Biomolecular Structure and Dynamics.
https://doi.org/10.1080/07391102.2023.2252915
```

## License

Code and scripts: MIT License (see `LICENSE`). Documentation: CC BY 4.0.

## Contact

Fia Fathiana Wulan — [fathianawulan@gmail.com] · [https://linkedin.com/in/fathianawulan] · [github.com/fiafw]

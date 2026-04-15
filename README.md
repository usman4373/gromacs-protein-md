# Protein Molecular Dynamics Simulation with GROMACS

- This guide provides a step‑by‑step workflow for setting up and running a molecular dynamics (MD) simulation of a protein using GROMACS.
- The steps cover structure preparation, system building, solvation, ion addition, energy minimization, equilibration (NVT and NPT), production and post-processing of MD run.

## Prerequisites

- [GROMACS](https://www.gromacs.org/) installed (version 2020 or later recommended)
- A protein structure file `protein.pdb`
- MDP parameter files (`ions.mdp`, `minim.mdp`, `nvt.mdp`, `npt.mdp`, `md.mdp`) - available from [standard GROMACS tutorials](http://www.mdtutorials.com/gmx/) or created manually

---

## Step 1: Process the PDB file with `pdb2gmx`

Generate the topology, position restraint file, and a processed structure file.

```bash
gmx pdb2gmx -f protein.pdb -o protein_processed.gro -water tip3p
```

> **Note:** When running `pdb2gmx` you will be prompted to choose a force field (e.g., CHARMM, Amber, OPLS). Select the one appropriate for your system.

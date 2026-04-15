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

**Outputs:**

- topol.top – system topology
- posre.itp – position restraints for the protein
- protein_processed.gro – processed coordinate file

## Step 2: Build the system (box + solvation)

#### 2.1 Define the simulation box
- Center the protein and set a distance of 1.0 nm from the box edge.

```bash
gmx editconf -f protein_processed.gro -o protein_newbox.gro -c -d 1.0 -bt cubic
```

| Option | Meaning |
|--------|--------|
| `-c`   | Center the protein in the box |
| `-d`   | Minimum distance (nm) from protein to box edge |
| `-bt`  | Box type (cubic, triclinic, etc.) |

#### 2.2 Solvate the system
- Fill the box with water molecules (using the SPC216 water model).

```bash
gmx solvate -cp protein_newbox.gro -cs spc216.gro -o protein_solv.gro -p topol.top
```













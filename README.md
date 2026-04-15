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

### 2.1 Define the simulation box
- Center the protein and set a distance of 1.0 nm from the box edge.

```bash
gmx editconf -f protein_processed.gro -o protein_newbox.gro -c -d 1.0 -bt cubic
```

| Option | Meaning |
|--------|--------|
| `-c`   | Center the protein in the box |
| `-d`   | Minimum distance (nm) from protein to box edge |
| `-bt`  | Box type (cubic, triclinic, etc.) |

### 2.2 Solvate the system
- Fill the box with water molecules (using the SPC216 water model).

```bash
gmx solvate -cp protein_newbox.gro -cs spc216.gro -o protein_solv.gro -p topol.top
```

| Option |	Meaning |
|--------|----------|
| `-cp`  | Protein configuration file |
| `-cs`  | Solvent configuration file |

**Output:** protein_solv.gro (solvated system) and updated topol.top.

## Step 3: Add ions to neutralize the system
- First, assemble a `.tpr` file using an `ions.mdp` parameter file.

```bash
gmx grompp -f ions.mdp -c protein_solv.gro -p topol.top -o ions.tpr
```
- Now add ions, replacing water molecules. When prompted, select the SOL group (typically group number 13) – this ensures ions are placed only in the solvent, not inside the protein.

```bash
gmx genion -s ions.tpr -o protein_solv_ions.gro -p topol.top -pname NA -nname CL -neutral
```

| Option   | Meaning |
|----------|--------|
| `-s`     | Input `.tpr` structure file |
| `-p`     | Update topology file |
| `-pname` | Name of positive ion (e.g., NA) |
| `-nname` | Name of negative ion (e.g., CL) |
| `-neutral` | Add enough ions to neutralize the system |

**Output:** `protein_solv_ions.gro` (final system with ions) and updated topol.top.

## Step 4: Energy minimization
- Energy minimization removes bad contacts and relaxes the system.

### 4.1 Prepare the minimization input

```bash
gmx grompp -f minim.mdp -c protein_solv_ions.gro -p topol.top -o em.tpr
```

### 4.2 Run the minimization

```bash
gmx mdrun -v -s em.tpr -deffnm em
```

| Option   | Meaning |
|----------|--------|
| `-v`     | Verbose output |
| `-deffnm`| Base name for output files (e.g., em.gro, em.edr, etc.) |

**Outputs:** `em.gro`, `em.edr`, `em.log`, `em.trr`

### 4.3 Verify minimization success
- Potential energy should be negative and on the order of `105105–106106 kJ/mol`.
- Maximum force should be no greater than `1000 kJ mol⁻¹ nm⁻¹` (check `em.log`).
- Analyze the potential energy over minimization steps:

```bash
gmx energy -f em.edr -o potential.xvg
```
> At the prompt, type `10 0` to select potential energy (term 10) and exit.

## Step 5: Equilibration
- Equilibration stabilizes temperature (NVT) and then pressure (NPT).

### 5.1 NVT equilibration (constant temperature)

```bash
gmx grompp -f nvt.mdp -c em.gro -r em.gro -p topol.top -o nvt.tpr
gmx mdrun -s nvt.tpr -deffnm nvt
```

Check temperature progression:

```bash
gmx energy -f nvt.edr -o temperature.xvg
```
> At the prompt, type `16 0` to select the system temperature.

### 5.2 NPT equilibration (constant pressure)

```bash
gmx grompp -f npt.mdp -c nvt.gro -r nvt.gro -t nvt.cpt -p topol.top -o npt.tpr
gmx mdrun -v -s npt.tpr -deffnm npt
```

Check pressure and density:

```bash
gmx energy -f npt.edr -o pressure.xvg   # type 18 0
gmx energy -f npt.edr -o density.xvg    # type 24 0
```

## Step 6: Production MD run
- Run the final MD simulation using the `md.mdp` parameter file.

```bash
gmx grompp -f md.mdp -c npt.gro -t npt.cpt -p topol.top -o md_10ns.tpr
```

Execute the simulation (GPU acceleration shown; adapt as needed):

```bash
gmx mdrun -v -s md_10ns.tpr -deffnm md_10ns -nb gpu
```

If you don't have a GPU, use CPU threads (e.g., 12 threads):

```bash
gmx mdrun -v -s md_10ns.tpr -deffnm md_10ns -nt 12
```

**Outputs:** trajectory (`md_0_1.xtc`), energy file (`md_0_1.edr`), and log file.

## Step 7: Post‑processing and analysis
<p align="justify">After the production MD run, the trajectory may contain artifacts due to periodic boundary conditions (PBC). Molecules can diffuse across box boundaries, making them appear “broken” or “jumping”. The first step is to correct this by centering the protein and removing PBC jumps.</p>

### 7.1 Remove periodic boundary effects

```bash
gmx trjconv -s md_10ns.tpr -f md_10ns.xtc -o md_10ns_noPBC.xtc -pbc mol -center
```

During execution you will be prompted twice:

- Select group for centering – choose the protein (typically group 1).
- Select group for output – choose the system (group 0).

This command wraps all molecules into the unit cell (`-pbc mol`) and centers the protein in the box (`-center`). The output trajectory `md_10ns_noPBC.xtc` is ready for analysis.

> Tip: For large trajectories, you can use `-ur` compact or `-pbc nojump` instead of `-pbc mol` depending on your needs. See the [GROMACS trjconv documentation](https://manual.gromacs.org/current/onlinehelp/gmx-trjconv.html) for details.
 
### 7.2 Common structural analyses
All analyses below use the corrected trajectory (`md_10ns_noPBC.xtc`) and the run input file (`md_10ns.tpr`).

### 7.2.1 RMSD – Root Mean Square Deviation

Measures how much the protein structure deviates from a reference (usually the starting structure) over time.

```bash
gmx rms -s md_10ns.tpr -f md_10ns_noPBC.xtc -o RMSD.xvg -tu ns
```
At the prompts:
- For least squares fit – choose 4 (Backbone)
- For RMSD calculation – choose 4 (Backbone)

**Output:** `RMSD.xvg` (RMSD in `nm` vs. time in `ns`)

### 7.2.2 Radius of gyration (rGyr)
Indicates the compactness of the protein.

```bash
gmx gyrate -s md_10ns.tpr -f md_10ns_noPBC.xtc -o gyrate.xvg
```

At prompt:
- Select group – choose 1 (Protein)

**Output:** `gyrate.xvg` (radius of gyration in `nm`)

### 7.2.3 RMSF – Root Mean Square Fluctuation
Per‑residue flexibility (requires an index file with residue numbers).

```bash
gmx rmsf -s md_10ns.tpr -f md_10ns_noPBC.xtc -o rmsf.xvg -res
```

At prompt:
- Select group for fitting – usually 4 (Backbone)
- Select group for RMSF – 4 (Backbone) or 1 (Protein) – use `-res` to get per‑residue output.

**Output:** `rmsf.xvg` (RMSF in `nm` per residue)

### 7.2.4 SASA – Solvent Accessible Surface Area
Measures the surface area exposed to solvent.

```bash
gmx sasa -s md_10ns.tpr -f md_10ns_noPBC.xtc -o sasa.xvg -or resarea.xvg
```
At prompt:
- Select group – 1 (Protein)

**Outputs:**
- `sasa.xvg` – total SASA over time (`nm²`)
- `resarea.xvg` – residue‑wise SASA

### 7.3 Automated plotting and advanced analysis with Dynamics‑Visualizer

Instead of manually plotting each `.xvg` file, you can use the **[Dynamics‑Visualizer](https://github.com/usman4373/Dynamics-Visualizer)** – a Streamlit‑based app that automates post‑MD analysis and visualization for GROMACS trajectories.

#### What it does

- **Standard analyses** – Reads `.xvg` files (RMSD, RMSF, SASA, radius of gyration) and produces publication‑ready time‑series plots.
- **Principal Component Analysis (PCA)** – Computes and plots PCA for protein‑only or protein‑ligand systems with time‑based colour bars.
- **Dynamic Cross‑Correlation Matrix (DCCM)** – Generates heatmaps showing correlated motions between residues (and ligand, if present).
- **Trajectory visualizer** – Extracts frames from your `.xtc` trajectory, renders PNGs using PyMOL, and assembles MP4 videos of the simulation.

#### How to use it

1. **Install** the tool (see [its README](https://github.com/usman4373/Dynamics-Visualizer) for conda/pip instructions, including PyMOL).
2. **Prepare your data** – Use the **centered trajectory** (`md_10ns_noPBC.xtc`) and the run input file (`md_10ns.tpr`) from Step 7.1.
3. **Run the app**:
   ```bash
   streamlit run app.py

## Notes
- The `.mdp` files (`ions.mdp`, `minim.mdp`, `nvt.mdp`, `npt.mdp`, `md.mdp`) contain simulation parameters. They must match the force field and water model you selected. Obtain them from a trusted tutorial or adjust accordingly.
- Always verify group indices when using gmx genion or other interactive modules (gmx energy, gmx rms). Use `gmx` help groups to list groups in a `.tpr` file.
- For large systems, consider using `-ntmpi` and `-nt` to optimise parallel performance.












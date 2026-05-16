# brinkmanPimpleFoam — Tutorial Case

This tutorial case simulates the steady state flow over a reconfigured vegetation canopy, aiming to extract the shielding that occurs on the internal articulations of the plant.

Any questions about specific use of this case, or info into the research, please email me at maxvyze@outlook.com


> **Note:** This is a computationally heavy case, simulating close to 22 million cells for upwards of 2000 iterations for convergence. The specs of the PC used are listed below, along with average convergence times, so please simulate with the knowledge that it will likely not be done the same day you start it.
>
> | Component | Details |
>|---|---|
>| CPU | Ryzen 9 9900x|
>| CPU Cores (physical / logical) | 12/24 |
>| RAM | 65GB DDR5 CL40 |
>| Operating System | Ubuntu 18.04|
>| foam-extend Version | 4.1 |
>| ||
>| Total Cell Count | ~22 million |
>| Parallel Cores Used | 12 |
>| Iterations to Convergence | ~2000 |
>| Average Time per Iteration | 40-60 s|
>| Total Wall-Clock Time | 100,000 s (25-30 hours)|
>| Peak Memory Usage | ~50GB|

---

## Overview

This is a case I have created adjacent to my dissertation research setup, labelled 'A Volume-Penalised Immersed Boundary Framework for Modelling Flow-Induced Reconfiguration of Aquatic Vegetation'. Here, we are modelling turbulent open-channel flow past a single reconfigured, branched plant.

The case uses Rufus Dickinson's EABM.jl Julia structural library to generate the plant, which has been imported into this setup using the cellSetDict.cyl file in the 'save/' folder. This aims to use the Immersed Boundary Method (IBM), penalised implicitly, through a Brinkman penalty in the momentum equation of 1e6 within the plant region, effectively treating the obstacle as a solid object.



---

## Prerequisites

Before running this case you will need:

- **foam-extend 4.1** sourced and compiled. 
> **Note**: There is no testing whether this works on OpenFOAM's main branch, or even other versions of foam-extend (though the latter should be fine). If you try this on another version and have success, please let me know, so I can update this!

- **`brinkmanPimpleFoam`** - see the brinkmanPimpleFoam [documentation](https://github.com/mvyze10/brinkmanPimpleFoam) for download.

- **MPI** — the case is configured for 12 cores; to change see [Modifying the Case](#modifying-the-case)

> **Note:** once the case has been downloaded, please input:
>```bash
>chmod +x All*
>```
>This unlocks the AllRun-adjacent files for use.

---

## Case Structure

```
brinkmanPimpleFoamTutorial/
├── 0_org/                       # Initial field conditions (copied to 0/ at run time)
│   ├── U                        # Velocity — 1.0 m/s inlet
│   ├── p                        # Kinematic pressure
│   ├── k                        # Turbulent kinetic energy (k-omega SST)
│   ├── omega                    # Specific dissipation rate
│   ├── nut                      # Turbulent viscosity
│   └── plantMask                # Porous zone indicator — set to 0 here, stamped by setFields
├── constant/
│   ├── turbulenceProperties     # Selection of RAS modelling
│   ├── RASProperties            # Turbulence model selection (kOmegaSST)
│   └── transportProperties      # Kinematic viscosity (water, 1e-6 m²/s)
├── save/                        # Template dictionaries copied into system/ at run time
│   ├── blockMeshDict            # Base mesh definition
│   ├── cellSetDict.wide         # Wide refinement region
│   ├── cellSetDict.medium       # Medium refinement region for wake
│   ├── cellSetDict.tight        # Tight refinement region close to plant
│   ├── cellSetDict.cyl          # Plant geometry
│   ├── refineMeshDict.wide      # Refinement settings for wide pass
│   ├── refineMeshDict.medium    # Refinement settings for medium pass
│   └── refineMeshDict.tight     # Refinement settings for tight pass
├── system/
│   ├── controlDict              # Run control, time stepping, function objects
│   ├── fvSchemes                # Discretisation schemes
│   ├── fvSolution               # Solver settings and PIMPLE controls
│   ├── decomposeParDict         # Parallel decomposition (12 cores)
│   └── setFieldsDict            # Stamps plantMask = 1 onto the vegetationZone
├── AllRun                       # Full pipeline: clean + mesh + solve
├── AllMesh                      # Meshing only
├── AllSolve                     # Solve only
├── AllClean                     # Clean case files only
├── AllCleanSolve	         # Clean case files generated after meshing only (anything in AllSolve)
└── Images/                      # Folder for example visualisations, such as the ones used in this file
```

---

## Plant Geometry

- The plant is built from chained `cylinderToCell` and `sphereToCell` segments in `save/cellSetDict.cyl` — each pair defines one stem segment, with the sphere capping the end to avoid gaps at joints.

    > **Note:** This is a single plant example I have used, that does not reference any specific plant or model. I have tried to create an equiradial plant, so the drag force is affected symmetrically. Documentation of the structural library I used to generate the plant is [here](https://github.com/Rufnacous/EABM.jl/tree/main).
- These are assembled into a cell set called `vegetationZone` using `cellSet`, then converted to a cell zone with `setsToZones -noFlipMap`
- `setFields` then finds all cells in `vegetationZone` and sets `plantMask = 1` on them
> **Note:** The solver smooths the mask boundary over two averaging passes at startup. This is purely to avoid numerical instability encountered when adding a penalty of a large magnitude (anything over 10e4 would likely require this), however this can be changed if needed, with methods for this located within the solver's README file.

---

## Mesh

- Domain: x = [-1, 3] m, y = [0, 0.5] m, z = [-1.25, 1.25] m. The plant sits at the origin
- Base mesh: 80 × 10 × 50 cells from `blockMesh`
- Three refinement passes are applied in sequence before the plant zone is created:
  - Wide (×1) — large region that captures the far wake
  - Medium (×2) — tighter region, resolves the approach flow and near wake
  - Tight (×2) — close to the plant
- Final cell count: ~22 million cells

---

## Running the Case

### Quick start

There are two ways to run this setup:

- 1. Run the entire simulation setup and solve in one go, using:

    ```bash
    ./AllRun
    ```

    > **Warning:** `AllRun` cleans the entire case directory first, removing any existing mesh and solution. Use `./AllCleanSolve` then `./AllSolve` if you only want to re-run the solver on an existing mesh.


- 2. Running each part separately (recommended) to both allow easier debugging, and to get a better understanding of each section.

    For this, run each of `./AllMesh`, `./AllSolve` separately, with `./AllClean` and `./AllCleanSolve` reserved for wiping past simulations.




    *All logs are written to `logs/` — check these if anything fails.*

---

## Expected Output

All residuals dropping to ~1e-5 is an good indication of convergence for this case. It will take at least a day on hardware similar to mine. Using:
```bash
tail -f logs/log.mpirun
```
Will allow you to see a continuous print out of the log in the terminal. Once the U and p initial residuals drop to below 1e-4, you can get a ~90% accurate simulation. Continuing to the full 1e-5 will guarantee convergence.

- Residuals for `p`, `k`, and `omega` are written every time step to `postProcessing/residuals/`
- Time-averaged fields (`UMean`, `pMean`, `UPrime2Mean`) begin accumulating from `t = 10 s` (set in `controlDict` under `fieldAverage`)
- Probe data at the locations defined in `controlDict` is written to `postProcessing/probes/`
 

![alt text](images/ySlice_1.0U_x.png)

![alt text](images/zSlice_1.0U_UMean.png)

---

## Visualising in ParaView

Open `brinkmanPimpleFoam.foam` in ParaView to load the case. Suggested starting points:

- **Slice** — slicing y-normal at y = 0.15 will show the above photo (using blue-red-extended colour scheme)
- **`plantMask`** — tick the box labelled 'read zones' underneath the field values in the properties tab and then select the 'Extract Block' filter with the vegetationZone selected, to show the plant structure.
- **`UMean`** — time-averaged streamwise velocity, available once the simulation has run past `t = 10 s`
- **Streamlines** seeded upstream give a clear picture of how flow is diverted by the canopy

---

## Modifying the Case

| What to change | Where |
|---|---|
| Inlet flow speed | `0_org/U` — edit the `fixedValue` on the `in` patch |
| Turbulence intensity (k, omega) | `0_org/k` and `0_org/omega` |
| Penalty magnitude | `UEqn.H` in the solver source — change `1e6`, then recompile with `wmake` |
| Number of parallel cores | `system/decomposeParDict` and the `mpirun -np` line in `AllSolve` |
| Plant geometry | `save/cellSetDict.cyl` — change to any cellSetDict file theoretically |

---

## Notes
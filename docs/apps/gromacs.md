---
tags:
  - Free
---

# Gromacs

Gromacs is a very efficient engine to perform molecular dynamics
simulations and energy minimizations particularly for proteins. However,
it can also be used to model polymers, membranes and e.g. coarse grained
systems. It also comes with plenty of analysis scripts.

[TOC]

## Available

=== "Puhti"
    | Version | Available modules | Notes |
    |:-------:|:------------------|:-----:|
    |2020.5   |`gromacs/2020.5`
    |2020.7   |`gromacs/2020.7`
    |2021.4   |`gromacs/2021.4-plumed`|Module with Plumed available
    |2021.5   |`gromacs/2021.5`<br>`gromacs/2021.5-cuda`|GPU-enabled module available
    |2021.6   |`gromacs/2021.6`
    |2022.2   |`gromacs/2022.2`<br>`gromacs/2022.2-cuda`|GPU-enabled module available
    |2022.3   |`gromacs/2022.3`<br>`gromacs/2022.3-cuda`|GPU-enabled module available
    |2022.4   |`gromacs/2022.4`<br>`gromacs/2022.4-cuda`|GPU-enabled module available

=== "Mahti"
    | Version | Available modules | Notes |
    |:-------:|:------------------|:-----:|
    |2020.4   |`gromacs/2020.4-plumed`|Module with Plumed available
    |2020.5   |`gromacs/2020.5`
    |2021.3   |`gromacs/2021.3`
    |2021.4   |`gromacs/2021.4-plumed`|Module with Plumed available
    |2021.5   |`gromacs/2021.5`
    |2022     |`gromacs/2022`<br>`gromacs/2022-cp2k`|Module with CP2K available for QM/MM
    |2022.1   |`gromacs/2022.1`<br>`gromacs/2022.1-cp2k`|Module linked with CP2K available for QM/MM
    |2022.2   |`gromacs/2022.2`<br>`gromacs/2022.2-cuda`|GPU-enabled module available
    |2022.3   |`gromacs/2022.3`<br>`gromacs/2022.3-cuda`|GPU-enabled module available
    |2022.4   |`gromacs/2022.4`<br>`gromacs/2022.4-cuda`|GPU-enabled module available

=== "LUMI"
    | Version | Available modules | Notes |
    |:-------:|:------------------|:-----:|
    |2022.5   |`gromacs/2022.5`<br>`gromacs/2022.5-plumed`|Module with Plumed available
    |2023     |`gromacs/2023-dev-rocm`|Unofficial GPU-enabled fork developed by AMD

- Puhti and Mahti also have `gromacs-env/<year>` modules for loading the recommended latest minor version from each year (replace `<year>` accordingly).
- To access modules on LUMI, first load the CSC module tree into use with `module use /appl/local/csc/modulefiles`
- If you want to use command-line [Plumed tools](plumed.md), load the Plumed module.

!!! info
    We only provide the parallel version `gmx_mpi`, but it can
    be used for grompp, editconf etc. similarly to the serial version.
    Instead of `gmx grompp ...`, give `gmx_mpi grompp`.

## License

Gromacs is a free software available under LGPL, version 2.1.

## Usage

Initialize recommended version of Gromacs on Puhti like this:

```bash
module purge
module load gromacs-env
```

Use `module spider` to locate other versions. To load these modules, you
need to first load its dependencies, which are shown with
`module spider gromacs/<version>`.

<!-- The module will set `OMP_NUM_THREADS=1`
as otherwise mdrun will spawn threads for cores it _thinks_ are free. -->

### Notes about performance

It is important to set up the simulations properly to use resources efficiently.
The most important aspects to consider are:

- If you run in parallel, make a scaling test for each system - don't use more cores
  than is efficient. Scaling depends on many aspects of your system and used algorithms,
  not just size.
- Use a recent version - there has been significant speedup over the years
- Minimize unnecessary disk I/O - never run batch jobs with -v (the verbose flag)
  for mdrun
- For large jobs, use full nodes (multiples of 40 cores on Puhti or multiples
  of 128 cores on Mahti), see example below.
- Performance with GPU depends on what you offload and the optimium depends on many factors.
  Please consult the [excellent ENCCS online materials](https://enccs.github.io/gromacs-gpu-performance/)

For a more complete description, consult the [mdrun performance checklist] on the
Gromacs page.

We recommend using the latest versions as they have most bugs fixed and
tend to be faster. If you switch the major version, check that the
results are comparable.

A scaling test with a very large system (1M+ particles) may take a while to
load balance optimally. Rather than running very long scaling tests in advance,
it is better to increase the number of nodes in your production simulation **IF**
you see better performance than in the scaling test at the scaling limit.

### Example parallel batch script for Puhti

```bash
#!/bin/bash
#SBATCH --time=00:15:00
#SBATCH --partition=large
#SBATCH --ntasks-per-node=40
#SBATCH --nodes=2
#SBATCH --account=<project>
##SBATCH --mail-type=END #uncomment to get mail

# this script runs a 80 core (2 full nodes) gromacs job, requesting 15 minutes time

module purge
module load gromacs-env
export OMP_NUM_THREADS=1

srun gmx_mpi mdrun -s topol -maxh 0.2 -dlb yes
```

!!! Note
    To avoid multi node parallel jobs to spread over more nodes
    than necessary, don't use the `--ntasks` flag, but specify `--nodes` and
    `--ntasks-per-node=40` to get full nodes. This minimizes communication
    overhead and fragmentation of node reservations. Don't use the large
    partition for jobs with less than 40 cores.

### Example serial batch script for Puhti

```bash
#!/bin/bash
#SBATCH --time=00:15:00
#SBATCH --partition=small
#SBATCH --ntasks=1
#SBATCH --account=<project>
##SBATCH --mail-type=END #uncomment to get mail

# this script runs a 1 core gromacs job, requesting 15 minutes time

module purge
module load gromacs-env
export OMP_NUM_THREADS=1

srun gmx_mpi mdrun -s topol -maxh 0.2
```

### Example GPU batch script for Puhti

```bash
#!/bin/bash
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=10
#SBATCH --gres=gpu:v100:1
#SBATCH --time=00:10:00
#SBATCH --partition=gpu
#SBATCH --account=<project>
##SBATCH --mail-type=END #uncomment to get mail

module purge
module load gromacs-env/2022-gpu

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun gmx_mpi mdrun -s verlet -dlb yes

# additional flags, like these, may be useful - test!
# srun gmx_mpi mdrun -pin on -pme gpu -pmefft gpu -nb gpu -bonded gpu -update gpu -nstlist 200 -s verlet -dlb yes
```

!!! Note
    Please make sure that using one GPU (and upto 10 cores) is
    faster than using one full node of CPU cores according
    to the [Usage policy](../../computing/usage-policy) page under
    Computing. Otherwise, don't use GPUs.

Submit the script with `sbatch script_name.sh`

### Example mpi-only parallel batch script for Mahti

```bash
#!/bin/bash
#SBATCH --time=00:15:00
#SBATCH --partition=medium
#SBATCH --ntasks-per-node=128
#SBATCH --nodes=2
#SBATCH --account=<project>
##SBATCH --mail-type=END #uncomment to get mail

# this script runs a 256 core (2 full nodes, no hyperthreading) gromacs job,
# requesting 15 minutes time

module purge
module load gromacs-env

export OMP_NUM_THREADS=1

srun gmx_mpi mdrun -s topol -maxh 0.2 -dlb yes
```

### Example mixed parallel batch script for Mahti

```bash
#!/bin/bash
#SBATCH --time=00:15:00
#SBATCH --partition=medium
#SBATCH --ntasks-per-node=64
#SBATCH --cpus-per-task=2
#SBATCH --nodes=2
#SBATCH --account=<project>
##SBATCH --mail-type=END #uncomment to get mail

# this script runs a 256 core (2 full nodes, no hyperthreading) gromacs job,
# requesting 15 minutes time and 64 tasks per node, each with 2 OpenMP threads

module purge
module load gromacs-env

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun gmx_mpi mdrun -s topol -maxh 0.2 -dlb yes
```

### Example GPU batch script for LUMI

!!! info "Note"
    Gromacs multi-GPU simulations benefit greatly from GPU-aware MPI. However,
    as Gromacs might not recognize that the underlying MPI is GPU-aware, one
    needs to force it with `export GMX_FORCE_CUDA_AWARE_MPI=true` (see below).

```bash
#!/bin/bash
#SBATCH --partition=standard-g
#SBATCH --account=<project>
#SBATCH --time=01:00:00
#SBATCH --nodes=1
#SBATCH --gpus-per-node=8     # 8 GCDs per node on LUMI (interpreted as separate GPUs by Slurm)
#SBATCH --ntasks-per-node=8
#SBATCH --cpus-per-task=7     # Only 63 cores per GPU node available on LUMI for computation

module use /appl/local/csc/modulefiles
module load gromacs/2023-dev-rocm

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export MPICH_GPU_SUPPORT_ENABLED=1

export GMX_FORCE_UPDATE_DEFAULT_GPU=true
export GMX_ENABLE_DIRECT_GPU_COMM=true
export GMX_FORCE_CUDA_AWARE_MPI=true

srun gmx_mpi mdrun -s topol -pin on -nb gpu -bonded gpu -pme gpu -npme 1 -gpu_id 01234567
```

Below is an example of the GPU performance using the STMV benchmark. Note that running
Gromacs with GPU-aware MPI is currently possible only on LUMI. Please consider also the
size of your system when using GPUs – the STMV benchmark contains more than 1 million
atoms. Smaller systems are typically best run using just a single GPU. If possible,
use [`multidir` ensemble simulations](#high-throughput-computing-with-gromacs) for
accelerated sampling.

![Gromacs scaling on GPUs on Mahti and LUMI](../img/gmx-gpu.png 'Gromacs scaling on GPUs on Mahti and LUMI')

### High-throughput computing with Gromacs

Gromacs comes with a built-in `multidir` functionality, which allows users to run
multiple concurrent simulations within one Slurm allocation. This is an excellent
option for high-throughput use cases, where the aim is to run several similar, but
independent, jobs. Notably, multiple calls of `sbatch` or `srun` are not needed,
which decreases the load on the batch queue system. Please consider this option if
you're running high-throughput workflows or jobs such as replica exchange, umbrella
sampling or adaptive weight histogram (AWH) free energy simulations using Gromacs.

An example `multidir.sh` batch script for running a `multidir` Gromacs job is
provided below. This example adapts the production part of the
[lysozyme tutorial](http://www.mdtutorials.com/gmx/lysozyme/) by considering 8
similar copies of the system that have been equilibrated with different velocity
initializations. Inputs corresponding to each copy are named identically `md_0_1.tpr`
and placed in subdirectories `run*` as illustrated below by the output of the `tree`
command.

```console
$ tree
.
├── multidir.sh
├── run1
│   └── md_0_1.tpr
├── run2
│   └── md_0_1.tpr
├── run3
│   └── md_0_1.tpr
├── run4
│   └── md_0_1.tpr
├── run5
│   └── md_0_1.tpr
├── run6
│   └── md_0_1.tpr
├── run7
│   └── md_0_1.tpr
└── run8
    └── md_0_1.tpr
```

```bash
#!/bin/bash
#SBATCH --time=00:30:00
#SBATCH --partition=medium
#SBATCH --ntasks-per-node=128
#SBATCH --nodes=1
#SBATCH --account=<project>

# this script runs a 128 core gromacs multidir job (8 simulations, 16 cores per simulation)

module purge
module load gromacs-env

export OMP_NUM_THREADS=1

srun gmx_mpi mdrun -multidir run* -s md_0_1.tpr -dlb yes
```

By issuing `sbatch multidir.sh` in the parent directory, all simulations are run concurrently
using one full Mahti node without hyperthreading so that each system is allocated 16 cores.
As the systems were initialized with different velocities, we obtain 8 distinct trajectories
and an improved sampling of the phase space (see RMSD analysis below). This is a great option
for enhanced sampling when your system does not scale beyond a certain core count.

![Root-mean-squared-deviations of the simulated replicas](../img/multidir-rmsd.svg 'Root-mean-squared-deviations of the simulated replicas')

For further details on running Gromacs multi-simulations, see the [official Gromacs documentation].

### Visualizing trajectories and graphs

In addition to the `view` tool of Gromacs (not available at CSC),
trajectory files can be visualized with the following programs:

- [VMD](vmd.md) visualizing program for large biomolecular systems
- [Grace](grace.md) plotting graphs produced with Gromacs tools
- [PyMOL] molecular modeling system (not available at CSC)

!!! Note
    Please don't run visualization or heavy Gromacs tool scripts in
    the login node (see [usage policy for details](../../computing/usage-policy)).
    You can run the tools in the [interactive partition](../computing/running/interactive-usage.md)
    by prepending your `gmx_mpi` command with `orterun -n 1`, e.g.
    `orterun -n 1 gmx_mpi msd -n index -s topol -f traj`).

## References

Cite your work with the following references:

- GROMACS 4: Algorithms for Highly Efficient, Load-Balanced, and
    Scalable Molecular Simulation. Hess, B., Kutzner, C., van der
    Spoel, D. and Lindahl, E. J. Chem. Theory Comput., 4, 435-447
    (2008).
- GROMACS: Fast, Flexible and Free. D. van der Spoel, E. Lindahl, B.
    Hess, G. Groenhof, A. E. Mark and H. J. C.Berendsen, J. Comp. Chem.
    26 (2005) pp. 1701-1719
- *GROMACS: High performance molecular simulations through multi-level
    parallelism from laptops to supercomputers*
    M. J. Abraham, T. Murtola, R. Schulz, S. Páll, J. C. Smith, B. Hess, E.
    Lindahl *SoftwareX* 1 (2015) pp. 19-25
- *Tackling Exascale Software Challenges in Molecular Dynamics Simulations with
    GROMACS* In S. Markidis & E. Laure (Eds.), Solving Software Challenges for Exascale
    S. Páll, M. J. Abraham, C. Kutzner, B. Hess, E. Lindahl 8759 (2015) pp. 3-27
- *GROMACS 4.5: a high-throughput and highly parallel open source molecular
    simulation toolkit* S. Pronk, S. Páll, R. Schulz, P. Larsson, P. Bjelkmar, R. Apostolov, M. R.
    Shirts, J. C. Smith, P. M. Kasson, D. van der Spoel, B. Hess, and E. Lindahl
    Bioinformatics 29 (2013) pp. 845-54

See your simulation log file for more detailed references
for methods applied in your setup.

## More information

- Gromacs home page: [http://www.gromacs.org/](http://www.gromacs.org/)
- [Hands-on tutorials] by Justin A. Lemkul, on [GROMACS tutorial home](https://tutorials.gromacs.org/)
  and by [Bert de Groot group](https://www3.mpibpc.mpg.de/groups/de_groot/compbio/index.html)
- [Lots of material at BioExcel EU project]
- [HOW-TO](https://manual.gromacs.org/documentation/current/how-to/index.html) section on the
  Gromacs pages
- Gromacs [documentation] and [mdrun performance checklist]
- [The PRODRG Server] for online creation of small molecule topology
- [Advanced Gromacs Workshop materials](https://enccs.github.io/gromacs-gpu-performance/)

  [mdrun performance checklist]: https://manual.gromacs.org/current/user-guide/mdrun-performance.html
  [documentation]: http://manual.gromacs.org/documentation
  [PyMOL]: http://www.pymol.org/
  [The PRODRG Server]: https://www.sites.google.com/site/vanaaltenlab/prodrg
  [Lots of material at BioExcel EU project]: http://bioexcel.eu/software/gromacs/
  [Hands-on tutorials]: http://www.mdtutorials.com/gmx/
  [official Gromacs documentation]: https://manual.gromacs.org/documentation/current/user-guide/mdrun-features.html#running-multi-simulations

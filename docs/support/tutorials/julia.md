# Running Julia jobs
Instructions for running serial, parallel, and GPU jobs with Julia on Puhti and Mahti.

[TOC]


## Prerequisites
These instructions are adapted from the general instructions of [**running jobs**](../../computing/running/getting-started.md) on Puhti and Mahti.
Furthermore, we assume general knowledge about the [**Julia environment**](../../apps/julia.md).
We use the following Julia project structure in the example jobs.
We also assume that it is our working directory when running the commands.

```
.
├── Manifest.toml  # Automatically created a list of all dependencies
├── Project.toml   # Julia environment and dependencies
├── puhti.sh       # Puhti batch script
├── mahti.sh       # Mahti batch script
└── script.jl      # Julia script
```

We have to instantiate the project before running batch scripts.

```bash
module load julia
julia --project=. -e 'using Pkg; Pkg.instantiate()'
```

Julia has some important [environment variables for parallelization](https://docs.julialang.org/en/v1/manual/environment-variables/#Parallelization).
Because we use Slurm to reserve resources on Puhti and Mahti, we need to set the `JULIA_CPU_THREADS` and `JULIA_NUM_THREADS` environment variables to the number of reserved CPU cores.
The Julia module sets these environment variables the value of `--cpus-per-task` option using the `SLURM_CPUS_PER_TASK` environment variable.
If that option is not defined, for example, on login nodes, it sets the thread count to one.
The effect is the same as sourcing the following exports in shell.

```bash
export JULIA_CPU_THREADS=${SLURM_CPUS_PER_TASK:-1}
export JULIA_NUM_THREADS=$JULIA_CPU_THREADS
```

If you need to start a julia process with different number of threads than `JULIA_NUM_THREADS`, we recommend using the `--threads` option as follows.

```bash
julia --threads 2  # using two threads regardless of JULIA_NUM_THREADS value
```

Now we can explore project files for different types of jobs.


## Serial job
An example of a `Project.toml` project file.

```toml
[compat]
julia = "1.8"
```

An example of a `puhti.sh` Puhti batch script.

```bash
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=small
#SBATCH --time=00:15:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=1
#SBATCH --mem-per-cpu=1000

module load julia
srun julia --project=. script.jl
```

Mahti is intended for parallel computing; therefore, we do not include a Batch script for Mahti.

An example of a `script.jl` Julia code.

```julia
println("Hello world!")
```


## Single node job with multiple threads
We can use the `Base.Threads` library for multithreading in Julia.
We don't need to include libraries in `Base` in dependencies.

An example of a `Project.toml` project file.

```toml
[compat]
julia = "1.8"
```

An example of a `puhti.sh` Puhti batch script.

```bash
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=small
#SBATCH --time=00:15:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=3
#SBATCH --mem-per-cpu=1000

module load julia
srun julia --project=. script.jl
```

An example of a `mahti.sh` Mahti batch script.

```bash
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=medium
#SBATCH --time=00:15:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=128

module load julia
srun julia --project=. script.jl
```

An example of a `script.jl` Julia code.

```julia
using Base.Threads

# Number of threads
n = nthreads()

# Lets fill the id of each thread to the ids array.
ids = zeros(Int, 10*n)
@threads for i in eachindex(ids)
    ids[i] = threadid()
end

# Print the outputs.
println(n)
println(ids)
```


## Single node job with multiple processes
We can use `Distributed`, a standard library for multiple processes in Julia.
When we add `Distributed`, the `Project.toml` file will look as follows.

An example of a `Project.toml` project file.

```toml
[deps]
Distributed = "8ba89e20-285c-5b6f-9357-94700520ee1b"

[compat]
julia = "1.8"
```

An example of a `puhti.sh` Puhti batch script.

```bash
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=small
#SBATCH --time=00:15:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=3
#SBATCH --mem-per-cpu=1000

module load julia
srun julia --project=. script.jl
```

An example of a `mahti.sh` Mahti batch script.

```bash
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=medium
#SBATCH --time=00:15:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=128

module load julia
srun julia --project=. script.jl
```

An example of a `script.jl` Julia code.

```julia
using Distributed

# Add new workers before using @everywhere macros; otherwise the code won't be included in the new processes.
addprocs(Sys.CPU_THREADS)

# We must use the @everywhere macro to include the task function in the worker processes.
@everywhere function task()
    return (myid(), gethostname(), getpid())
end

# We run the task function in each worker process.
futures = [@spawnat id task() for id in workers()]

# Then, we fetch the output from the processes.
outputs = fetch.(futures)

# Remove processes after we are done.
rmprocs.(workers())

# Print the outputs.
println(task())
println.(outputs)
```


## Multi-node job with MPI
We can use the `MPI.jl` package to use MPI for multi-node parallelism.
In Puhti and Mahti it uses OpenMPI.
We have installed a specific version of `MPI.jl` to the shared environment.
We recommend that you use that version because it has been tested to work.
You can find the version by activating the shared environment and running `Pkg.status` as follows:

```bash
julia --project="$JULIA_CSC_ENVIRONMENT" -e 'using Pkg; Pkg.status("MPI")'
```

An example of a `Project.toml` project file.

```toml
[deps]
MPI = "da04e1cc-30fd-572f-bb4f-1f8673147195"

[compat]
julia = "1.8"
MPI = "=0.20.8"
```

An example of a `puhti.sh` Puhti batch script.

```bash
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=large
#SBATCH --time=00:15:00
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=2
#SBATCH --cpus-per-task=1
#SBATCH --mem-per-cpu=1000

module load julia
srun julia --project=. script.jl
```

An example of a `mahti.sh` Mahti batch script.

```bash
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=medium
#SBATCH --time=00:15:00
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=128
#SBATCH --cpus-per-task=1

module load julia
srun julia --project=. script.jl
```

An example of a `script.jl` Julia code.

```julia
using MPI

MPI.Init()
comm = MPI.COMM_WORLD
rank = MPI.Comm_rank(comm)
size = MPI.Comm_size(comm)
println("Hello from rank $(rank) out of $(size) from host $(gethostname()) and process $(getpid()).")
MPI.Barrier(comm)
```


## Single node job with one GPU
Puhti and Mahti contain NVidia GPUs.
We can use them via the `CUDA.jl` package.
We have installed a specific version of `CUDA.jl` to the shared environment.
We recommend that you use that version because it has been tested to work.
You can find the version by activating the shared environment and running `Pkg.status` as follows:

```bash
julia --project="$JULIA_CSC_ENVIRONMENT" -e 'using Pkg; Pkg.status("CUDA")'
```

An example of a `Project.toml` project file.

```toml
[deps]
CUDA = "052768ef-5323-5732-b1bb-66c8b64840ba"

[compat]
julia = "1.8"
CUDA = "=4.0.1"
```

An example of a `puhti.sh` Puhti batch script.

```bash
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=gpu
#SBATCH --time=00:15:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=10
#SBATCH --mem=4000
#SBATCH --gres=gpu:v100:1

module load julia
srun julia --project=. script.jl
```

An example of a `mahti.sh` Mahti batch script.

```bash
#!/bin/bash
#SBATCH --job-name=example
#SBATCH --account=<project>
#SBATCH --partition=gpusmall
#SBATCH --time=00:15:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=32
#SBATCH --gres=gpu:a100:1

module load julia
srun julia --project=. script.jl
```

An example of a `script.jl` Julia code.

```julia
using CUDA

@show CUDA.versioninfo()
n = 2^20
x = CUDA.fill(1.0f0, n)
y = CUDA.fill(2.0f0, n)
y .+= x
println(all(Array(y) .== 3.0f0))
```


## Linear algebra backends and threading
The default `LinearAlgebra` backend on Julia is OpenBLAS.
If we want to, we can use MKL instead of OpenBLAS.
MKL is often faster than OpenBLAS, especially on Puhti, when using multiple threads.
We should load the `MKL` library before other linear algebra libraries.

```julia
using MKL
```

The Julia module sets the number of threads for OpenBLAS and MKL backends to the number of CPU threads.

```bash
export OPENBLAS_NUM_THREADS=$JULIA_CPU_THREADS
export MKL_NUM_THREADS=$JULIA_CPU_THREADS
```

We must be careful not to oversubscribe cores when using BLAS operations within Julia threads or processes.
We can change the amount of BLAS threads at runtime using the `BLAS.set_num_threads` function.

```julia
using Base.Threads
using LinearAlgebra

# Number of threads
n = nthreads()

# Define a matrix
X = rand(1000, 1000)

# Set the number of threads to one before performing BLAS operations of multiple Julia threads.
BLAS.set_num_threads(1)
Y = zeros(n)
@threads for i in 1:n  # uses n Julia threads
    Y[i] = sum(X * X)  # uses one BLAS thread
end

# Set the number of threads back to the default when performing BLAS operation on a single Julia Thread.
BLAS.set_num_threads(n)
Z = zeros(n)
for i in 1:n           # uses one Julia thread
    Z[i] = sum(X * X)  # uses n BLAS threads
end
```

There are [caveats](https://discourse.julialang.org/t/matrix-multiplication-is-slower-when-multithreading-in-julia/56227/12?u=carstenbauer) for using different numbers than one or all cores of BLAS threads on OpenBLAS and MKL.


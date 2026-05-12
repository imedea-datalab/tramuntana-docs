# Tramuntana HPC Cluster - Submit Jobs with SLURM, Conda & R

This guide covers everything you need to know about setting up software environments with Conda and R, and submitting computational jobs using SLURM on the Tramuntana cluster.

---

## Table of Contents

5. [Setting Up Conda Environments](#setting-up-conda-environments)
6. [Using R for Statistical Computing](#using-r-for-statistical-computing)
7. [Submitting Jobs with SLURM](#submitting-jobs-with-slurm)
8. [Job Arrays - Running Multiple Similar Jobs](#job-arrays---running-multiple-similar-jobs)
9. [Monitoring Your Jobs](#monitoring-your-jobs)

**Other Guides:**
- [Overview & Getting Started](01-tramuntana-overview-getting-started.md)
- [Best Practices & Help](03-best-practices-help.md)

---

## Setting Up Conda Environments

### What is Conda?

Conda is a package manager that helps you install and manage software packages and their dependencies. It's essential for Python-based scientific computing.

### Step 1: Initialize Conda

The first time you use conda, initialize it:

```bash
# Initialize conda for your shell
/data/shared/miniforge3/bin/conda init bash

# Reload your shell configuration
source ~/.bashrc
```

You should now see `(base)` at the beginning of your prompt.

### Step 2: Create Your First Environment

```bash
# Create a new environment named 'myproject' with Python 3.11
conda create -n myproject python=3.11

# Activate the environment
conda activate myproject
```

### Step 3: Install Packages

```bash
# Install packages using conda
conda install numpy pandas matplotlib

# Or install using pip (if not available in conda)
pip install some-package
```

### Step 4: Save Your Environment

Create a shareable environment file:

```bash
# Export your environment
conda env export > environment.yml
```

This creates a file that others can use to recreate your exact environment:

```bash
# Recreate environment from file
conda env create -f environment.yml
```

### Managing Conda Environments

```bash
# List all environments
conda env list

# Activate an environment
conda activate myproject

# Deactivate current environment
conda deactivate

# Remove an environment
conda env remove -n myproject

# Update all packages in current environment
conda update --all
```

### Saving Disk Space with Conda

Conda can use a lot of disk space. Here's how to manage it:

```bash
# Clean package cache
conda clean --all

# Check conda disk usage
du -sh ~/.conda/

# Move conda environments to /data (if needed)
# First, create a directory in /data
mkdir -p /data/your-username/conda-envs

# Create environments there
conda create --prefix /data/your-username/conda-envs/myproject python=3.11

# Activate with full path
conda activate /data/your-username/conda-envs/myproject
```

---

## Using R for Statistical Computing

### What is R?

R is a programming language and environment for statistical computing and graphics. It's widely used for data analysis, statistical modeling, and visualization.

### R Installation on Tramuntana

R 4.4.2 is installed system-wide and available on all compute nodes:

- **Location**: `/data/shared/R/4.4.2`
- **Version**: R 4.4.2 (2024-10-31) "Pile of Leaves"
- **Setup script**: `/data/shared/R/setup-R.sh`

### Step 1: Load R Environment

Before using R, load the R environment:

```bash
# Load R
source /data/shared/R/setup-R.sh

# Verify R is available
R --version
```

### R Job Script Example

```bash
#!/bin/bash
#SBATCH --job-name=r-analysis
#SBATCH --output=r-output-%j.txt
#SBATCH --error=r-error-%j.err
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --partition=cpu

# Load R environment
source /data/shared/R/setup-R.sh

# Print R information
echo "R version:"
R --version

echo "Running R analysis..."

# Run R script
Rscript my_analysis.R

# Or use R CMD BATCH
# R CMD BATCH my_analysis.R

echo "Analysis complete!"
```

### Using RStudio Server for Interactive R Work

RStudio Server provides a familiar RStudio interface in your web browser, running on the cluster's compute nodes with full access to your requested resources.

#### When to Use RStudio Server vs Batch Jobs

**Use RStudio Server for:**
- Interactive data exploration and visualization
- Developing and debugging R scripts
- Testing code with different parameters
- Interactive statistical analysis
- Learning R or new packages

**Use batch SLURM jobs for:**
- Long-running analyses (>4 hours)
- Production workflows
- Automated pipelines
- Jobs that don't need interaction

#### Quick Start Guide

**Step 1: Request Resources**

From tramuntana, request compute resources:

```bash
# Basic session (8 CPUs, 16GB RAM, 4 hours)
salloc --cpus-per-task=8 --mem=16G --time=04:00:00

# Intensive session (16 CPUs, 64GB RAM, 8 hours)
salloc --cpus-per-task=16 --mem=64G --time=08:00:00
```

Check which node was assigned:
```bash
squeue -u $USER
# Note the NODELIST column (e.g., pampero, ada, tramuntana-n1, thor)
```

**Step 2: Create SSH Tunnel**

From your PC (with VPN active), open a new terminal:

```bash
# Replace 'username' with your username
# Replace 'nodename' with the node from squeue (e.g., pampero)
ssh -L 8787:localhost:8787 -J username@tramuntana username@nodename
```

Keep this terminal open while using RStudio.

**Step 3: Open RStudio**

In your web browser, go to:
```
http://localhost:8787
```

Login with your LDAP credentials (same as tramuntana).

**Step 4: Working in RStudio**

Your RStudio session has:
- Access to all your files in `/home/username/` and `/data/`
- The exact resources you requested (CPUs, memory)
- All R packages you've installed

Verify your environment:
```r
# Check you're on the right node
system("hostname")

# Check available resources
system("free -h")
system("nproc")

# Check R library paths
.libPaths()
# Should show: ~/R/x86_64-pc-linux-gnu-library/4.4
```

**Step 5: Install Packages**

Packages install automatically to your personal library:

```r
install.packages("ggplot2")
install.packages("dplyr")
install.packages("tidyr")
```

Installed packages are available in both RStudio Server and SLURM batch jobs.

**Step 6: Close Session**

When finished:
1. Save your work in RStudio
2. Close browser tab
3. In the terminal where you're connected to the node: `exit`
4. Close the SSH tunnel terminal (Ctrl+C)

#### RStudio Server Tips

**Package Management:**
- Packages install once and are available everywhere
- Check what's installed: `.libPaths()` and `installed.packages()`
- Large packages (>100MB) count toward your home quota

**Working with Data:**
- Small datasets (<10GB): Can keep in `/home/`
- Large datasets: Store in `/data/YourGroup/` and read directly
- Very large data: Consider reading in chunks with `data.table::fread()`

**Resource Monitoring:**
```r
# Monitor memory usage
pryr::mem_used()

# Clear large objects
rm(large_object)
gc()  # Force garbage collection
```

**Saving Work:**
- RStudio auto-saves your scripts
- For long sessions, manually save workspace: `save.image("mywork.RData")`
- Load later: `load("mywork.RData")`

**Common Issues:**

- **Port already in use**: Use different port: `ssh -L 18787:localhost:8787 ...` then open `http://localhost:18787`
- **Session timeout**: RStudio closes after 2 hours of inactivity - save regularly
- **Out of memory**: Request more with `salloc --mem=...`

For detailed instructions and troubleshooting, see the separate RStudio Server user guide.

---

## Submitting Jobs with SLURM

### What is SLURM?

SLURM is the job scheduler that manages computational resources on Tramuntana. Instead of running programs directly, you submit job scripts that tell SLURM what resources you need and what to run.

### Your First Job Script

Create a file called `first-job.sh`:

```bash
#!/bin/bash
#SBATCH --job-name=test-job          # Name of your job
#SBATCH --output=output-%j.txt       # Output file (%j = job ID)
#SBATCH --error=error-%j.txt         # Error file
#SBATCH --ntasks=1                   # Number of tasks
#SBATCH --cpus-per-task=4            # CPUs per task
#SBATCH --mem=8G                     # Memory needed
#SBATCH --time=01:00:00              # Time limit (HH:MM:SS)
#SBATCH --partition=cpu              # Partition to use
#SBATCH --mail-type=END,FAIL         # Email when job ends or fails (optional)
#SBATCH --mail-user=your-email@uib.es # Your email (optional)

# Print job information
echo "Job started at: $(date)"
echo "Running on node: $(hostname)"
echo "Job ID: $SLURM_JOB_ID"
echo "Partition: $SLURM_JOB_PARTITION"

# Your commands here
python my-script.py

echo "Job finished at: $(date)"
```

### Submit Your Job

```bash
# Submit the job
sbatch first-job.sh

# You'll see a message like:
# Submitted batch job 12345
```

### Job Script for GPU Computing

The cluster has two GPU nodes available:
- **tramuntana-n1**: 1x NVIDIA L40S (48GB)
- **thor**: 2x NVIDIA RTX 6000 Ada (48GB each)

Request GPUs using the generic `--gres=gpu:` syntax. SLURM will assign available GPUs automatically:

```bash
#!/bin/bash
#SBATCH --job-name=gpu-job
#SBATCH --output=gpu-output-%j.txt
#SBATCH --error=gpu-error-%j.txt
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --gres=gpu:1                 # Request 1 GPU (any available)
#SBATCH --time=04:00:00
#SBATCH --partition=gpu              # Use GPU partition
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=your-email@uib.es

# Load your conda environment
source /data/shared/miniforge3/bin/activate
conda activate myproject

# Verify GPU is accessible
nvidia-smi

# Your GPU program
python train-model.py

echo "GPU job completed"
```

**For multi-GPU jobs** (thor has 2 GPUs):
```bash
#SBATCH --gres=gpu:2                 # Request 2 GPUs
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=96G
```

### Job Script with Conda Environment

```bash
#!/bin/bash
#SBATCH --job-name=conda-job
#SBATCH --output=output-%j.txt
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=02:00:00

# Initialize conda
source /data/shared/miniforge3/bin/activate

# Activate your environment
conda activate myproject

# Run your analysis
python analysis.py

# Deactivate when done
conda deactivate
```

### Job Script with R

```bash
#!/bin/bash
#SBATCH --job-name=r-analysis
#SBATCH --output=r-output-%j.txt
#SBATCH --error=r-error-%j.err
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --partition=cpu

# Load R environment
source /data/shared/R/setup-R.sh

# Run R analysis
Rscript my_analysis.R

echo "R analysis completed"
```

### Understanding SLURM Options

| Option | Description | Example | Default | Maximum |
|--------|-------------|---------|---------|---------|
| `--job-name` | Name for your job | `--job-name=experiment1` | N/A | N/A |
| `--output` | Where to save output | `--output=results.txt` | N/A | N/A |
| `--error` | Where to save errors | `--error=errors.txt` | N/A | N/A |
| `--ntasks` | Number of tasks | `--ntasks=1` | 1 | N/A |
| `--cpus-per-task` | CPUs per task | `--cpus-per-task=8` | 1 | 256 (cpu/express), 128 (gpu) |
| `--mem` | Memory required | `--mem=32G` | 2GB (cpu/express), 48GB (gpu) | 1.5TB |
| `--time` | Maximum runtime | `--time=12:00:00` | 1 hour | 10 days (cpu/gpu), 2 hours (express) |
| `--partition` | Queue to use | `--partition=cpu` or `--partition=gpu` | cpu | N/A |
| `--gres=gpu` | GPU resources | `--gres=gpu:1` (any GPU), `--gres=gpu:2` (2 GPUs) | None | 2 (thor), 1 (tramuntana-n1) |
| `--nodelist` | Specific node | `--nodelist=pampero,ada,thor,tramuntana-n1` | N/A | N/A |
| `--mail-type` | Email notifications | `--mail-type=BEGIN,END,FAIL` | None | N/A |
| `--mail-user` | Email address | `--mail-user=you@uib.es` | N/A | N/A |

### Available Partitions

Tramuntana has three SLURM partitions optimized for different workloads:

| Partition | Nodes | Max Time | Priority | Default | Best For |
|-----------|-------|----------|----------|---------|----------|
| **express** | pampero, ada, tramuntana-n1, thor | 2 hours | 200 | No | Quick tests, debugging, compilation |
| **cpu** | pampero, ada, tramuntana-n1, thor | 10 days | 50 | Yes | CPU-intensive parallel jobs, R analyses |
| **gpu** | tramuntana-n1, thor | 10 days | 100 | No | AI/ML, GPU-accelerated tasks |

**Resource Limits:**

| Partition | Default CPUs | Max CPUs | Default Memory | Max Memory | User Limits |
|-----------|--------------|----------|----------------|------------|-------------|
| **express** | 1 core | 256 cores | 2 GB | 1.5 TB | Priority access for quick jobs |
| **cpu** | 1 core | 256 cores | 2 GB | 1.5 TB | Max 20 jobs running, 50 in queue |
| **gpu** | 1 core | 128 cores | 48 GB | 1.5 TB | Max 20 jobs running, 50 in queue |

**Important Notes:**
- Jobs requesting more than default resources must explicitly specify them using `--cpus-per-task` and `--mem`
- Jobs exceeding memory limits will be automatically terminated
- Default job runtime is 1 hour; specify `--time` for longer jobs
- All nodes are available in cpu and express partitions
- GPU nodes (tramuntana-n1, thor) available in gpu partition; thor has 2x RTX 6000 Ada GPUs
- **User limits**: Maximum 20 jobs running simultaneously, maximum 50 jobs in queue per user

---

## Job Arrays - Running Multiple Similar Jobs

Job arrays allow you to submit many similar jobs with a single script, ideal for parameter sweeps, processing multiple files, or running the same analysis with different inputs.

### Basic Job Array Syntax

```bash
#!/bin/bash
#SBATCH --job-name=array_job
#SBATCH --output=array_%A_%a.out     # %A = job ID, %a = array task ID
#SBATCH --error=array_%A_%a.err
#SBATCH --array=1-10                  # Run tasks 1 through 10
#SBATCH --partition=cpu
#SBATCH --time=01:00:00
#SBATCH --mem=8G

# SLURM_ARRAY_TASK_ID contains the current task number (1-10)
echo "Processing task $SLURM_ARRAY_TASK_ID"

# Example: process different input files
INPUT_FILE="data_${SLURM_ARRAY_TASK_ID}.txt"
python process.py $INPUT_FILE

# Example: run with different parameters
PARAM=$SLURM_ARRAY_TASK_ID
python simulation.py --parameter=$PARAM
```

### R Job Array Example

```bash
#!/bin/bash
#SBATCH --job-name=r-array
#SBATCH --output=r-array_%A_%a.out
#SBATCH --error=r-array_%A_%a.err
#SBATCH --array=1-20
#SBATCH --partition=cpu
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=02:00:00

# Load R
source /data/shared/R/setup-R.sh

# Pass array task ID to R script
Rscript analysis.R $SLURM_ARRAY_TASK_ID

# Or create R script that uses the task ID
R --vanilla <<EOF
task_id <- as.numeric(Sys.getenv("SLURM_ARRAY_TASK_ID"))
cat("Processing task:", task_id, "\n")

# Your R analysis here using task_id
# For example: read different data files
data_file <- paste0("data_", task_id, ".csv")
data <- read.csv(data_file)

# Process and save results
results <- analyze(data)
write.csv(results, paste0("results_", task_id, ".csv"))
EOF
```

---

## Monitoring Your Jobs

### Check Job Status

```bash
# View your running and pending jobs
squeue -u your-username

# View all jobs in the queue
squeue

# Detailed information about a specific job
scontrol show job 12345
```

### Understanding Job States

- **PD** (Pending): Job is waiting for resources
- **R** (Running): Job is currently running
- **CG** (Completing): Job is finishing
- **CD** (Completed): Job finished successfully
- **F** (Failed): Job failed
- **CA** (Cancelled): Job was cancelled

### Cancel a Job

```bash
# Cancel a specific job
scancel 12345

# Cancel all your jobs
scancel -u your-username

# Cancel all pending jobs
scancel -u your-username -t PENDING
```

### View Job History

```bash
# View completed jobs
sacct -u your-username

# Detailed view of a specific job
sacct -j 12345 --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS

# View jobs from last 7 days
sacct -u your-username -S $(date -d '7 days ago' +%Y-%m-%d)
```

### Check Job Output in Real-Time

```bash
# Watch output file as job runs
tail -f output-12345.txt

# Check last 50 lines
tail -50 output-12345.txt

# Follow both output and error
tail -f output-12345.txt error-12345.txt
```

### Check Job Resource Usage

```bash
# Get efficiency report for completed job
seff 12345

# Detailed resource usage
sacct -j 12345 --format=JobID,JobName,MaxRSS,Elapsed,CPUTime,TotalCPU
```

---

## Quick Job Submission Checklist

**Before Running Jobs**:
- [ ] Initialize conda or R environment
- [ ] Create your environment (conda) or load R setup
- [ ] Test your code in express partition with small data
- [ ] Estimate required resources (CPUs, memory, time)
- [ ] Choose appropriate partition (express/cpu/gpu)
- [ ] Create job script with appropriate SLURM directives
- [ ] Add email notifications if desired
- [ ] Double-check file paths in your script
- [ ] Ensure data is in the right location (/data for large datasets)
- [ ] Check you're not at the 20 running jobs limit

**After Submitting Jobs**:
- [ ] Note your job ID from sbatch output
- [ ] Check job status with `squeue -u $USER`
- [ ] Monitor output files (`tail -f output-jobid.txt`)
- [ ] Review resource usage after completion (`seff jobid`)
- [ ] Clean up unnecessary files
- [ ] Check your disk quota usage regularly

---

**Next Steps**: Continue to [Best Practices & Help](03-best-practices-help.md) for troubleshooting and advanced tips

**Institution**: IMEDEA UIB-CSIC  
**Cluster Name**: Tramuntana  
**Last Updated**: April 20, 2026
**Version**: 1.3
**Changes**: 
- v1.3 (Apr 20, 2026): Updated SLURM version to 23.11.4 following cluster-wide upgrade to Ubuntu 24.04 LTS.
- v1.2 (Feb 18, 2026): Added comprehensive RStudio Server section with setup, usage tips, and troubleshooting. Updated R package installation path.
- v1.1: Updated partition information for new nodes (thor, ada). Updated resource limits (256 cores, 1.5TB RAM max). Changed GPU request syntax to generic `--gres=gpu:N`. Updated partition max time to 10 days.
- v1.0 (Nov 17, 2025): Added R 4.4.2 usage instructions, examples, and job array patterns

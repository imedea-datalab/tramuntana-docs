# Advanced SLURM Commands & Monitoring

This guide covers advanced usage of SLURM, including deep-dives into job parameters, interactive GPU monitoring, job arrays, and comprehensive command references.

**Other Practical Guides:**
- [**Quick Start & Commands**](0-quick-start.md) — Connecting to the cluster and running your first job.
- [**Software & Environments**](environments-and-software.md) — Managing Python environments (`uv`, Conda) and using R/RStudio.

---

## 📑 Table of Contents

| # | Topic |
|---|-------|
| 1 | [Important SLURM Arguments Explained](#1-important-slurm-arguments-explained) |
| 2 | [Checking Resource Usage (GPU, CPU, RAM)](#2-checking-resource-usage-gpu-cpu-ram) |
| 3 | [Job Arrays — Running Many Independent Jobs](#3-job-arrays--running-many-independent-jobs) |
| 4 | [Commands & Monitoring Reference](#4-commands--monitoring-reference) |

---
## 1. Important SLURM Arguments Explained

Whether you're using `sbatch`, `salloc`, or `srun`, you use these arguments to tell SLURM exactly what hardware you need. Let's build up from the smallest unit to the largest.

### The Hierarchy: CPUs → Tasks → Nodes

Think of it like a building:
- **CPUs** are the individual desks where work gets done.
- A **Task** (a process) is a person sitting at one or more desks. Each task is an independent running copy of your program.
- A **Node** is an entire office building full of desks.

---

#### `--cpus-per-task=4`
**"Give each of my processes 4 CPU cores."**

Most of the time, your Python script is a single process that just needs one CPU. But some libraries (like NumPy, PyTorch data loaders, or anything using multi-threading or C extensions under the hood) can use multiple CPUs at once within a single process. If your code does that, give it more CPUs-per-task.

---

#### `--ntasks=4`
**"I need to run 4 separate processes."**

This books 4 processing "slots" on the cluster. But here's the crucial detail: **SLURM only books the slots — it does NOT automatically run your code 4 times.** If you write `python my_script.py` in your `.slurm` file, Linux treats it like a normal desktop and only spawns **one process** on whichever core it was dropped into. The other 3 slots sit completely empty.

To actually use all 4 slots, you need a **launcher** — either `srun` (which is SLURM's built-in launcher) or a library like `mpirun` that knows how to read SLURM's environment and spread your code across the booked slots.

---

#### `--nodes=2`
**"I need 2 physical servers."**

Use this when a single machine doesn't have enough resources for your job. For example, you check `sinfo` and see that ada has only 50 free cores but you need 100 — so you request 2 nodes to combine resources across machines.

You can check what's currently available on each node:
```bash
sinfo -N -l
```

> **Important:** When your job runs across multiple nodes, your code needs to communicate over the network between them. This only works if your code uses a parallel library (like MPI or PyTorch DDP). A regular `python script.py` does **not** know how to talk across network boundaries.

---

#### `--ntasks-per-node=2`
**"Spread my tasks evenly across nodes."**

If you asked for `--nodes=2 --ntasks=4`, SLURM could put all 4 tasks on one node. This flag forces it to put exactly 2 on each.

---

#### `--nodelist=thor,ada`
**"I specifically want to run on these exact machines."**

This forces the job scheduler to use the exact nodes you name. If any of them are busy, your job will stay pending in the queue until they're free.

This is useful for GPU jobs when you know which node has the GPU you want:
```bash
#SBATCH --nodelist=thor       # Run specifically on thor (2x RTX 6000 Ada GPUs)
```

---

#### `--partition=gpu`
**"Put me in the GPU lane."**

Chooses which partition (see Section 2 above) your job will run in. If you need a GPU, you must use `--partition=gpu`.

> [!WARNING]
> **Important Notes — Defaults & Limits on Tramuntana**
>
> - Jobs requesting more than default resources must **explicitly specify** them using `--cpus-per-task` and `--mem`.
> - Jobs exceeding their memory limits will be **automatically terminated** by SLURM (via cgroups).
> - Default job runtime is **1 hour**; specify `--time` for longer jobs.
> - All nodes are available in the `cpu` and `express` partitions.
> - GPU nodes (`tramuntana-n1`, `thor`) are available in the `gpu` partition; `thor` has 2× RTX 6000 Ada GPUs.
> - **User limits**: Maximum **20 jobs running** simultaneously, maximum **50 jobs in queue** per user.

---

## 5. GPU Jobs on Tramuntana

Only two nodes have GPUs:
- **tramuntana-n1**: 1× NVIDIA L40S (48 GB VRAM)
- **thor**: 2× NVIDIA RTX 6000 Ada (48 GB VRAM each)

To use a GPU, you need two things: the `gpu` partition and the `--gres=gpu_mem:N` flag, where **N is the amount of GPU VRAM in GB** that your job needs.

> [!IMPORTANT]
> **On Tramuntana, you request GPU *memory*, not whole GPUs.** The cluster shares GPUs between users. Instead of locking out an entire 48 GB GPU for a job that only needs 8 GB, you ask for exactly what you need: `--gres=gpu_mem:8`. This lets multiple jobs run on the same physical GPU simultaneously.
>
> If you try to use the standard `--gres=gpu:1` syntax, the system will silently cap your allocation to a 12 GB default. **Always use `--gres=gpu_mem:N` to get the amount you actually want.**

### GPU Job Script Example

```bash
#!/bin/bash
#SBATCH --job-name=gpu-training
#SBATCH --output=gpu-output_%j.txt
#SBATCH --error=gpu-error_%j.txt
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --gres=gpu_mem:20             # Request 20 GB of GPU VRAM
#SBATCH --time=08:00:00
#SBATCH --partition=gpu               # Must use GPU partition!
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=your-email@uib.es

# --- Using uv (recommended) ---
uv run python train_model.py

# --- Using Conda instead? ---
# source /data/shared/miniforge3/bin/activate
# conda activate myproject
# python train_model.py
# conda deactivate
```

> [!CAUTION]
> 🚨 **CHANGE YOUR EMAIL!** Replace `your-email@uib.es` with **your actual email address!**

If you want to request a **specific node** (e.g., you need thor for its higher RAM):
```bash
#SBATCH --gres=gpu_mem:40
#SBATCH --nodelist=thor
```

### Common GPU Memory Amounts

| What you're doing | Suggested `gpu_mem` |
|---|---|
| Small model inference / quick test | `gpu_mem:4` to `gpu_mem:8` |
| Medium model training | `gpu_mem:16` to `gpu_mem:24` |
| Large model / full GPU | `gpu_mem:48` (entire GPU) |

> [!NOTE]
> Each GPU has 48 GB of VRAM. The maximum you can request for a single job is `gpu_mem:48` (one full GPU).
>
> **Multi-GPU jobs are not currently supported.** The cluster does not yet have the configuration for jobs that span multiple GPUs with inter-GPU communication (e.g., PyTorch DDP across 2 GPUs). If your model fits in 48 GB, you're good. If there's demand for multi-GPU training in the future, this can be set up.

### How GPU Sharing Works

Multiple people can share the same physical GPU on Tramuntana. When you request `--gres=gpu_mem:20`, SLURM reserves 20 GB of VRAM on one GPU for your job. A watchdog process continuously monitors actual memory usage and enforces the limit — so you won't crash other people's work and they won't crash yours. Just write your code normally (e.g., `tensor.to("cuda:0")`) and SLURM handles the rest.

---

## 2. Checking Resource Usage (GPU, CPU, RAM)

### Checking CPU & RAM Usage

To quickly check how much CPU and RAM is currently being used across all nodes in the cluster, use the custom command:

```bash
check_cpu_ram
```

This will output a summary table showing the total and used CPUs and RAM for each node:

```text
=======================================================================
                      CPU & RAM STATUS
=======================================================================
Node            | CPU (Used/Total)   | RAM GB (Used/Total)
-----------------------------------------------------------------------
pampero         |    0 /   64 CPUs |     0 /   378 GB
barracuda       |    0 /   20 CPUs |     0 /    62 GB
tramuntana-n1   |    0 /   48 CPUs |     0 /   377 GB
ada             |    0 /  256 CPUs |     0 /   756 GB
thor            |  114 /  128 CPUs |  1228 /  1512 GB
=======================================================================
```

This is very useful to check before submitting a large job to ensure there are enough resources available on the node you want to use.

### Checking GPU Memory Usage

Because GPUs are shared, you need to be aware of what's actually happening on them. There are two sides to GPU monitoring:

**1. SLURM's bookkeeping** — what SLURM *thinks* is reserved (the accounting side).
**2. Actual GPU usage** — what the hardware is *really* using right now.

### Step 1: Overview of free GPU memory
Before submitting a job, you can quickly check how much GPU memory is currently free across all nodes using the custom command:
```bash
check_gpu
```
*(This shows total, used, and free memory shards/GBs per node).*

### Step 2: Check what jobs are running on the GPU nodes
```bash
squeue -u $USER
```
This shows your jobs and which nodes they're on.

### Step 3: Peek inside a running GPU job

To see the live GPU status on the node where your job is running, you can "hop into" the job using `srun --jobid`:

```bash
srun --jobid=<YOUR_JOB_ID> --overlap --pty nvidia-smi
```

This runs `nvidia-smi` inside your existing job's allocation (using `--overlap` to share the same resources — it doesn't request new ones). You'll see output like this:

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.86.15              Driver Version: 570.86.15      CUDA Version: 12.8     |
|--------------------------------------------+------------------------+-------------------+
| GPU  Name                 Persistence-M    | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap    |           Memory-Usage | GPU-Util  Compute M. |
|============================================+========================+===================|
|   0  NVIDIA RTX 6000 Ada Generation   On   | 00000000:31:00.0   Off |                Off |
| 30%   42C    P8              24W / 300W    |    8192MiB / 49140MiB  |      0%    Default |
+--------------------------------------------+------------------------+-------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                           GPU Memory     |
|        ID   ID                                                            Usage          |
|=========================================================================================|
|    0   N/A  N/A     12345      C   python                                    8192MiB     |
|    0   N/A  N/A     67890      C   python                                    4096MiB     |
+-----------------------------------------------------------------------------------------+
```

The `Processes` table at the bottom shows you each job's actual VRAM consumption — `8192MiB` means that process is using ~8 GB of the GPU's 48 GB.

### Step 3b: Quick GPU eavesdrop (without an existing job)

Want to check GPU usage *before* submitting a job, or just curious how busy the GPUs are right now? You can launch a lightweight "eavesdrop" session that requests only 1 GB of VRAM and gives you live stats:

```bash
srun --gres=gpu_mem:1 --time=00:02:00 --partition=gpu --immediate=3 --pty watch -n 1 nvidia-smi
```

Here's what each flag does:

| Flag | Purpose |
|---|---|
| `--gres=gpu_mem:1` | Asks for only 1 GB of VRAM — the bare minimum so you're barely touching the GPU. |
| `--time=00:02:00` | Short time limit, so the scheduler gives you high priority. |
| `--immediate=3` | If resources aren't available within 3 seconds, cancel and exit instead of waiting. If even 1 GB isn't available, the GPU is completely maxed out and there's no point running a job. |
| `watch -n 1 nvidia-smi` | Refreshes `nvidia-smi` every second so you see live GPU stats. |

This is perfect for a quick sanity check before submitting a big training job.

### Step 4: Check efficiency after a job completes
```bash
seff <job_id>
```

---

### Understanding `--overlap` (The "Sharing" Flag)

`--overlap` tells SLURM: *"Let me run this new task on resources that are already busy running something else."*

You generally do **not** need `--overlap` for a basic interactive session. The main use case is when you've allocated resources, started a long-running job, and then want to run a second command (like `nvidia-smi`) on that same allocation *at the same time*, sharing the same CPUs. Without `--overlap`, SLURM would block the second command because the CPUs are already in use.

**Example:**
```bash
# You have a training job running as job 12345.
# You want to check GPU status without stopping it:
srun --jobid=12345 --overlap --pty nvidia-smi
```

This is the main reason you see `--overlap` in the GPU monitoring commands above — it lets you "eavesdrop" on your own running job without interfering with it.

---

## 3. Job Arrays — Running Many Independent Jobs

Job arrays are for when you want to run the **same script many times** with different inputs — like trying different hyperparameters, processing different data files, or running simulations with different random seeds. Each copy runs completely independently — no communication between them.

### When to use `--array` vs `--ntasks`

This is a common source of confusion:

- **`--ntasks`** = *"I have ONE program that is split into multiple communicating processes"* (like MPI or DDP). They run simultaneously and talk to each other.
- **`--array`** = *"I have MANY independent copies of the same program"* that don't talk to each other at all. Each one gets its own separate job with its own resources.

> [!IMPORTANT]
> Each job in an array gets its own **separate** allocation of the resources you specified. So if you write `--mem=16G --array=1-10`, that's 10 independent jobs, each with 16 GB — not 10 jobs sharing 16 GB!

### Example: Hyperparameter Sweep

Imagine you want to train a model with 5 different learning rates. Instead of writing 5 separate `.slurm` files, you write **one** and use `--array`:

```bash
#!/bin/bash
#SBATCH --job-name=hyperparam_sweep
#SBATCH --output=sweep_%A_%a.txt      # %A = main job ID, %a = array task ID
#SBATCH --error=sweep_err_%A_%a.txt
#SBATCH --array=1-5                    # Run this 5 times (task IDs: 1, 2, 3, 4, 5)
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --gres=gpu_mem:12              # 12 GB of GPU VRAM per array task
#SBATCH --time=06:00:00
#SBATCH --partition=gpu
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=your-email@uib.es

# SLURM gives each copy a unique number in $SLURM_ARRAY_TASK_ID
# We can use this to pick different hyperparameters!

# Define learning rates as an array
LEARNING_RATES=(0.001 0.005 0.01 0.05 0.1)

# Pick the one that corresponds to this task's ID (arrays are 0-indexed)
LR=${LEARNING_RATES[$((SLURM_ARRAY_TASK_ID - 1))]}

echo "Running task $SLURM_ARRAY_TASK_ID with learning rate $LR"

uv run python train_model.py --lr=$LR --experiment_id=$SLURM_ARRAY_TASK_ID
```

> [!CAUTION]
> 🚨 **CHANGE YOUR EMAIL!** Replace `your-email@uib.es` with **your actual email address!**

Submit it once:
```bash
sbatch hyperparam_sweep.slurm
```

SLURM creates 5 independent jobs. Each one gets its own GPU, its own 16 GB of RAM, and its own unique `$SLURM_ARRAY_TASK_ID` (1 through 5). They can all run in parallel on different nodes if resources are available.

### Other Array Patterns

```bash
#SBATCH --array=1-100          # Tasks 1 through 100
#SBATCH --array=1-100%10       # Same, but max 10 running at a time
#SBATCH --array=0-20:5         # Tasks 0, 5, 10, 15, 20 (step size of 5)
```

---

## 4. Commands & Monitoring Reference

### Quick Command Reference

| Command | What It Does |
|---------|-------------|
| `check_gpu` | Custom command to check free/used GPU memory on nodes. |
| `check_cpu_ram` | Custom command to check free/used CPU and RAM on all nodes. |
| `squeue` | See all jobs on the cluster. |
| `squeue -u $USER` | See only **your** running and pending jobs. |
| `scancel <job_id>` | Cancel/stop a specific job. |
| `scancel -u $USER` | Cancel **all** your jobs. |
| `scancel -u $USER -t PENDING` | Cancel only your **pending** (queued) jobs. |
| `sinfo` | See the live state of all nodes and partitions (which are idle, busy, etc.). |
| `sinfo -N -l` | Detailed per-node view: how many CPUs/memory are free on each. |
| `seff <job_id>` | See how efficiently a **completed** job used its resources (CPU %, memory used vs requested). |
| `sacct -u $USER` | View your job history (past completed/failed jobs). |
| `scontrol show job <job_id>` | Get all the details about a specific job (node, resources, start time, etc.). |
| `srun --jobid=<id> --overlap --pty nvidia-smi` | Peek at live GPU usage inside a running job. |
| `srun --gres=gpu_mem:1 --time=00:02:00 --partition=gpu --immediate=3 --pty watch -n 1 nvidia-smi` | Quick GPU eavesdrop — live stats with minimal resources. |

---

### Understanding Job States

When you run `squeue`, the **ST** column shows a two-letter code. Here's what each means:

| Code | State | Meaning |
|------|-------|---------|
| **PD** | Pending | Job is waiting in the queue for resources. |
| **R** | Running | Job is currently running on a compute node. |
| **CG** | Completing | Job is finishing up (some processes may still be active). |
| **CD** | Completed | Job finished successfully (exit code 0). |
| **F** | Failed | Job terminated with a non-zero exit code. |
| **CA** | Cancelled | Job was cancelled by the user or an administrator. |

---

### Cancelling Jobs

```bash
# Cancel a specific job
scancel 12345

# Cancel all your jobs
scancel -u $USER

# Cancel only your pending (queued) jobs — leaves running jobs untouched
scancel -u $USER -t PENDING
```

---

### Viewing Job History

```bash
# View your completed/failed jobs
sacct -u $USER

# Detailed view of a specific job
sacct -j 12345 --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS
```

> [!TIP]
> `MaxRSS` shows the peak memory your job actually used — compare this to what you requested with `--mem` to see if you're over- or under-requesting.

---

### Monitoring Output in Real-Time

Your `sbatch` jobs write output to the file you specified in `--output`. You can watch it live:

```bash
# Follow output as the job runs (like a live stream)
tail -f output_12345.txt

# Check just the last 50 lines
tail -50 output_12345.txt

# Follow both output and error files simultaneously
tail -f output_12345.txt error_12345.txt
```

---

### Checking Resource Usage After a Job

```bash
# Quick efficiency report — shows CPU %, memory used vs. requested
seff 12345

# Detailed breakdown
sacct -j 12345 --format=JobID,JobName,MaxRSS,Elapsed,CPUTime,TotalCPU
```

Use `seff` after every job to learn whether you're requesting too much or too little. Over-requesting wastes cluster capacity; under-requesting risks your job being killed.

---


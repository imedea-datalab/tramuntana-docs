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
| 4 | [Job Dependencies — Running Successive & Chained Jobs (Pipelines)](#4-job-dependencies--running-successive--chained-jobs-pipelines) |
| 5 | [Commands & Monitoring Reference](#5-commands--monitoring-reference) |
| 6 | [Transferring Files with `rsync` (Local ↔ Cluster & SLURM Batch Transfers)](#6-transferring-files-with-rsync-local--cluster--slurm-batch-transfers) |
| 7 | [SSH Setup & Passwordless Login (Linux, macOS, Windows)](#7-ssh-setup--passwordless-login-linux-macos-windows) |

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

### Step 3: Peek inside a running job (Monitoring Live CPU, Memory & GPU)
<a id="step-3-peek-inside-a-running-job-monitoring-live-cpu--gpu"></a>

Once your job is running, you can "peek" inside from the login node to check how many of your allocated resources (CPUs, RAM, or GPUs) your code is actually utilizing:

- **CPU & RAM jobs:** `srun --jobid=<YOUR_JOB_ID> --overlap --pty top -u $USER`
- **GPU & VRAM jobs:** `srun --jobid=<YOUR_JOB_ID> --overlap --pty nvidia-smi`
- **Interactive shell inside job:** `srun --jobid=<YOUR_JOB_ID> --overlap --pty bash`

---

#### A. Live CPU Utilization (`top`)
To see live CPU core utilization, load, and thread activity for your job's processes:

```bash
srun --jobid=<YOUR_JOB_ID> --overlap --pty top -u $USER
```
- **How `--overlap` works:** It tells SLURM to share the CPUs already allocated to your running job so you can launch `top` without requesting extra resources or queuing.
- **Reading the CPU numbers (`%CPU`):** In `top`, **100% = 1 full CPU core**. If your code is multi-threaded and using 4 cores at 100%, `top` will show `400% CPU` for your process (or separate lines showing ~`100% CPU` if using parallel worker processes). Press `q` to exit `top` anytime without interrupting your job.

#### B. Live Memory (RAM) Monitoring (`top` & `sstat`)
When checking how much RAM your code is consuming inside `top`, keep these key rules in mind:

1. **Do NOT rely on `%MEM` to determine your job's memory limit!**
   - Standard `top` calculates `%MEM` against the **entire physical node's RAM** (e.g., 1.5 TB on `thor`, 768 GB on `ada`), **NOT** against the memory you requested with `#SBATCH --mem`.
   - On a large node like `thor` (1.5 TB RAM), a process using 1.5 GB will display as only `0.1%`!
2. **Look at the `RES` (Resident Set Size) column instead:**
   - The **`RES`** column shows the **actual physical RAM** currently occupied by each process.
   - **`g` suffix** (e.g., `1,2g`): Process is using **~1.2 GB** of physical RAM.
   - **`m` suffix** (e.g., `983m`): Process is using **~983 MB** of physical RAM.
   - **Plain numbers without a suffix** (e.g., `937324`): Standard Linux `top` displays these in **Kilobytes (KiB)**. Divide by $1,024$ to get MB ($\approx 915\text{ MB}$), or by $\approx 1{,}000{,}000$ to get GB ($\approx 0.9\text{ GB}$).
   - **Pro-tip:** While in `top`, press the lowercase letter **`e`** to cycle memory display units (KiB $\to$ MiB $\to$ GiB) so every process shows up cleanly in Megabytes or Gigabytes.
3. **Remember: `top` lists memory PER PROCESS, not total job memory:**
   - If your job runs parallel tasks (e.g., in R via `mclapply`/`future` or Python `multiprocessing`), each row is one worker process.
   - If you have 30 workers each showing `1.2g` in `RES`, your job is actually using $30 \times 1.2\text{ GB} = \mathbf{36\text{ GB}}$ of RAM in total! Make sure your `#SBATCH --mem` request covers the sum of all workers plus a safety buffer.
4. **Checking total job RAM across all processes (`sstat`):**
   To see real-time peak and average memory across all processes in your running job without calculating manually:
   ```bash
   sstat -j <YOUR_JOB_ID>.batch --format=JobID,MaxRSS,AveRSS
   ```

#### C. Live GPU Utilization (`nvidia-smi`)
To see live GPU status and VRAM consumption on the node where your job is running:

```bash
srun --jobid=<YOUR_JOB_ID> --overlap --pty nvidia-smi
```

This runs `nvidia-smi` inside your existing job's allocation. You'll see output like this:

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

#### C. Hop Directly into the Compute Node with an Interactive Shell
Want full interactive terminal access on the compute node where your job is executing?
```bash
srun --jobid=<YOUR_JOB_ID> --overlap --pty bash
```
This drops you directly into an interactive shell inside your job allocation on that compute node. You can run `htop`, `nvidia-smi`, `ps aux`, or inspect temporary files. Type `exit` when you are done.

#### D. Alternative: SSH into the Compute Node
Check which node your job is running on (`squeue -u $USER` under `NODELIST`, e.g. `thor`), then SSH directly to it from `tramuntana`:
```bash
ssh thor
htop -u $USER
```
Thanks to `pam_slurm_adopt`, Slurm automatically attaches your SSH connection into your job's cgroup container.

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

**Examples:**
```bash
# Check live GPU status without stopping your job:
srun --jobid=12345 --overlap --pty nvidia-smi

# Hop directly into an interactive shell inside your running job:
srun --jobid=12345 --overlap --pty bash
```

This is the main reason you see `--overlap` in the monitoring commands above — it lets you "eavesdrop" on your own running job without interfering with it.

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

### Profiling and Sizing Job Arrays

Because SLURM applies resource directives (`--cpus-per-task`, `--mem`, `--gres=gpu_mem:...`) **per individual array task**, do not profile all array tasks at once.

**Recommended Workflow:**
1. **Profile an individual task first:** Run `tramuntana-profile` on your script as a standalone single job (or test your heaviest representative parameter).
2. **Include fallback defaults in your script:** Ensure your script runs cleanly when `$SLURM_ARRAY_TASK_ID` is not yet set by SLURM:
   ```bash
   # In Bash:
   TASK_ID=${SLURM_ARRAY_TASK_ID:-1}
   ```
   ```python
   # In Python:
   task_id = os.environ.get("SLURM_ARRAY_TASK_ID", "1")
   ```
   ```r
   # In R:
   task_id <- Sys.getenv("SLURM_ARRAY_TASK_ID", "1")
   ```
3. **Index resources into the final array script:** Take the exact resources recommended by the profiler report (e.g. `--cpus-per-task=4`, `--mem=8G`), add `#SBATCH --array=1-N` to the header, and index your parameter sweeps using `$SLURM_ARRAY_TASK_ID`.

---

## 4. Job Dependencies — Running Successive & Chained Jobs (Pipelines)

In scientific workflows, tasks rarely happen in isolation. Often your analysis requires multiple consecutive stages:
1. **Stage 1 (Data Preprocessing / Ingestion):** CPU-heavy, needs modest RAM (e.g. 4 CPUs, 16 GB RAM).
2. **Stage 2 (Model Training / Simulation):** GPU-intensive, needs a high-end GPU and substantial RAM (e.g. 1 GPU, 8 CPUs, 64 GB RAM on `thor`).
3. **Stage 3 (Evaluation, Plotting, & Archiving):** Lightweight post-processing, needs minimal resources (e.g. 2 CPUs, 8 GB RAM).

> [!WARNING]
> **Why you shouldn't bundle everything into a single `.slurm` script:**  
> If you combine all 3 stages into one script and request a GPU (`--gres=gpu_mem:...`), the cluster reserves and locks that GPU for the **entire duration** of the job — including during Stage 1 and Stage 3 when your code isn't even touching the GPU! This unnecessarily hogs limited GPU hardware on `thor` or `tramuntana-n1`, wastes your quota/priority, and prevents other researchers from running their GPU workloads.

The solution is **Job Dependencies** using SLURM's `--dependency` (or `-d`) flag. This allows you to submit separate `.slurm` scripts for each stage, instructing SLURM to hold each successive job in the queue and start it automatically only after the prerequisite job finishes.

---

### The Core Dependency Syntax

```bash
sbatch --dependency=<type>:<job_id> next_step.slurm
```

You can specify different dependency conditions depending on how you want SLURM to react if an earlier step succeeds or fails:

| Dependency Type | Behavior | Best Used For |
|---|---|---|
| `afterok:<job_id>` | Runs **only if** `<job_id>` finishes successfully (exit code `0`). | **Standard pipelines.** Step 2 runs only if Step 1 succeeds without error. |
| `afterany:<job_id>` | Runs after `<job_id>` finishes, **regardless of exit status** (success or failure). | **Cleanup & notifications.** Archiving logs, temporary scratch cleanup, or sending alerts. |
| `afternotok:<job_id>`| Runs **only if** `<job_id>` fails or crashes (non-zero exit code). | **Error handling & diagnostics.** Dumping diagnostic logs or sending failure alerts. |
| `after:<job_id>` | Runs after `<job_id>` has begun execution (or finished). | Overlapping staggered jobs or warm starts. |
| `aftercorr:<job_id>`| For **Job Arrays**: Array task $N$ of Job 2 runs only after array task $N$ of Job 1 finishes successfully. | 1-to-1 paired processing pipelines (e.g. per-sample preprocessing &rarr; per-sample training). |

---

### How to Submit Successive Jobs

#### Method A: Automated Pipeline via a Bash Script (Recommended)

Instead of manually waiting for a job ID and typing commands into your terminal, create an automated submission script (e.g., `run_pipeline.sh`) on the login node. 

By adding the `--parsable` flag to `sbatch`, SLURM returns **only the numeric Job ID** (e.g., `12345`) instead of the full `"Submitted batch job 12345"` text, making it trivial to store in bash variables:

```bash
#!/bin/bash
# ==============================================================================
# run_pipeline.sh - Automated 3-Stage Pipeline on Tramuntana
# ==============================================================================

echo "Submitting multi-stage pipeline..."

# Stage 1: CPU Preprocessing (runs on CPU partition)
JOB1=$(sbatch --parsable step1_preprocess.slurm)
echo "✅ Stage 1 (Preprocessing) submitted: Job ID $JOB1"

# Stage 2: GPU Training (runs on GPU partition, only if Stage 1 succeeds)
JOB2=$(sbatch --parsable --dependency=afterok:$JOB1 step2_train.slurm)
echo "✅ Stage 2 (Training) submitted: Job ID $JOB2 (waiting on $JOB1)"

# Stage 3: CPU Evaluation & Plots (runs only if Stage 2 succeeds)
JOB3=$(sbatch --parsable --dependency=afterok:$JOB2 step3_evaluate.slurm)
echo "✅ Stage 3 (Evaluation) submitted: Job ID $JOB3 (waiting on $JOB2)"

# Stage 4: Scratch Cleanup (runs whether training succeeded or failed)
JOB4=$(sbatch --parsable --dependency=afterany:$JOB2 step4_cleanup.slurm)
echo "✅ Stage 4 (Cleanup) submitted: Job ID $JOB4 (runs after $JOB2 completes)"

echo ""
echo "Pipeline submitted successfully! Track progress with: squeue -u \$USER"
```

To launch the entire pipeline with one command:
```bash
bash run_pipeline.sh
```

---

#### Method B: Manual Chaining in the Terminal

If you already submitted a long-running job and decide you want another job to run immediately when it finishes:

1. Check your running job's ID:
   ```bash
   squeue -u $USER
   # Suppose your running job ID is 10521
   ```
2. Submit your next job with `--dependency=afterok:10521`:
   ```bash
   sbatch --dependency=afterok:10521 step2_train.slurm
   # Output: Submitted batch job 10522
   ```

---

#### Method C: Defining Dependencies Inside the `#SBATCH` Script

You can also specify the dependency directly inside the header of your second `.slurm` file if you already know the ID of the parent job:

```bash
#!/bin/bash
#SBATCH --job-name=step2_train
#SBATCH --partition=gpu
#SBATCH --gres=gpu_mem:24
#SBATCH --dependency=afterok:10521
#SBATCH --time=12:00:00

python train.py
```

> [!TIP]
> **Dynamic is better than hardcoding:** Method A (the launcher script with `--parsable`) is strongly recommended over hardcoding IDs in your `#SBATCH` files, because you never need to manually edit file contents between runs.

---

### Advanced Chaining Patterns

#### 1. Waiting on Multiple Jobs (AND Logic)
If a consolidation step requires multiple independent preprocessing jobs to all complete successfully before running:

```bash
# Separate multiple job IDs with colons (:)
sbatch --dependency=afterok:10521:10522:10523 step2_aggregate.slurm
```
*Step 2 will remain pending until **all three** jobs (`10521`, `10522`, and `10523`) succeed.*

#### 2. Waiting on Any Job (OR Logic)
If a step should trigger as soon as **any one** of multiple alternative jobs finishes successfully:

```bash
# Separate conditions with a question mark (?)
sbatch --dependency=afterok:10521?afterok:10522 step2_notify.slurm
```

#### 3. Fan-Out / Fan-In (Array &rarr; Merge Pipeline)
A common pattern is running a single preprocessing job, fanning out into a 50-task parallel Job Array, and then running a single aggregation job once all 50 tasks finish:

```bash
#!/bin/bash
# 1. Preprocessing (single job)
PREP_ID=$(sbatch --parsable prep.slurm)

# 2. Parallel Processing (Array of 50 tasks, starts after prep succeeds)
ARRAY_ID=$(sbatch --parsable --dependency=afterok:$PREP_ID --array=1-50 process_array.slurm)

# 3. Aggregation (single job, starts only when ALL 50 array tasks succeed)
MERGE_ID=$(sbatch --parsable --dependency=afterok:$ARRAY_ID merge_results.slurm)

echo "Submitted Fan-Out/Fan-In Pipeline: Prep=$PREP_ID -> Array=$ARRAY_ID -> Merge=$MERGE_ID"
```

---

### Monitoring Successive Jobs in `squeue`

When you submit chained jobs, dependent jobs immediately show up in `squeue`, but their state will be **`PD` (Pending)** and their reason will be **`(Dependency)`**:

```text
$ squeue -u $USER
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
             10521       cpu   step1_  atiwari  R       3:45      1 ada
             10522       gpu   step2_  atiwari PD       0:00      1 (Dependency)
             10523       cpu   step3_  atiwari PD       0:00      1 (Dependency)
```

* **While Job 10521 is running:** Jobs 10522 and 10523 sit safely in the queue without consuming any CPU, RAM, or GPU resources.
* **When Job 10521 succeeds (exit code 0):** SLURM automatically transitions Job 10522 to eligible status, and it begins running as soon as GPU resources are available.
* **What happens if Job 10521 fails?**  
  If Job 10521 crashes or exits with a non-zero code, SLURM detects that the `afterok` condition will never be met. Job 10522's reason will switch to:
  ```text
  NODELIST(REASON): (DependencyNeverSatisfied)
  ```
  SLURM will cancel the dependent job and never run it. This prevents downstream jobs from corrupting files or wasting compute when prerequisites fail.

---

## 5. Commands & Monitoring Reference

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
| `srun --jobid=<id> --overlap --pty top -u $USER` | Peek at live CPU and memory (RAM) usage inside your running job. |
| `srun --jobid=<id> --overlap --pty nvidia-smi` | Peek at live GPU usage inside a running job. |
| `srun --jobid=<id> --overlap --pty bash` | Open an interactive terminal directly inside your running job allocation. |
| `srun --gres=gpu_mem:1 --time=00:02:00 --partition=gpu --immediate=3 --pty watch -n 1 nvidia-smi` | Quick GPU eavesdrop — live stats with minimal resources. |
| `sstat -j <id>.batch --format=JobID,MaxRSS,AveRSS` | Query real-time memory usage (Max/Ave RSS) across all processes in a running job. |

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

## 6. Transferring Files with `rsync` (Local ↔ Cluster & SLURM Batch Transfers)

`rsync` (Remote Sync) is the recommended tool for moving code, datasets, and simulation results between your local computer and Tramuntana. Unlike basic `scp`, `rsync` only copies files that have changed (diffs), provides a live progress bar, and automatically resumes interrupted transfers if your network connection drops.

### Recommended Flags: `-avzP`

```bash
rsync -avzP source/ destination/
```
- `-a` (**archive**): Recursively syncs files while preserving permissions, timestamps, owner info, and symlinks.
- `-v` (**verbose**): Displays filenames as they are being transferred.
- `-z` (**compress**): Compresses file data during transfer to speed up network throughput.
- `-P` (**progress & partial**): Shows a live progress bar and speed, and keeps partially transferred files so interrupted transfers can resume instead of restarting from 0%.

> [!IMPORTANT]
> **The Trailing Slash Rule (`dir/` vs `dir`):**
> - `rsync -avzP ./my_code/ user@tramuntana:~/my_code/` copies the **contents** inside `./my_code` directly into `~/my_code/`.
> - `rsync -avzP ./my_code user@tramuntana:~/` creates the folder `my_code` inside `~/`.
> - When in doubt, test your command first with `--dry-run` to see what would be transferred without making any changes!

---

### Scenario 1: Running `rsync` from your LOCAL machine (Recommended)

Running `rsync` from your personal laptop or workstation terminal (connected to IMEDEA Ethernet or VPN) is the **simplest and most common workflow**, because your laptop can directly connect to the Tramuntana login node (`10.33.0.143`).

#### A. Uploading to Tramuntana (Local → Cluster)

- **Upload to `/home` (personal code, git repos, configs — backed up nightly):**
  ```bash
  # Upload a project folder
  rsync -avzP ./my_project/ username@10.33.0.143:~/my_project/

  # If you set up ~/.ssh/config alias:
  rsync -avzP ./my_project/ tramuntana:~/my_project/
  ```

- **Upload to `/data` (large group datasets, raw inputs — NOT backed up):**
  ```bash
  # Upload directly to your research group directory on shared storage
  rsync -avzP ./raw_data/ username@10.33.0.143:/data/<your_group_name>/raw_data/
  ```

#### B. Downloading from Tramuntana (Cluster → Local)

- **Download results from `/home` to your laptop:**
  ```bash
  rsync -avzP username@10.33.0.143:~/my_project/results/ ./local_results/
  ```

- **Download large output files from `/data`:**
  ```bash
  rsync -avzP username@10.33.0.143:/data/<your_group_name>/model_output/ ./local_results/
  ```
> **💡 Easier Alternatives:**
> - **GUI:** Use the **Open OnDemand Web File Explorer** (**Files → Home Directory**). You can upload and download files directly through your web browser with a simple button click or drag-and-drop, without touching SSH at all!

---

### Scenario 2: Transferring Large Datasets on the Cluster via SLURM (The Preferred Method)

When migrating large datasets internally on the cluster—such as moving folders from compute node local drives (`/datalocal` on `thor` or `ada`, `/datafast`) to central shared storage (`/data`)—**do NOT use `cp`, `mv`, or unthrottled `rsync` directly in a terminal on the login node or compute nodes.**

> [!CAUTION]
> 🚨 **Why interactive transfers are prohibited on `/data`:**
> 1. **Cluster-Wide Network Flooding:** The central storage server (`tramuntana-nas`) serves `/data` across the network via NFS. If a user runs unthrottled `cp` or `rsync` on multi-terabyte datasets, it completely saturates the network link and storage controller. Every other researcher running `ls`, `cd`, editing files, or submitting jobs will experience severe system latency or frozen terminal sessions (uninterruptible disk sleep).
> 2. **Session Vulnerability:** If you run an interactive transfer in your terminal and your laptop sleeps, VPN drops, or SSH times out, the transfer is killed abruptly, leaving partially copied and fragmented data.
>
> **The preferred and official way is to run a SLURM batch job** that copies in the background with strict bandwidth throttling (`--bwlimit=100M`) and streams data directly to the NAS over SSH, bypassing local NFS mount bottlenecks.

---

#### 1. How to Calculate Transfer Time & Size `--time`

The official SLURM transfer script caps bandwidth at **100 MB/s** (`--bwlimit=100M`). This ensures the NAS remains fast and responsive for all other researchers while your data moves steadily in the background.

##### The Mathematical Formula:
* Speed: $100\text{ MB/s} = 360,000\text{ MB/hour} = 360\text{ GB/hour} \approx 0.36\text{ TB/hour}$.
* Daily throughput: $0.36\text{ TB/hour} \times 24\text{ hours} \approx 8.64\text{ TB/day}$.

$$\text{Estimated Time (Hours)} = \frac{\text{Dataset Size in GB}}{360} \approx \frac{\text{Dataset Size in TB}}{0.36}$$

> [!NOTE]
> **Small Files Overhead:** If your dataset contains hundreds of thousands or millions of small files (< 1 MB each, such as raw text logs, small CSVs, or image tiles), filesystem metadata and directory traversal will reduce the effective speed. **Always add a 30%–50% safety buffer** to your calculated time when setting `#SBATCH --time`.

##### Quick Reference Table:
| Dataset Size | Bandwidth Cap | Real-World Throughput | Estimated Duration | Recommended `#SBATCH --time` |
| :--- | :--- | :--- | :--- | :--- |
| **250 GB** | 100 MB/s | ~360 GB/h | ~45 minutes | `--time=02:00:00` |
| **500 GB** | 100 MB/s | ~360 GB/h | ~1.4 hours | `--time=03:00:00` |
| **1 TB** | 100 MB/s | ~360 GB/h | ~2.8 hours | `--time=06:00:00` |
| **5 TB** | 100 MB/s | ~360 GB/h | ~14 hours | `--time=1-00:00:00` (1 day) |
| **10 TB** | 100 MB/s | ~360 GB/h | ~28 hours (~1.2 days) | `--time=2-00:00:00` (2 days) |
| **25 TB** | 100 MB/s | ~360 GB/h | ~70 hours (~2.9 days) | `--time=4-00:00:00` (4 days) |
| **55 TB** | 100 MB/s | ~360 GB/h | ~153 hours (~6.4 days) | `--time=10-00:00:00` (10 days) |

---

#### 2. Pre-Transfer Cleanup Checklist

Before copying anything to `/data`, audit your directories and clean up:
* 🗑️ **Delete intermediate & scratch files:** Remove temporary checkpoint files, unneeded intermediate caches, and raw logs.
* 🗑️ **Delete regenerable outputs:** If an output can be easily recomputed with a short script, do not consume multi-terabyte central storage.
* 🗑️ **Do NOT duplicate public datasets:** Datasets such as **ERA5, CMIP6, and CMEMS** are centrally maintained in `/data/opendata/`. Delete any personal local duplicates.

---

#### 3. Where to Put Your Data in `/data`

* **Datasets under 500 GB:** Transfer them directly to your research group's directory:
  `/data/<your_group>/` (e.g., `/data/POC/`, `/data/DataLab/`, etc.).
* **Large Datasets (> 500 GB, tens of TBs):** Transfer them to the dedicated staging area:
  `/data/tmp/$USER/your-folder/` *(This folder is already pre-created on the cluster)*.

---

#### 4. Step-by-Step: Setting Up and Running the Batch Copy Job

##### Step 1: Copy the Template Script to Your Working Directory
On the login node (`tramuntana`), copy the official cluster template:
```bash
cp /data/shared/move-to-data.sbatch .
```

##### Step 2: Edit the Script
Open `move-to-data.sbatch` in your favorite editor (e.g., `nano move-to-data.sbatch`):

```bash
#!/bin/bash
# ==============================================================================
# Tramuntana: Copy a large folder into /data via SLURM Batch
# Big copies stream over SSH straight to the NAS (tramuntana-nas), not through
# the /data NFS mount: much faster, and does not slow down /data for others.
# ==============================================================================
#SBATCH --job-name=move-to-data
#SBATCH --partition=cpu
#SBATCH --nodelist=thor              # CHANGE: Node where your data currently sits (thor or ada)
#SBATCH --cpus-per-task=2
#SBATCH --mem=4G
#SBATCH --time=14-00:00:00           # CHANGE: Sized according to the time calculation table
#SBATCH --dependency=singleton       # Guarantees only one copy job with this name runs at a time
#SBATCH --output=move-to-data-%j.log

# ------------------------------------------------------------------------------
# CONFIGURE PATHS: Keep the trailing slash (/) at the end of both paths!
# ------------------------------------------------------------------------------
SRC=/datalocal/your-folder/
DST=/data/tmp/$USER/your-folder/

# ------------------------------------------------------------------------------
# EXECUTE RSYNC:
# --no-g --chmod=Dg+s : Sets group ownership to destination group for quotas
# --partial           : Resumes interrupted transfers from where they left off
# --mkpath            : Automatically creates parent directories at destination
# --stats             : Outputs total transfer size and speed in the log
# --bwlimit=100M      : Caps bandwidth to 100 MB/s to prevent NFS saturation
# -e ssh              : Streams directly over SSH to the NAS (tramuntana-nas)
# ------------------------------------------------------------------------------
rsync -a --no-g --chmod=Dg+s --partial --mkpath --stats --bwlimit=100M -e ssh "$SRC" "tramuntana-nas:$DST"
```

> [!IMPORTANT]
> **Key Directives Explained:**
> * **`--nodelist=thor` vs `--nodelist=ada`:** Set this to the node where the source data physically resides. If your data is on `ada`, change this to `--nodelist=ada`.
> * **`--dependency=singleton`:** If you submit multiple migration jobs with the same name (`#SBATCH --job-name=move-to-data`), SLURM will automatically queue them sequentially and run only **one job at a time**, preventing multiple copies from competing for the same disk I/O.
> * **The Trailing Slash (`/`):** Both `SRC` and `DST` must end with `/`. Leaving off the slash will cause `rsync` to nest folders unintentionally (e.g. `your-folder/your-folder/`).

##### Step 3: Submit the Job
```bash
sbatch move-to-data.sbatch
# Output: Submitted batch job 14210
```

##### Step 4: Monitor the Transfer Progress
You can check if the job is running with:
```bash
squeue -u $USER
```
To view the live transfer log and speed:
```bash
tail -f move-to-data-*.log
```
*(Press `Ctrl+C` to exit the log view).*

##### Step 5: Verify Transfer Completeness (Idempotency)
When the job finishes, verify that everything transferred correctly:
1. **Re-submit the exact same script:**
   ```bash
   sbatch move-to-data.sbatch
   ```
   Because `rsync` only copies modified or missing files, it will perform a fast scan. If it reports that **0 files** were copied, your data is 100% synchronized!
2. **Optional Dry-Run Checksum Verification:**
   To perform an in-depth checksum comparison between source and destination without modifying files:
   ```bash
   rsync -a -n --checksum -i -e ssh "$SRC" "tramuntana-nas:$DST"
   ```

> [!WARNING]
> **Reminder on Cluster Backups:**  
> Personal code, scripts, git repos, and configuration files stored in `/home` are backed up nightly. The bulk shared storage `/data` (including `/data/tmp/`) does **not** currently have automated off-site backups. Always ensure critical scripts and project repositories are stored in `/home` or pushed to remote Git repositories (GitHub/GitLab).

---

## 7. SSH Setup & Passwordless Login (Linux, macOS, Windows)

Connecting to Tramuntana using `ssh username@10.33.0.143` and typing your password every time works, but it quickly becomes repetitive. By setting up an **SSH host alias** in your config file and configuring **SSH keys**, you can log in, run interactive jobs, and execute file transfers with `rsync` instantly—without typing passwords or remembering IP addresses ever again.

---

### Part 1: Setting Up `~/.ssh/config` (Host Alias)

An SSH configuration file lets you create a friendly nickname (like `tramuntana`) for the cluster. Once configured, typing `ssh tramuntana` automatically fills in the IP address, your username, and your SSH key.

#### The Configuration Block

Add the following block to your local SSH configuration file:

```ssh-config
Host tramuntana
    HostName 10.33.0.143
    User <your_imedea_username>
    IdentityFile ~/.ssh/id_ed25519
```
*(Replace `<your_imedea_username>` with your actual IMEDEA username).*

#### Where is this file located?

- **On Linux and macOS:**
  The file is located at `~/.ssh/config` (inside your home directory).
  1. Open a terminal and ensure the `.ssh` directory exists with proper permissions:
     ```bash
     mkdir -p ~/.ssh && chmod 700 ~/.ssh
     ```
  2. Open or create the config file in your preferred editor:
     ```bash
     nano ~/.ssh/config
     ```
  3. Paste the configuration block above, save, and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).
  4. Ensure strict file permissions (SSH will ignore the file if permissions are too open):
     ```bash
     chmod 600 ~/.ssh/config
     ```

- **On Windows (PowerShell / Windows Terminal):**
  The file is located at `C:\Users\<YourUser>\.ssh\config`.
  1. Open PowerShell and ensure your `.ssh` directory exists:
     ```powershell
     if (!(Test-Path "$HOME\.ssh")) { New-Item -ItemType Directory -Path "$HOME\.ssh" }
     ```
  2. Open or create the config file using Notepad:
     ```powershell
     notepad $HOME\.ssh\config
     ```
  3. Paste the configuration block above into Notepad, save (`Ctrl+S`), and close Notepad.
     *(Note: Windows OpenSSH automatically expands `~/.ssh/id_ed25519` to your user profile folder).*

---

### Part 2: Passwordless Login with SSH Keys

Setting up an SSH key pair allows Tramuntana to authenticate your computer cryptographically. You will never need to type your IMEDEA password when logging in or running file transfers.

#### Step 1: Generate an SSH Key Pair (on your local computer)

Open your local terminal (or PowerShell on Windows) and run:

```bash
ssh-keygen -t ed25519
```

- When prompted to *Enter file in which to save the key*, press **Enter** to accept the default location (`~/.ssh/id_ed25519`).
- When prompted for a *passphrase*, you can press **Enter** twice for fully passwordless login, or enter a passphrase if you want extra protection on your local machine.

This generates two files in your `~/.ssh/` directory:
- `id_ed25519`: Your **private key** (Keep this safe on your laptop! Never share or upload it).
- `id_ed25519.pub`: Your **public key** (This is the key that gets copied to Tramuntana).

#### Step 2: Copy your Public Key to Tramuntana

- **On Linux & macOS:**
  Use the built-in `ssh-copy-id` command:
  ```bash
  ssh-copy-id tramuntana
  # (Or: ssh-copy-id username@10.33.0.143)
  ```
  Type your IMEDEA password one last time.

- **On Windows (PowerShell):**
  Windows does not include `ssh-copy-id` out of the box, but you can copy your public key with this single PowerShell command:
  ```powershell
  type $HOME\.ssh\id_ed25519.pub | ssh tramuntana "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
  ```
  Type your IMEDEA password one last time when prompted.

#### Test Your Connection:
```bash
ssh tramuntana
```
You should now connect directly to the Tramuntana login node without entering your password!

---

### How This Simplifies `rsync` File Transfers

With your SSH alias and passwordless keys configured, **`rsync` transfers become effortless and completely automated**. You no longer need to type long IP addresses or stop to enter passwords:

```bash
# Upload project code to /home:
rsync -avzP ./my_project/ tramuntana:~/my_project/

# Upload large dataset to /data:
rsync -avzP ./raw_data/ tramuntana:/data/<your_group_name>/raw_data/

# Download simulation outputs back to your laptop:
rsync -avzP tramuntana:~/my_project/results/ ./local_results/
```

> [!TIP]
> **Understanding `rsync` in depth:**
> For a full breakdown of how `rsync` works, the `-avzP` flags, the critical trailing slash rule (`dir/` vs `dir`), bidirectional upload/download workflows, and SLURM batch migrations, see [**Section 6: Transferring Files with `rsync` (Local ↔ Cluster & SLURM Batch Transfers)**](#6-transferring-files-with-rsync-local--cluster--slurm-batch-transfers).




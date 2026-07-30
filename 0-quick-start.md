# Quick Start & Commands Cheat Sheet

**Just looking for the commands? You're in the right place.**  
This guide contains the absolute essentials. For more detailed topics, check out:
- [**Commands In-Depth**](0b-commands-in-depth.md): Advanced SLURM arguments, Job Arrays, and GPU monitoring.
- [**Software & Environments**](environments-and-software.md): Setting up Python (`uv`, Conda) and R.
- [**Part 1: Under the Hood**](1-under-the-hood.md): A deeper explanation of *why* things work and how the cluster is architected.

---

### 📑 Table of Contents

| # | Topic |
|---|-------|
| 1. | [Connecting via SSH](#1-connecting-via-ssh) |
| 2. | [Running Jobs with SLURM](#2-running-jobs-with-slurm) |
|   | — [A. `sbatch` — Background Jobs](#a-sbatch--background-jobs-the-most-common-way) |
|   | — [B. `salloc` — Interactive Resource Reservation](#b-salloc--interactive-resource-reservation) |
|   | — [C. `srun` — Running Commands & Parallel Computing](#c-srun--running-commands--parallel-computing) |
|   | — [Which one should I use?](#which-one-should-i-use-the-decision-tree) |
| 3. | [GPU Jobs on Tramuntana](#3-gpu-jobs-on-tramuntana) |
| 4. | [Getting Set & Go — Quick Checklist](#4-getting-set--go--quick-checklist) |

---

## 1. Connecting via SSH

Depending on where you are right now, the command to connect changes slightly:

**🏢 If you are in the IMEDEA office:**
```bash
ssh your-username@tramuntana
```

**🏠 If you are outside the office (Home, Traveling):**
*(Make sure your UIB VPN is connected first!)*
```bash
ssh your-username@10.33.0.143
```
> ⚠️ **Note**: When prompted for a password, type your IMEDEA credentials. As a security feature, **nothing will appear on the screen** while you type. Just type it and press Enter.

---

## 2. Running Jobs with SLURM

> [!TIP]
> **🚀 Open OnDemand is Live!**
>
> If you just want to run interactively, you can now launch sessions like **VS Code**, **MATLAB**, or **RStudio** directly from your browser — no terminal commands needed. Just go to [https://10.33.0.143/](https://10.33.0.143/), pick your resources, and start working.
>
> Check out the [**Open OnDemand Guide**](open-ondemand.md) for full instructions, including how to fix the blank screen issue in VS Code Jupyter Notebooks.

SLURM is the cluster manager. You don't run heavy code directly on the login node — instead, you ask SLURM to run it on a compute node for you. There are three main ways to do this: `sbatch`, `salloc`, and `srun`. Each one serves a different purpose.

---

### A. `sbatch` — Background Jobs (The Most Common Way)

This is what you'll use 90% of the time. You prepare a script file (a `.slurm` file), and `sbatch` sends it off to run invisibly in the background on a compute node. You can close your laptop, go home, go to sleep — the job keeps running. When it's done, the output is saved to a text file for you to read later.

Think of it like dropping off your laundry. You hand it in, walk away, and come back later to pick up the finished result.

**1. Create a `.slurm` file (e.g., `my_job.slurm`):**

```bash
#!/bin/bash
#SBATCH --job-name=my_first_job       # A name so you can find it in the queue
#SBATCH --output=output_%j.txt        # File that will be created and where to save printed output (%j = Job ID)
#SBATCH --error=error_%j.txt          # File that will be created and where to save error messages
#SBATCH --ntasks=1                    # Number of tasks (processes) to run
#SBATCH --cpus-per-task=4             # Give your script 4 CPUs
#SBATCH --mem=16G                     # Reserve 16 GB of RAM
#SBATCH --time=04:00:00               # Maximum runtime (4 hours)
#SBATCH --partition=cpu               # Which partition (lane) to use
#SBATCH --mail-type=END,FAIL          # Email you when the job ends or fails
#SBATCH --mail-user=your-email@uib.es # Your email address

# Your actual commands go here
python my_script.py

# --- Using uv instead? ---
# uv run python my_script.py

# --- Using Conda instead? ---
# source /data/shared/miniforge3/bin/activate
# conda activate myproject
# python my_script.py
# conda deactivate
```

> [!CAUTION]
> 🚨 **CHANGE YOUR EMAIL!** Replace `your-email@uib.es` with **your actual email address**. If you forget, you won't receive any notifications about your job!

**2. Submit it:**
```bash
sbatch my_job.slurm
# Output: "Submitted batch job 12345"
```

**What each `#SBATCH` line means:**

| Parameter | What It Does |
|-----------|-------------|
| `--job-name=my_first_job` | A friendly name for your job, so you can recognize it in `squeue`. Pick something meaningful! |
| `--output=output_%j.txt` | The file where everything your script prints to the screen gets saved. `%j` is automatically replaced by the job's unique ID number. |
| `--error=error_%j.txt` | Same idea, but for error messages. If something crashes, look here first. |
| `--ntasks=1` | How many separate processes (tasks) SLURM should prepare for your job. For a simple single Python script, this is `1`. |
| `--cpus-per-task=4` | How many CPU cores each task gets. If your code uses multi-threading (e.g., NumPy, data loaders), give it more CPUs. |
| `--mem=16G` | How much RAM to reserve. If your job tries to use more than this, SLURM will kill it! Estimate generously, but don't waste. |
| `--time=04:00:00` | The maximum wall-clock time for your job (format: `HH:MM:SS`). If the job isn't done by then, SLURM terminates it. |
| `--partition=cpu` | Which partition (lane) to run on. See the partition table in [what is slurm](2-what-is-slurm.md). |
| `--mail-type=END,FAIL` | When to email you. `END` = when it finishes. `FAIL` = if it crashes. You can also add `BEGIN` to get emailed when it starts. |
| `--mail-user=...` | Where to send those email notifications. |

---

### B. `salloc` — Interactive Resource Reservation

Sometimes you don't want to submit a script and walk away — you want to sit at the keyboard and type commands live, like when you're debugging or testing something interactively. That's what `salloc` is for.

`salloc` tells SLURM: *"Hey, reserve me some resources. I want to use them manually, right now."*

```bash
salloc --cpus-per-task=4 --mem=16G --time=02:00:00
```

> [!NOTE]
> **What is a "shell"?** When you open your terminal app (like Terminal on Mac, or a PuTTY window), the program running *inside* that window is called a **shell** — it's the thing that reads what you type, interprets the command, and tells the operating system what to do. Common shells are **Bash** and **Zsh**. The terminal is just the window; the shell is the brain inside it. When we say SLURM gives you a "new shell session," it means a fresh instance of that command-reading program — ready for you to type into.

Your terminal will pause while SLURM looks for available resources. Once granted, you get a new shell session. Here's the important part: **your terminal prompt will look the same** — you're still technically on the login node. But SLURM has now officially booked resources for you on a compute node. You can then SSH into that compute node and run your code there, knowing that those resources are protected and reserved just for you.

To see which node was assigned to you:
```bash
squeue -u $USER
# Look at the NODELIST column (e.g., "thor", "ada")
```

Then SSH into that node:
```bash
ssh thor    # or whatever node was assigned
```

> ⚠️ **Warning:** If you lose your internet connection (Wi-Fi drops, laptop closes), your `salloc` session is killed instantly and your reservation is released. This is a live connection — there's no "reconnect."

---

### C. `srun` — Running Commands & Parallel Computing

`srun` is the Swiss Army knife of SLURM. It does two main things depending on context:

**Use 1: Quick one-off commands.** Want to quickly test something on a compute node without writing a whole `.slurm` file (e.g., checking GPU status)? Just run it directly:
```bash
srun --partition=gpu --gres=gpu_mem:1 nvidia-smi
```
Your terminal will freeze while SLURM waits for resources. Once a slot opens up, your command runs on the compute node and the output streams back to your screen. When it finishes, the resources are released. This is the primary use of `srun` — fire-and-forget one-off jobs.

> [!TIP]
> **Already inside an `salloc` session?** If you run `srun` from a shell that already has an active `salloc` reservation, SLURM is smart enough to detect that (via environment variables like `SLURM_JOB_ID`) and will automatically run your command on the **already-reserved** resources — no new allocation is created. This makes `srun` a convenient way to quickly fire off one-off commands on your existing reservation without needing to SSH into the node first.

**Use 2: Launching parallel tasks.** This is where `srun` really shines. If your code uses parallel computing libraries like **MPI** or **PyTorch DDP** (Distributed Data Parallel), `srun` is the matchmaker that launches your code on multiple CPUs or nodes simultaneously and gives each copy the information it needs to communicate with the others.

```bash
# Inside a .slurm file, after booking 4 tasks:
#SBATCH --ntasks=4
srun python my_parallel_script.py
```

Slurm instantly clones your script into 4 separate processes (copies) and physically places each one onto a booked CPU. They all start running at the exact same time.

Before those 4 copies even execute their first line of Python code, Slurm injects hidden environment variables into each process. This includes things like:
- `SLURM_PROCID`: Tells the specific copy its unique rank (e.g., "I am copy 0," "I am copy 1").
- `SLURM_NPROCS`: Tells the copy the total number of tasks (e.g., "There are 4 of us total").
- `SLURM_LAUNCH_NODE_IPADDR`: Tells the copies the network IP address of the main node.

This is where your multiprocessing or distributed library (like PyTorch DDP or MPI) takes over inside the script. When your code initializes the distributed backend (e.g., `init_process_group` in PyTorch), the library automatically looks at those environment variables. Yes, PyTorch DDP and MPI both have built-in support for the Slurm environment. It instantly discovers:
- How many other copies exist.
- Which specific number it is.
- Where the master node is to establish a network connection.

> [!IMPORTANT]
> **`srun` vs plain `python`:** If your `.slurm` script says `#SBATCH --ntasks=4` but you just type `python script.py`, SLURM only runs it **one time** on one CPU. The other 3 sit completely empty and wasted! You must use `srun python script.py` to actually launch 4 copies across all your booked resources.

**Use 3: Getting an interactive shell on a node (`srun --pty`).**
```bash
srun --partition=express --cpus-per-task=4 --mem=8G --pty bash
```
*(You can also use `--pty zsh` if you prefer.)*

This requests resources (like 1 node, some CPUs) and immediately drops you into a live shell on the compute node — your prompt changes to something like `user@thor:~$`. It feels exactly like SSHing into the node, but SLURM tracks the resource usage perfectly. You see `stdout`/`stderr` instantly in your terminal, and no output files are created unless you explicitly redirect them.

This is handy for quick interactive testing **in a terminal**.

> [!CAUTION]
> **What happens when you exit the shell?**
> - **Background processes will be KILLED.** If you start a process in the background (e.g., `python script.py &`) and exit the shell, Slurm will instantly kill it. It does not matter whether you pre-allocated resources with `salloc` or let `srun --pty bash` allocate them on the fly. Once the shell exits, the job step ends, and Slurm strictly cleans up all associated processes.
> - **`sbatch` jobs are SAFE.** `sbatch` behaves completely differently. If you type `sbatch my_job.slurm` inside an interactive shell, you aren't actually running the job in that shell. You are just sending a message to the Slurm controller saying, *"Please put this new, independent job into the queue."* Because it gets its own independent Job ID, it will survive even if you close your interactive shell.

> [!TIP]
> **How to keep interactive sessions alive (The `tmux` trick)**
> If you want to run an interactive shell, start a long script, and then safely disconnect your laptop without the job dying, use `tmux` on the **login node** first:
> 1. `tmux new -s my_session` *(Starts a persistent terminal window on the login node).*
> 2. `srun --pty bash` *(Requests resources and jumps to the compute node inside that window).*
> 3. `python train.py` *(Start your long-running task).*
> 4. Press **`Ctrl+B`**, let go, then press **`D`** *(Detaches the window so you can safely disconnect your SSH).*
> 5. `tmux attach -t my_session` *(Run this when you reconnect later to resume exactly where you left off).*
>
> *Note: Slurm still perfectly tracks this! You will see it running in `squeue`, and it will still respect your requested time limits.*

> [!WARNING]
> **Don't try to launch `srun`, `mpirun`, or parallel jobs from inside `srun --pty`.** While SLURM technically allows running `srun` as a "job step" within an existing allocation, it's unreliable in this context — you're constrained to the parent job's resources, environment variables can conflict, and MPI launchers often fail to detect the allocation correctly. If you need to run parallel tasks or multi-step workflows interactively, use `salloc` instead — it gives you a proper allocation from which you can safely launch `srun` and MPI commands.

> [!NOTE]
> **`srun --pty` vs `salloc` + SSH — what's the difference?**
>
> Both give you interactive access to a compute node, but they work differently:
>
> | | `srun --pty bash` | `salloc` + `ssh` |
> |---|---|---|
> | **What you get** | A terminal shell on the node. That's it. | A full resource reservation — you can SSH in and do *anything*: run VS Code remote, start Jupyter, open multiple terminals, etc. |
> | **Waiting** | You must wait in the queue. Once resources are free, you're dropped in automatically. | `salloc` books the resources first. Once the reservation is granted, you SSH in whenever you're ready — no additional wait. |
> | **Flexibility** | Single terminal session, terminal-only. | Full access: VS Code, Jupyter, multiple SSH sessions, etc. |
> | **Best for** | Quick one-off testing in a terminal. | Longer interactive/development sessions requiring richer tooling. |

**Use 4: Tracking sub-processes (Job Steps).**
Whenever you execute `srun`, Slurm creates a "Job Step" within the overarching allocation. 
- In your `.slurm` file, if you run `srun python run_colbert.py`, it creates Job Step `<jobid>.0`, and a subsequent `srun python run_vllm.py` creates Job Step `<jobid>.1`.
- (Even when you use `srun --pty bash` interactively, Slurm creates a Job Step just to run the bash terminal process. Once that terminal process exits, the Job Step is marked as completed, and Slurm terminates anything else attached to it).

Because of this, you can use `srun` multiple times within a single `.slurm` file, and Slurm will track these sub-processes individually. You can use the command `sstat -j <jobid> -a --format=JobID,AveCPU,MaxRSS` to see the exact real-time RAM and CPU usage of each specific job step.

---

### Which one should I use? (The decision tree)

| Situation | Use | Why |
|-----------|-----|-----|
| Running a long script overnight | `sbatch` | It runs in the background, survives disconnections, and saves output to a file. |
| Quick 5-minute test of a small script | `srun` (directly) | No need to write a `.slurm` file. Just fire and forget. |
| Quick terminal testing on a node | `srun --pty bash` | Drops you into a shell on the compute node instantly. |
| Debugging interactively, running things step by step | `salloc` | You get a resource reservation and can type commands live. |
| Interactive work with VS Code / Jupyter | `salloc` + SSH | Full access to tools beyond the terminal. |
| Launching parallel/distributed code (MPI, DDP) | `sbatch` + `srun` inside the script | `sbatch` for the background wrapper, `srun` for the parallel launch. |



**💡 Tip:** Before you submit your job, check if the cluster has enough available resources!
To check available CPU and RAM across the cluster:
```bash
check_cpu_ram
```

---

## 3. GPU Jobs on Tramuntana

Only three nodes have GPUs:
- **tramuntana-n1**: 1× NVIDIA L40S (48 GB VRAM)
- **thor**: 2× NVIDIA RTX 6000 Ada (48 GB VRAM each)
- **barracuda**: 1x NVIDIA GV100 (32 GB VRAM)

**💡 Tip:** To quickly check how much GPU memory is currently free across the cluster, run:
```bash
check_gpu
```

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

## 4. Getting Set & Go — Quick Checklist

Use this to make sure you haven't missed a step, from first connection to finished job.

**🔌 1. Connect**
- [ ] VPN is connected (if working from outside the office)
- [ ] SSH into tramuntana (`ssh your-username@tramuntana` or `@10.33.0.143`)

**🧭 2. Pick Your SLURM Command**
- [ ] Long / overnight job → write a `.slurm` file and use `sbatch`
- [ ] Quick one-off test → use `srun` directly
- [ ] Interactive terminal on a node → use `srun --pty bash`
- [ ] Interactive session with VS Code / Jupyter → use `salloc` + SSH
- [ ] Parallel / distributed code (MPI, DDP) → `sbatch` + `srun` inside the script

**📝 3. Prepare Your Job Script (for `sbatch`)**
- [ ] Set a meaningful `--job-name`
- [ ] Specify `--output` and `--error` files (use `%j` for job ID)
- [ ] Choose the right `--partition` (`express`, `cpu`, or `gpu`)
- [ ] Set `--cpus-per-task` and `--mem` based on your workload
- [ ] Set a realistic `--time` limit (default is only 1 hour!)
- [ ] Replace `your-email@uib.es` with **your actual email** 🚨
- [ ] Double-check all file paths in the script

**🎮 4. GPU Jobs (skip if CPU-only)**
- [ ] Use `--partition=gpu`
- [ ] Use `--gres=gpu_mem:N` (**not** `--gres=gpu:1` — that silently caps you at 12 GB)
- [ ] Check available VRAM with `check_gpu` before submitting
- [ ] Optionally use `--nodelist=thor`, `--nodelist=tramuntana-n1`, or `--nodelist=barracuda` for a specific GPU

**🚀 5. Submit & Monitor**
- [ ] Check available CPU/RAM with `check_cpu_ram` before submitting
- [ ] Submit: `sbatch my_job.slurm` — note the job ID
- [ ] Check status: `squeue -u $USER`
- [ ] Watch output live: `tail -f output_<jobid>.txt`
- [ ] For GPU jobs, peek with: `srun --jobid=<id> --overlap --pty nvidia-smi`

**📊 6. After the Job**
- [ ] Review efficiency: `seff <jobid>` — are you over/under-requesting resources?
- [ ] Check detailed history: `sacct -j <jobid> --format=JobID,JobName,MaxRSS,Elapsed,CPUTime,TotalCPU`
- [ ] Clean up unnecessary output files

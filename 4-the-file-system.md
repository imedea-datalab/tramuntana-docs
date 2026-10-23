# Part 4: The File System

You're logged in! You are now sitting on the login node (`tramuntana`). 

If you type `ls /` (which means "list everything in the root folder"), you will see two very important folders: **`home`** and **`data`**.

---
## 1. Home and Data folders

When you go inside the `/home` directory, you'll have your own folder in there (`/home/your-username`). This is your personal space. It has a specific size limit (quota) assigned to you, typically 200GB.

If you work in a big research group, the data you need for your computations is probably much bigger than 200GB. That's what the `/data` folder is for. It is the massive shared warehouse for your group's large datasets. Inside `/data` you will have your own research group's folder for example if you work in a research group called a-combs, you will have a folder `/data/a-combs`. Inside this folder you can store all your large datasets.

So in `/home` you will only store your codes, scripts and small data files that you personally need to run your jobs. In `/data` you will store all your group's large datasets. 

### Checking Quotas & Understanding Limits

You can check your research group's current storage usage in `/data` at any time from the login node (`tramuntana`) by running:
```bash
check_quota
```
This command displays an updated status report of your group's **Used Space**, **Soft Limit**, **Hard Limit**, and **Grace Period Timer**.

#### Understanding Soft vs. Hard Limits
* **Soft Limit (typically 15 TB):** An initial threshold that triggers a warning when crossed.
* **Hard Limit (typically 20 TB):** The absolute maximum storage capacity you cannot exceed.
* **The Grace Period (7 Days):** When your group's storage exceeds the Soft Limit (for example, reaching 16.5 TB), a 7-day countdown timer begins. In our HPC cluster, **when usage remains above the soft limit for longer than the 7-day grace period, the filesystem automatically locks the directory to prevent further writes**, even if you haven't hit the absolute hard limit yet!
* **Quota Extensions on Demand:** Both `/data` and `/home` quotas are **not permanent ceilings**—they can be increased upon request. If your research group or project needs additional capacity, simply contact the cluster administrators and your limit will be expanded.

To regain write access once locked, you must either delete or archive enough files to drop back below the soft limit, or contact the system administrator to request a quota extension.

But these folders (`/data` and `/home`) aren't really on the login node or compute nodes. They actually live on **completely different nodes** — the storage servers we met in [Part 1](1-under-the-hood.md):
* `/home` lives on **`migjorn`** (connected via an ultra-fast 50 GbE network).
* `/data` lives on **`tramuntana-nas`** (connected via a 10 GbE network).

When you open `/data` or `/home` on your compute node, you are looking directly at drives on those remote servers across physical network cables. Data is streamed in real time.

---

## 2. The Cable Problem & The Network Traffic Jam

Now you know that your files live on storage servers, but your computations happen on compute nodes (like `ada` or `thor`). 

This brings us to a crucial point: **Every time a compute node reads or writes a file in `/home` or `/data`, that data has to physically travel through a network cable.**

Think of the network cable as a **highway**:
* If your program reads a file once, does heavy math for an hour, and writes one small result file, this is totally fine—it puts just a couple of cars on the highway.
* But what if your program reads dozens of huge files and writes a massive combined file (e.g. running `cdo mergetime` across 50 multi-GB NetCDF files), or writes thousands of files constantly? 

This is called an **I/O heavy task** (Input/Output heavy). When an I/O-heavy program runs directly over `/data`, it acts like hundreds of semi-trucks flooding the highway at once. 

**The result? A complete traffic jam:**
1. **Your job slows down significantly**, because the network queue is flooded.
2. **The entire compute node freezes for all other users**: When another researcher runs `ls`, `cd`, uses tab-autocomplete, or tries to log in, their command gets stuck waiting in the traffic jam (entering uninterruptible sleep, or "D state").

---

## 3. Storage Performance: How the Options Compare

We ran empirical benchmarks on `ada` to measure the real write throughput of each option:

| Storage Location | Physical Location | Measured Write Speed | Time to Write 100 GB | Key Consideration |
| :--- | :--- | :--- | :--- | :--- |
| **`/data`** | Remote (`tramuntana-nas`) | **~100 – 150 MB/s (0.1 GB/s)** *(0 MB/s during jams)* | **~12 – 16 minutes** | Huge space (200 TB pool), but slowest. Heavy I/O causes cluster-wide freezes. |
| **`/home`** | Remote (`migjorn`) | **~600 MB/s (0.6 GB/s)** | **~2 minutes 45 seconds** | **6x faster** than `/data` (uses 50 GbE & NVMe). Great for scripts and medium datasets. *(200 GB quota, expandable on demand)*. |
| **Local Scratch (`/scratch`)** | Physically inside compute node | **~1,000 MB/s (1.0 GB/s)** | **~1 minute 40 seconds** | **10x faster** than `/data`! Zero network traffic. Temporary only—**must be cleaned up after jobs**. |

---

## 4. Where Should I Put My Data? (The Decision Guide)

To keep your computations fast and the cluster smooth for everyone, follow this decision tree:

### Option A: Light Reading or Small Outputs $\rightarrow$ Use `/data`
* **When to use:** Your job reads files a few times and produces modest output files.
* **Why:** You don't need to copy files back and forth; the shared network path handles normal, non-intensive reads smoothly.

### Option B: Medium Datasets with Frequent I/O $\rightarrow$ Use `/home`
* **When to use:** Your code writes checkpoints, reads/writes repeatedly, and fits within your quota.
* **Why:** `/home` writes at **600 MB/s** (6x faster than `/data`).
* **Note:** If your project needs more than the default 200 GB, contact the administrators for an on-demand quota extension.

### Option C: Heavy Merging, Big I/O, or 10+ Files $\rightarrow$ Use Local Scratch (`/scratch`)
* **When to use:** Heavy operations like `cdo mergetime`, unpacking large archives, training loops writing frequent raw shards, or processing datasets over 10 GB.
* **Why:** `ada` has **7 TB of ultra-fast local NVMe storage** (`/scratch`). It writes at **1.0 GB/s**, sits directly on the internal hardware bus, and uses **0% network bandwidth**—preventing any cluster freezes.

---

## 5. Working with Local Scratch: Best Practices & Cleanup

Because compute node local scratch drives (`/scratch`) are shared temporary workspaces with no automated quotas, **you must clean up after your job completes**.

### The Golden Rule of Local Scratch:
> ⚠️ **Always delete your temporary files from `/scratch` once your results are copied back to `/data` or `/home`.** Leaving massive files in `/scratch` will eventually fill the drive and cause other jobs on that node to fail with `No space left on device`.

### Recommended Slurm Batch Template:
Here is how to structure an I/O-heavy job (like a large `cdo` merge) safely:

```bash
#!/bin/bash
#SBATCH --job-name=fast_merge
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G

# 1. Create a unique temporary directory on the fast local NVMe drive
SCRATCH_DIR=/scratch/$USER/job_${SLURM_JOB_ID}
mkdir -p "$SCRATCH_DIR"

# 2. Run your heavy processing with output going to the LOCAL drive
# (Example: merging files from /data and saving output locally at 1.0 GB/s)
cdo -O mergetime /data/opendata/CMIP6/.../*.nc "$SCRATCH_DIR/merged_output.nc"

# 3. Transfer ONLY the final result file back to shared storage
cp "$SCRATCH_DIR/merged_output.nc" /data/my-group/results/

# 4. MANDATORY CLEANUP: Remove the temporary scratch folder
rm -rf "$SCRATCH_DIR"
```

By following this pattern:
1. Your merge finishes **10 times faster**.
2. Network traffic is kept to near zero.
3. The local scratch drive remains clean and available for everyone.

---

## Recap: What You've Learned

1. **Your files aren't on the compute node by default.** `/home` and `/data` live on dedicated storage servers (`migjorn` and `tramuntana-nas`).
2. **The Cable Problem:** Streaming heavy read/write tasks over `/data` jams the network and freezes interactive access on the compute node.
3. **Speed Hierarchy:** Local `/scratch` (~1.0 GB/s) > `/home` (~0.6 GB/s) > `/data` (~0.1 GB/s).
4. **Quotas on Demand:** Both `/data` and `/home` quotas can be extended whenever your research requires it.
5. **Clean Up Scratch:** Always remove your temporary `/scratch` folders after copying the final results back to shared storage.

---

**Next up**: In [Part 5: Your First Job](5-your-first-job.md), we'll finally put everything together and you'll write and submit your very first batch job to the cluster using SLURM.

---
**Navigation:**
🏠 [Home](README.md) | 📖 [1. Under the Hood](1-under-the-hood.md) | ⚙️ [2. What is SLURM?](2-what-is-slurm.md) | 🔌 [3. Getting Connected](3-getting-connected.md) | 📁 **4. The File System** |
---

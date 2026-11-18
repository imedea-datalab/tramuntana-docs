# Part 4: The File System

You're logged in! You are now sitting on the login node (`tramuntana`). 

If you type `ls /` (which means "list everything in the root folder"), you will see two very important folders: **`home`** and **`data`**. Since November 2026 there are two more shared folders, **`projects`** and **`archive`** (see [below](#projects-and-archive)).

---
## 1. Home and Data folders

When you go inside the `/home` directory, you'll have your own folder in there (`/home/your-username`). This is your personal space. It has a specific size limit (quota) assigned to you, typically 200GB.

If you work in a big research group, the data you need for your computations is probably much bigger than 200GB. That's what the `/data` folder is for. It is the massive shared warehouse for your group's large datasets. Inside `/data` you will have your own research group's folder for example if you work in a research group called a-combs, you will have a folder `/data/a-combs`. Inside this folder you can store all your large datasets.

So in `/home` you will only store your codes, scripts and small data files that you personally need to run your jobs. In `/data` you will store all your group's large datasets. 

### `/projects` and `/archive`

* **`/projects`** is fast shared space (NVMe, 56 TB in total) for **one research project at a time**: each project gets its own folder, for example `/projects/SEALINE`, and only the project members can open it. Each project folder has a **5 TB quota** (warning at 4.5 TB); check it with `df -h /projects/<PROJECT>`. It is **backed up every night**, like `/home`. Use it for the data a project is actively working on, especially if it reads or writes a lot or has many small files.
  * **`/projects/opendata`** is the exception: it holds public datasets that several projects use, and **every cluster user can read it** (only the Data Lab and the projects that downloaded the data can write). Look there before downloading a public dataset. For now it has the CANARI Large Ensemble (`/projects/opendata/CANARI`, with a `README.txt` that explains what each folder is and how to cite it).
* **`/archive`** is slow, large space (hard drives, 98 TB in total) for **finished data you want to keep but rarely read**, with one folder per group. It sits on its own drives, so reading or writing there does not slow down `/data`. Like `/data`, it has **no backup**.

You cannot create these folders yourself: **ask the Data Lab** for a `/projects` folder (give the project name and who needs access) or an `/archive` folder for your group.

Copying into them:
* `cp -a`, `rsync -a` and `tar` keep the permissions of the source, which can leave the other project members unable to write. After copying into `/projects`, run `chmod -R g+rwX` on what you copied.
* Moving something into `/projects` or `/archive` from `/home`, `/data` or another project is a **full copy**, not an instant `mv`. For big copies use the same Slurm template as for `/data` ([`move-to-data.sbatch`](0b-commands-in-depth.md#scenario-2-transferring-large-datasets-on-the-cluster-via-slurm-the-preferred-method)), with `DST` pointing to your `/projects` or `/archive` folder.

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
* `/home` lives on **`migjorn`** (NVMe RAID5, connected via 2 × 25 Gbps bonded = 50 Gbps shared cluster link).
* `/data` lives on **`tramuntana-nas`** (spinning HDD RAID6, connected via 2 × 10 Gbps bonded = 20 Gbps shared cluster link; each compute node connects via a 10 Gbps port).
* `/projects` lives on **`migjorn`** too, on its own NVMe RAID5, and `/archive` on **`tramuntana-nas`**, on its own HDD RAID6.

All four are available on the login node and on every compute node. `barracuda`, `vscode-node01` and `vscode-node02` reach them through the 1 Gbps office network, so big transfers are faster from `ada`, `thor`, `tramuntana-n1` or `pampero`.

When you open `/data` or `/home` on your compute node, you are looking directly at drives on those remote servers across physical network cables (mounted via NFS). Data is streamed in real time.

---

## 2. The Cable & Disk Problem: Why Heavy `/data` I/O Causes Traffic Jams

Now you know that your files live on storage servers, but your computations happen on compute nodes (like `ada` or `thor`). 

This brings us to two crucial points:
1. **Every time a compute node reads or writes a file in `/home` or `/data`, that request travels over the network (NFS) to the storage server.**
2. **A transfer is only as fast as its slowest link.** For `/data` (`tramuntana-nas`), the slowest link is not the 10 Gbps cable—it is the **spinning HDD RAID6 array**, which sustains roughly **100–150 MB/s total writes shared across all users**.

Think of `/data` and its NFS worker threads as a **shared highway and counter**:
* If your program reads a file once, does heavy math for an hour, and writes one small result file, this is totally fine—it puts just a couple of cars on the highway.
* But what if your program reads dozens of huge files and writes a massive combined file (e.g. running `cdo mergetime` across 50 multi-GB NetCDF files), or writes thousands of files constantly? 

This is called an **I/O-heavy task** (Input/Output heavy). When an I/O-heavy program runs directly over `/data`, it floods the spinning disks and NFS queues faster than the mechanical drives can write the data.

**The result? A severe bottleneck:**
1. **Your job slows down significantly**, waiting on the shared mechanical disks.
2. **Other `/data` operations slow down or stall**: When another researcher runs `ls` or reads files inside `/data`, their requests have to contend with the heavy disk load.

---

## 3. Storage Performance: How the Options Compare

We ran empirical benchmarks on the cluster under normal load to measure the real write throughput and bottlenecks of each option:

| Storage Location | Physical Location & Slowest Link | Measured Write Speed | Time for 100 GB | Time for 1 TB | Key Consideration |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Local Scratch (`/scratch`)** | Physically inside compute node (`ada` 7 TB NVMe). *Bottleneck: local NVMe drive.* | **~890 MB/s** | **~2 min** | **~19 min** | **Fastest option!** Zero network traffic and no encryption. Temporary only—**must be cleaned up after jobs**. |
| **`/home`** | Remote (`migjorn`, NVMe RAID5 over NFS). *Bottleneck: single-stream NFS round trips + RAID5 parity.* | `cp`/`dd`: **420–600 MB/s**<br>`rsync`: **~180 MB/s** | **3–4 min** (`cp`) | **30–40 min** (`cp`) | **4–5x faster** than `/data`. Great for scripts, environments, and medium datasets. *(200 GB quota, expandable on demand)*. |
| **`/data`** | Remote (`tramuntana-nas`, HDD RAID6 over NFS). *Bottleneck: spinning HDD array shared by all users.* | Sustained: **~100–150 MB/s**<br>`rsync --bwlimit=100M`: **~77 MB/s** | **11–17 min** | **2–3 hours** | Huge space (182 TB pool), but slowest. Never run heavy intermediate I/O directly on `/data`. |
| **`/projects`** | Remote (`migjorn`, its own NVMe RAID5 over NFS). | Not benchmarked separately; same kind of drives and server as `/home`. | — | — | Shared space for one project, 5 TB per project, backed up nightly. |
| **`/archive`** | Remote (`tramuntana-nas`, its own HDD RAID6 over NFS). | Not benchmarked separately; same kind of drives as `/data`. | — | — | For finished data you rarely read. No backup. |
| **Office Desktop → Cluster** | Desktop 1 Gbps NIC → campus switch → login node. *Bottleneck: 1 Gbps campus link.* | **~110 MB/s** | **~15 min** | **~2.5 hours** | Naturally capped by the 1 Gbps port (safe for `/data` without `--bwlimit`). |

---

## 4. Where Should I Put My Data? (The Decision Guide)

To keep your computations fast and the cluster smooth for everyone, follow this decision tree:

### Option A: Light Reading or Small Outputs (< 10 GB) $\rightarrow$ Use `/data`
* **When to use:** Your job reads files a few times and produces modest output files (< 10 GB).
* **Why:** You don't need to copy files back and forth; the shared network path handles normal, non-intensive reads smoothly.
* **Tip:** Moving files *within* `/data` (e.g. `mv /data/tmp/$USER/x /data/<group>/x`) is **instant** (0 MB/s disk copy) because Linux only updates the folder pointer. However, `mv` *across different mounts* (like `/scratch` → `/data`) is actually an unthrottled copy + delete!

### Option B: Medium Datasets with Frequent I/O $\rightarrow$ Use `/home`
* **When to use:** Your code writes checkpoints, reads/writes repeatedly, and fits within your quota.
* **Why:** `/home` writes at **420–600 MB/s** with `cp` (4–5x faster than `/data`).
* **Note:** If your project needs more than the default 200 GB, contact the administrators for an on-demand quota extension.

### Option C: Heavy Merging, Big I/O (> 10 GB), or Many Files $\rightarrow$ Use Local Scratch (`/scratch`)
* **When to use:** Heavy operations like `cdo mergetime`, unpacking large archives, simulation working directories, training loops writing frequent shards, or processing datasets over 10 GB.
* **Why:** `ada` has **7 TB of ultra-fast local NVMe storage** (`/scratch`). It writes at **~890 MB/s**, sits directly inside the machine, and uses **0% network bandwidth**.

### Option D: Data Shared by a Project Team $\rightarrow$ Use `/projects`
* **When to use:** Several people work on the same datasets for a research project, and the data is read or written often.
* **Why:** It is NVMe like `/home`, only the project members can open it, and it is backed up every night. Ask the Data Lab for a folder.

### Option E: Finished Data You Want to Keep $\rightarrow$ Use `/archive`
* **When to use:** Results of finished projects, raw inputs you may need again, data you are moving out of `/data/tmp`.
* **Why:** It frees space in `/data` and `/projects` without slowing `/data` down. It has no backup, so keep a second copy of anything that cannot be regenerated.

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
#SBATCH --nodelist=ada
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G

# 1. Create a unique temporary directory on the fast local NVMe drive
SCRATCH_DIR=/scratch/$USER/job_${SLURM_JOB_ID}
mkdir -p "$SCRATCH_DIR"

# 2. Run your heavy processing with output going to the LOCAL drive (~890 MB/s)
cdo -O mergetime /data/opendata/CMIP6/.../*.nc "$SCRATCH_DIR/merged_output.nc"

# 3. Transfer ONLY the final result file back to /data (throttled to leave NAS headroom)
rsync -a --partial --bwlimit=100M "$SCRATCH_DIR/merged_output.nc" /data/my-group/results/

# 4. MANDATORY CLEANUP: Remove the temporary scratch folder
rm -rf "$SCRATCH_DIR"
```

By following this pattern:
1. Your heavy I/O finishes **6–9 times faster** on local NVMe.
2. Intermediate network traffic to `/data` is kept to zero, and the final copy back is safely throttled (`--bwlimit=100M`).
3. The local scratch drive remains clean and available for everyone.

> [!TIP]
> **Need to migrate or transfer multi-terabyte datasets into `/data`?**  
> Never use interactive `cp` or unthrottled `rsync` from your terminal to copy large amounts of data to `/data`, as this floods the storage server. Always use a dedicated background SLURM transfer job with rate-limiting (`/data/shared/move-to-data.sbatch`). See the full [Large Data Migration Guide in Commands In-Depth](0b-commands-in-depth.md#scenario-2-transferring-large-datasets-on-the-cluster-via-slurm-the-preferred-method).

---

## Recap: What You've Learned

1. **Your files aren't on the compute node by default.** `/home` and `/projects` live on `migjorn`, and `/data` and `/archive` on `tramuntana-nas`.
2. **The Bottleneck:** `/data` runs on a shared spinning HDD RAID6 array (~100–150 MB/s sustained writes total across all users). Streaming heavy intermediate read/write tasks directly on `/data` saturates the disks.
3. **Speed Hierarchy:** Local `/scratch` (~890 MB/s) > `/home` (420–600 MB/s with `cp`) > `/data` (~100–150 MB/s).
4. **Quotas on Demand:** Both `/data` and `/home` quotas can be extended whenever your research requires it.
5. **`/projects` and `/archive`:** one fast, backed-up folder per project (5 TB) and slow space for finished data (no backup). Ask the Data Lab for a folder.
6. **Clean Up Scratch:** Always remove your temporary `/scratch` folders after copying the final results back to shared storage.

---

**Next up**: In [Part 5: Your First Job](5-your-first-job.md), we'll finally put everything together and you'll write and submit your very first batch job to the cluster using SLURM.

---
**Navigation:**
🏠 [Home](README.md) | 📖 [1. Under the Hood](1-under-the-hood.md) | ⚙️ [2. What is SLURM?](2-what-is-slurm.md) | 🔌 [3. Getting Connected](3-getting-connected.md) | 📁 **4. The File System** |
---

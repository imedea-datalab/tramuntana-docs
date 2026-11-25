# Part 4: The File System

You're logged in and sitting on the login node (`tramuntana`)!

If you type `ls /` (list everything in the root folder), you will see the shared storage folders where research files live on Tramuntana: **`/home`**, **`/projects`**, **`/data`**, and **`/archive`**—plus fast local **`/scratch`** when your job runs on `ada`.

---

## 1. Where Your Files Live: The Five Storage Spaces

Each folder lives on specific hardware designed for a different kind of work:

| Folder | Physical Server & Disks | Capacity & Default Quota | Nightly Backup? | What Belongs Here |
| :--- | :--- | :--- | :---: | :--- |
| **`/home/<username>`** | **`migjorn`** (NVMe SSD RAID5, `ext4`) | `47 TB` pool · **200 GB soft / 400 GB hard** per user *(expandable)* | **Yes** | Your personal code, scripts, Git repos, configs, and small files. |
| **`/projects/<project>`** | **`migjorn`** (NVMe SSD RAID5, `xfs`) | `56 TB` pool · **4.5 TB soft / 5 TB hard** per active project *(on request)* | **Yes** | Fast shared NVMe space for one research project at a time (members only), datasets read repeatedly, and software environments (`.venv`, `.pixi`) — plus shared public datasets in **`/projects/opendata/`** (e.g. `CANARI`, readable by all users). |
| **`/data/<group>`** | **`tramuntana-nas`** (Spinning HDD RAID6, `xfs`) | `182 TB` pool · **15 TB soft / 20 TB hard** per group *(expandable)* | **No** | Main shared warehouse for your research group's large datasets (plus shared public data in `/data/opendata/`). |
| **`/archive/<group>`** | **`tramuntana-nas`** (Spinning HDD RAID6, `xfs`) | `99 TB` pool · Group/project quotas | **No** | Cold/long-term storage for completed projects and rarely accessed datasets. |
| **`/scratch`** *(currently on **`ada`** only)* | **Local PCIe NVMe** inside `ada` *(0 network traffic)* | **7 TB** shared temporary space *(no quota — **must delete files after job!**)* | **No** | Fastest temporary workspace (`~890 MB/s` write, `1.5–3+ GB/s` read) for unpacking `.tar` archives, `cdo` merges, and heavy intermediate I/O. |

> [!NOTE]
> **Local `/scratch` Hardware Roadmap & `pampero` Note:**  
> Today, **only `ada` has a local NVMe `/scratch` drive (`7 TB`)**. **We will be adding dedicated local NVMe `/scratch` drives to `thor`, `tramuntana-n1`, and upcoming compute nodes once funds are available to purchase the NVMe drives.** Until then, jobs on nodes without `/scratch` should use **`/projects`** (or `/dev/shm` in RAM for small datasets, or packed shard formats — see [Reading & Writing Best Practices](data-transfers-and-storage.md#3-reading-data-where--how-to-read-by-file-size--frequency)). Note that `pampero`'s local `37 TB` `/pampero` folder uses spinning hard drives (not fast NVMe) and does not speed up `/data` or `/projects`.

---

## 2. Checking Quotas & Understanding Backups

### Checking Your Usage
From `tramuntana`, you can check quotas anytime:
```bash
quota -s                    # Check your personal /home usage and limits
df -h /projects/<project>   # Check a specific /projects folder's 5 TB project quota
check_quota                 # Check your research group's /data usage, soft/hard limits, and grace timer
```

* **Soft Limit (typically 15 TB on `/data`) vs. Hard Limit (typically 20 TB):** Crossing the Soft Limit starts a **7-day grace countdown**. If your group stays above the Soft Limit for more than 7 days, writes are locked automatically—even below the Hard Limit—until files are cleaned/archived or your quota is raised.
* **Quotas Are Expandable on Demand:** `/home`, `/projects`, and `/data` quotas are **not fixed ceilings**. Whenever your research needs more space or a new `/projects` folder, simply contact the cluster administrators.
* **What Is Backed Up (and What Is Not):** Only **`/home` and `/projects`** are backed up nightly to the dedicated backup server (`io`). **`/data`, `/archive`, and `/scratch` have no backups**—keep source code in `/home` (and Git), and avoid dumping terabytes of disposable temporary scratch garbage into `/projects` so nightly backups stay clean.

---

## 3. Why Storage Choice Matters: NVMe SSDs vs. Spinning Hard Drives

Notice a key fact from Section 1: **`/home`, `/projects`, `/data`, and `/archive` are not inside the compute nodes.** They live on two dedicated storage servers (`migjorn` and `tramuntana-nas`) connected across the server-room network via NFS:

* **`migjorn` (`/home` & `/projects`) uses solid-state NVMe flash** (`50 Gbps` shared link): No moving parts—handles **hundreds of MB/s** of reads/writes and **tens of thousands of small file operations per second** effortlessly.
* **`tramuntana-nas` (`/data` & `/archive`) uses mechanical spinning hard drives** (`20 Gbps` shared link): Huge capacity (`281 TB` combined), and great at streaming **big sequential reads (`~450–535+ MB/s` uncapped)**. However, physical disk arms must move on every seek (`~150–400` random ops/sec), and **writes from each node to `/data` and `/archive` are capped at `~32–35 MB/s`** so one heavy writer can never freeze the cluster for others.

### The Three Golden Habits
1. **Reading big files once?** Read them straight from `/data`. **Reading big files repeatedly across many epochs/jobs?** Put the active dataset on **`/projects`** or **`ada:/scratch`** (see [Reading Data Guidelines](data-transfers-and-storage.md#3-reading-data-where--how-to-read-by-file-size--frequency)).
2. **Working with 10,000+ small files?** Never read or write loose tiny files directly on `/data`, `/archive`, or `/home`—pack them into a **`.tar`** (and unpack into `ada:/scratch` or RAM `/dev/shm`) or use **chunked shards** (WebDataset, HDF5, Zarr, Parquet) on `/projects` (see [Small-File Reading Options](data-transfers-and-storage.md#b-lots-of-small-files-100001000000-files--110-mb--read-once-or-read-many-times)).
3. **Moving large data (`> 30 GB` over the network, or `> 150–270 GB` between `/data` and `/archive`)?** Follow the [`cp` vs. `rsync` Size Cutoffs](data-transfers-and-storage.md#4-writing--moving-data-cp-vs-rsync-size-cutoffs) and submit a [SLURM transfer job (`move-to-data.sbatch`)](data-transfers-and-storage.md#6-large-cluster-transfers-via-slurm-move-to-datasbatch--transferring-in-parts) instead of running multi-hour interactive copies—and **always delete temporary files from `ada:/scratch`** when your job finishes!

> [!IMPORTANT]
> **Ready to upload, transfer, or optimize job I/O?**  
> See the complete operational guide—[**Data Transfers, Reading & Storage Best Practices**](data-transfers-and-storage.md)—for measured speeds on every path, exact `cp` vs. `rsync` size cutoffs, `rsync` desktop/VPN commands, `move-to-data.sbatch`, and chunked small-file workflows.

---

## Recap: What You've Learned

1. **Five storage spaces:** `/home` (code, backed up), `/projects` (fast shared NVMe, backed up), `/data` (large group datasets, HDD), `/archive` (cold HDD storage), and `ada:/scratch` (fastest local temporary NVMe on `ada`, with more nodes getting `/scratch` as NVMe funding arrives).
2. **Quotas & backups:** Check `/home` with `quota -s` and `/data` with `check_quota`; quotas can be expanded on request, and nightly backups cover `/home` and `/projects`.
3. **Match the workload to the drive:** Keep heavy intermediate writes and small-file random I/O off spinning `/data` disks—use `ada:/scratch`, `.tar`/shard containers, or `/projects`.

---

**Next up**: Head to [**Quick Start & Commands**](0-quick-start.md) to submit your first SLURM job, or read [**Data Transfers, Reading & Storage Best Practices**](data-transfers-and-storage.md) for all file transfer commands and I/O rules!

---
**Navigation:**
🏠 [Home](README.md) | 📖 [1. Under the Hood](1-under-the-hood.md) | ⚙️ [2. What is SLURM?](2-what-is-slurm.md) | 🔌 [3. Getting Connected](3-getting-connected.md) | 📁 **4. The File System**
---

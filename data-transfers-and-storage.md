# Data Transfers, Reading & Storage Best Practices

This guide is your operational reference for **uploading/downloading data, choosing where your files should live, reading large vs. small-file datasets efficiently, and moving terabytes safely** on Tramuntana.

**Related Guides:**
- [**Part 4: The File System**](4-the-file-system.md) — Beginner overview of folders (`/home`, `/projects`, `/data`, `/archive`, `/scratch`), quotas, and backups.
- [**Quick Start & Commands**](0-quick-start.md) — Connecting via SSH and running SLURM jobs (`sbatch`, `salloc`, `srun`).
- [**SLURM Commands & Monitoring**](0b-commands-in-depth.md) — Job arrays, chained pipelines (`--dependency`), and passwordless SSH setup.
- [**Open OnDemand**](open-ondemand.md) — Drag-and-drop browser File Explorer for small transfers.

---

## 📑 Table of Contents

| # | Topic |
|---|-------|
| 1 | [How the Network & Storage Speeds Work](#1-how-the-network--storage-speeds-work) |
| 2 | [Throughput (`MB/s`) vs. IOPS (`IO/s`): Big Files vs. Small Files](#2-throughput-mbs-vs-iops-ios-big-files-vs-small-files) |
| 3 | [Reading Data: Where & How to Read (By File Size & Frequency)](#3-reading-data-where--how-to-read-by-file-size--frequency) |
| 4 | [Writing & Moving Data: `cp` vs. `rsync` Size Cutoffs](#4-writing--moving-data-cp-vs-rsync-size-cutoffs) |
| 5 | [Transferring Files Between Your Computer & Tramuntana (`rsync`)](#5-transferring-files-between-your-computer--tramuntana-rsync) |
| 6 | [Large Cluster Transfers via SLURM (`move-to-data.sbatch`) & Transferring "In Parts"](#6-large-cluster-transfers-via-slurm-move-to-datasbatch--transferring-in-parts) |

---

## 1. How the Network & Storage Speeds Work

### How Everything Is Connected: Nested Boxes, Cables & Switches

Think of the network as **boxes inside boxes**, connected by cables to **switches** *(a **switch** is simply a central multi-plug connection box that lets any machine plugged into it talk directly to any other machine plugged into that same switch)*:

```text
┌────────────────────────────── PUBLIC INTERNET (Outside) ──────────────────────────────┐
│  [ Laptop at Home / Traveling ]                                                       │
│         │                                                                             │
│         ▼  Outside internet cable / Wi-Fi + UIB VPN (~5–25 MB/s)                      │
│  ┌────────────────────────── UNIVERSITY (UIB) NETWORK ─────────────────────────────┐  │
│  │      │                                                                          │  │
│  │      ▼  University cable                                                        │  │
│  │  ┌─────────────────── IMEDEA OFFICE NETWORK (1 Gbps cables) ─────────────────┐  │  │
│  │  │   │                                                                       │  │  │
│  │  │   │    [ Office Desktop ]      [ barracuda, vscode-node01, vscode-node02 ]│  │  │
│  │  │   │     │ (1 Gbps cable)                   │ (1 Gbps cables)              │  │  │
│  │  │   └─────┼──────────────┐    ┌──────────────┘                              │  │  │
│  │  │         ▼              ▼    ▼                                             │  │  │
│  │  │      ╔══════════════════════════════╗                                     │  │  │
│  │  │      ║        OFFICE SWITCH         ║  (1 Gbps connection box: lets       │  │  │
│  │  │      ╚══════════════╦═══════════════╝   office machines reach cluster)    │  │  │
│  │  │                     │                                                     │  │  │
│  │  │                     ▼  1 Gbps front-door cable (~110 MB/s shared)         │  │  │
│  │  │  ┌──────────────── TRAMUNTANA SERVER ROOM (Cluster Box) ───────────────┐  │  │  │
│  │  │  │                  │                                                  │  │  │  │
│  │  │  │       [ tramuntana Login Node ]                                     │  │  │  │
│  │  │  │                  │                                                  │  │  │  │
│  │  │  │                  ▼                                                  │  │  │  │
│  │  │  │     ╔═══════════════════════════════════════════════════╗           │  │  │  │
│  │  │  │     ║         CLUSTER SWITCH (Central Connection Box)   ║           │  │  │  │
│  │  │  │     ║   Lets any machine plugged in here talk directly  ║           │  │  │  │
│  │  │  │     ║        to any other machine on this switch        ║           │  │  │  │
│  │  │  │     ╚═══════╦══════════════════╦══════════════════╦═════╝           │  │  │  │
│  │  │  │             │                  │                  │                 │  │  │  │
│  │  │  │     10 Gbps │          50 Gbps │          20 Gbps │                 │  │  │  │
│  │  │  │  cable each │ (two 25G cables) │ (two 10G cables) │                 │  │  │  │
│  │  │  │             ▼                  ▼                  ▼                 │  │  │  │
│  │  │  │  ┌────────────────────┐ ┌────────────┐ ┌───────────────────┐        │  │  │  │
│  │  │  │  │ Main Compute Nodes │ │  migjorn   │ │  tramuntana-nas   │        │  │  │  │
│  │  │  │  │ • ada (+7 TB local │ │ (NVMe SSD) │ │  (Spinning HDD)   │        │  │  │  │
│  │  │  │  │   NVMe /scratch)   │ │ • /projects│ │ • /data (182 TB)  │        │  │  │  │
│  │  │  │  │ • thor             │ │   (56 TB)  │ │ • /archive (99 TB)│        │  │  │  │
│  │  │  │  │ • tramuntana-n1    │ │ • /home    │ │ (35 MB/s write    │        │  │  │  │
│  │  │  │  │ • pampero          │ │   (47 TB)  │ │  cap per node)    │        │  │  │  │
│  │  │  │  └────────────────────┘ └────────────┘ └───────────────────┘        │  │  │  │
│  │  │  └─────────────────────────────────────────────────────────────────────┘  │  │  │
│  │  └───────────────────────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

Three physical rules set the speed of every operation:
1. **A transfer is only as fast as the slowest cable or disk on its path.** Inside the server room, `ada`, `thor`, `tramuntana-n1`, and `pampero` each plug into the **Cluster Switch** with a **10 Gbps cable (`~1,100 MB/s`)**, talking directly to `migjorn` (**two 25 Gbps cables = `50 Gbps` total** shared across all nodes) and `tramuntana-nas` (**two 10 Gbps cables = `20 Gbps` total** shared across all nodes). Outside the server room, office desktops, `barracuda`, and `vscode-node01/02` use **1 Gbps office cables (`~110 MB/s`)**, and home laptops come in over the internet + UIB VPN (`~5–25 MB/s`).
2. **Writes to `/data` and `/archive` are capped at `~32–35 MB/s` per node.** Because `tramuntana-nas` uses mechanical spinning RAID6 hard drives, every compute node (and `tramuntana`) enforces a kernel **`35 MB/s` (`280 Mbit/s`) write limit** to `tramuntana-nas` so no single job can freeze `/data` for other researchers.
3. **Reads everywhere—and all NVMe writes (`/projects`, `/home`, `ada:/scratch`)—are completely uncapped.**

### Measured Real-World Speeds Across Every Path

| From → To | Slowest Link (Why It Runs at This Speed) | Write Speed | Read Speed *(Uncapped)* | Write Time (100 GB / 1 TB) |
| :--- | :--- | :--- | :--- | :--- |
| **`ada` ↔ `ada:/scratch`** *(local `7 TB` NVMe)* | Local PCIe NVMe drive (zero network, no encryption) | **`~890 MB/s`** | **`~1.5–3+ GB/s`** | ~2 min / ~19 min |
| **Compute ↔ `/projects`** (`migjorn` NVMe `xfs`) | 10 Gbps NFS + NVMe RAID5 parity | **`~420–873 MB/s`** (`cp`) | **`~375–607 MB/s`** *(up to `~1,100 MB/s` multi-stream)* | ~2–4 min / ~20–40 min (`cp`) |
| **Compute ↔ `/home`** (`migjorn` NVMe `ext4`) | 10 Gbps NFS + NVMe RAID5 *(`rsync` adds CPU overhead; avoid tiny-file bursts)* | `cp`: **`420–873 MB/s`**<br>`rsync`: **`~180 MB/s`** | **`~300–609 MB/s`** | ~3–4 min / ~30–40 min (`cp`) |
| **Same-server `/home` ↔ `/projects`** | Internal NVMe copy inside `migjorn` (`cp`/`mv`) | **`~400–740 MB/s`** | *(internal copy)* | ~2.5–4 min / ~25–40 min |
| **Compute ↔ Compute** (`scp` / `rsync -e ssh`) | Single-core SSH encryption over 10 Gbps switch | **`~220 MB/s`** | **`~220 MB/s`** | ~8 min / ~1.3 h |
| **Compute or `tramuntana` → `/data` or `/archive`** | Per-node **`35 MB/s` kernel write cap** *(reads uncapped)* | **`~32–35 MB/s`** per node | **`/data`:** `~450–535 MB/s` (`852 MB/s` cached)<br>**`/archive`:** `~325–444 MB/s` | **~50 min / ~8–8.5 h** |
| **Same-NAS `/data` → `/archive`** | **`cp`:** internal NAS copy (bypasses `35 MB/s` cap)<br>**`rsync`:** crosses node cable (`35 MB/s` cap) | **`cp` big files:** **`~300–355 MB/s`** (`~3–5 MB/s` tiny files!)<br>**`rsync`:** **`~25–35 MB/s`** | *(internal NAS copy)* | `cp` big files: ~5–10 min / ~50–90 min<br>`rsync`: ~50 min / ~8–8.5 h |
| **Same-NAS `/archive` → `/data`** | **`cp`:** internal NAS copy into live shared `/data`<br>**`rsync`:** crosses node cable (`35 MB/s` cap) | **`cp` big files:** **`~140–200 MB/s`** (`~3–5 MB/s` tiny files!)<br>**`rsync`:** **`~25–35 MB/s`** | *(internal NAS copy)* | `cp` big files: ~9–12 min / ~1.5–2 h<br>`rsync`: ~50 min / ~8–8.5 h |
| **Same-filesystem `mv`** *(inside `/data`, `/archive`, `/projects`, or `/home`)* | Pointer rename only (0 bytes copied) | **Instant (`0.00 s`)** | — | **`0 s` / `0 s`** *(even for 50 TB)* |
| **Office Desktop (`1 Gbps`) ↔ Cluster** | 1 Gbps campus link (`35 MB/s` cap if writing straight to `/data`) | `/projects` or `/home`: **`~100–110 MB/s`**<br>`/data` or `/archive`: **`~31–35 MB/s`** | **`~100–110 MB/s`** | `/projects`: ~15 min / ~2.5 h<br>`/data`: ~50 min / ~8–8.5 h |
| **Home Laptop via VPN ↔ Cluster** | Home internet upload + VPN encryption + latency | **`~5–25 MB/s`** | **`~5–50 MB/s`** | 1–5.5 h / 11–55 h |

> [!IMPORTANT]
> **Local `/scratch` Availability Today & Upcoming Hardware Upgrade:**  
> Currently, **only `ada` has a dedicated local NVMe `/scratch` drive (`7.0 TB`)**. `thor`, `tramuntana-n1`, `pampero`, and `barracuda` do **not** have local NVMe `/scratch` yet (`pampero`'s local `37 TB` `/pampero` folder is spinning HDD, not NVMe, and running on `pampero` does not speed up `/data` or `/projects`). **We will be adding local NVMe `/scratch` drives to `thor`, `tramuntana-n1`, and upcoming compute nodes once funds are available to purchase the NVMe drives.** Until then, use **`/projects`** (or **`/dev/shm`** in RAM for small datasets) when not on `ada`.

---

## 2. Throughput (`MB/s`) vs. IOPS (`IO/s`): Big Files vs. Small Files

Storage systems have two completely different performance limits:

| Metric | What It Measures | `/data` (`12×HDD` RAID6) | `/archive` (`8×HDD` RAID6) | `/projects` (`migjorn` NVMe `xfs`) | `ada:/scratch` (Local NVMe) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Throughput (`MB/s`)** | Megabytes streaming sequentially *(bottleneck for big files)* | **`~450–535 MB/s`** from platters *(up to **`852+ MB/s`** with NAS RAM cache)* | **`~325–450 MB/s`** *(peak `444 MB/s`)* | **`~600+ MB/s`** single stream *(up to `~1,100 MB/s` per 10 Gbps node; 50 Gbps total)* | **`~1,500–3,000+ MB/s`** read / **`~890 MB/s`** write |
| **IOPS (`IO/s`)** | Separate file open/seek operations per second *(bottleneck for tiny or random I/O)* | **`~150–400` random IOPS** *(mechanical arms take ~5–10 ms per physical seek!)* | **`~150–350` random IOPS** | **10,000s of IOPS** *(no moving parts; handles `3,500–10,500+ IOPS` smoothly)* | **10,000s–100,000s of IOPS** *(local PCIe bus)* |

### Why Big Sequential Reads Are Fine on `/data`, While Tiny Files Stall Everyone
* **2 hours of big sequential reads (`~450–535 MB/s`): Manageable for everyone.**  
  The NAS disk scheduler (`mq-deadline`) shares sequential bandwidth fairly between readers—if two heavy jobs read big files at once, each gets `~225–250 MB/s` instead of `450–500 MB/s`, and everyone else can start reading immediately. *That is why you can read large files once directly from `/data` without copying them elsewhere first.*
* **2 hours of random or tiny-file reads/writes (`~300+ IOPS`): Hurts everyone on the cluster.**  
  Opening or seeking across tens of thousands of tiny files forces the mechanical disk heads to physically jump every 5–10 ms, pinning the drives at **`94–100%` utilization** while moving as little as **`5.5 MB/s`**. When IOPS saturate the disk heads, other users (whether opening a tiny script or starting a big file read) wait in long seek queues just to start their work. *That is why workflows with thousands of small files must pack or relocate their data.*

---

## 3. Reading Data: Where & How to Read (By File Size & Frequency)

### A. Large Sequential Files (`> 100 MB–20 GB` NetCDF, video, `.tar`, model outputs)
* **Read ONCE → Leave on `/data` (or `/archive`) and read directly.**  
  Reads are uncapped (`450–535+ MB/s`). Copying a large file to `/scratch` just to read it once still reads `/data` once anyway—zero time saved.
* **Read MANY times → Put active dataset on `/projects` or `ada:/scratch`.**  
  If your job loops over the same `500 GB–5 TB` dataset across many training epochs, parameter sweeps, or multi-node runs, keep the active copy on **`/projects/<project>/`** (works from every CPU/GPU node) or stage to **`ada:/scratch`** (fastest single-node option when running on `ada`; delete when finished).

### B. Lots of Small Files (`10,000–1,000,000+` files `< 1–10 MB`) — Read ONCE *or* Read MANY Times
Whether you scan 100,000 small files **once** or loop over them **repeatedly** (e.g., PyTorch `DataLoader` over image patches, CSVs, genomics fragments), **never read loose small files directly from `/data`, `/archive`, or `/home`**. Follow these options from fastest to fallback:

1. **Option 1 (Fastest — Dataset fits in `ada:/scratch` `7 TB` or allocated RAM `/dev/shm`): Pack into 1 `.tar` & Unpack Once**  
   Store your dataset on `/data` as **one `.tar` archive** (or `.tar.zst`). At the start of your SLURM job, stream that single `.tar` sequentially once and extract locally:
   * **On `ada`:** extract into `/scratch/$USER/job_${SLURM_JOB_ID}/` (`7 TB` NVMe).
   * **On any node (if small, e.g. `< 10–50 GB` and fits inside your requested `#SBATCH --mem`):** extract into `/dev/shm/$USER/job_${SLURM_JOB_ID}/` (pure RAM speed).
   * **Mandatory cleanup:** Always `rm -rf` your `/scratch` or `/dev/shm` folder before the job exits!
2. **Option 2 (Best When Larger Than RAM / `/scratch`, or on GPU Nodes Without `/scratch`): Chunked / Streaming Shards**  
   Pack millions of tiny files into **`~0.5–2 GB` sequential shard files** so the disk streams large contiguous blocks instead of seeking per file:
   * **Streaming `.tar` shards:** **WebDataset** (`shard-{0000..0999}.tar`) or Python `tarfile` streaming iterators.
   * **Chunked array / columnar formats:** **HDF5 / NetCDF4**, **Zarr** (sharded / consolidated metadata), **Apache Parquet**, or **LMDB**.
   * Read repeated epochs from **`/projects`** (NVMe), or stream sequential passes straight from **`/data`**.
3. **Option 3 (Zero-Code-Change Fallback): Active Copy on `/projects` (`xfs`)**  
   If legacy code requires loose files and you cannot run on `ada`, keep the active dataset on **`/projects/<project>/`** (`migjorn` NVMe `xfs` handles `10,000s` of IOPS cleanly). Pack it into `.tar` shards before moving it to `/data` or `/archive` when the project finishes.

### Summary Read Decision Matrix

| File Size & Count | Read Frequency | Best Location & Method | Why |
| :--- | :--- | :--- | :--- |
| **Big files (`> 100 MB`)** | **Read ONCE** | **Read directly from `/data` or `/archive`** | Uncapped `450–535+ MB/s` sequential throughput; shares fairly. |
| **Big files (`> 100 MB`)** | **Read MANY times** | **`/projects`** *(any node)* or **`ada:/scratch`** *(on `ada`)* | Runs on NVMe (`600–3,000+ MB/s`) and keeps `/data` platters free. |
| **Lots of small files (`10k+` files `< 1–10 MB`)** | **Once OR Many times** *(fits in `/scratch` or RAM)* | **Option 1:** Store as **1 `.tar`** on `/data` → extract once into **`ada:/scratch`** or **`/dev/shm`** | Turns 100k slow HDD seeks into 1 fast sequential read + local NVMe/RAM I/O. |
| **Lots of small files (`10k+` files `< 1–10 MB`)** | **Once OR Many times** *(larger than RAM / no `/scratch`)* | **Option 2:** Stream **`~0.5–2 GB` shards** (WebDataset, HDF5, Zarr, Parquet, LMDB) from **`/projects`**<br>**Option 3:** Active copy on **`/projects` (`xfs`)** | Eliminates mechanical HDD head thrashing and `/home` `ext4` lock stalls. |

### Example SLURM Job Using `ada:/scratch` (With Mandatory Cleanup)
```bash
#!/bin/bash
#SBATCH --job-name=fast_io_job
#SBATCH --nodelist=ada
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --time=04:00:00

SCRATCH_DIR=/scratch/$USER/job_${SLURM_JOB_ID}
mkdir -p "$SCRATCH_DIR"
trap 'rm -rf "$SCRATCH_DIR"' EXIT   # Guarantees cleanup even if the script fails!

# 1. Unpack a .tar dataset once OR write heavy intermediate merges locally (~890 MB/s):
tar -xf /data/my-group/datasets/patches.tar -C "$SCRATCH_DIR/"
uv run python train_or_merge.py --input "$SCRATCH_DIR" --output "$SCRATCH_DIR/final_result.nc"

# 2. Copy ONLY the final result back to /data at the 35 MB/s rate limit:
rsync -a --no-g --chmod=Dg+s --partial --bwlimit=35M "$SCRATCH_DIR/final_result.nc" /data/my-group/$USER/results/
```

---

## 4. Writing & Moving Data: `cp` vs. `rsync` Size Cutoffs

Size cutoffs for `cp` vs. `sbatch` + `rsync` are based on a **~15-minute maximum disk saturation / interactive window** ($\text{Cutoff} = 15\text{ min} \times 60\text{ s} \times \text{Write Speed}$):

### 1. Moving Inside the *Same* Filesystem (`mv`) — Instant at Any Size
* Moving folders *within* `/data` (e.g., `mv /data/tmp/$USER/run1 /data/<group>/$USER/run1`), *within* `/archive`, *within* `/projects`, or *within* `/home` takes **`0.00` seconds** (even for 50 TB) because Linux only updates the directory pointer.
* **Warning:** `mv` *between two different mounts* (`/scratch → /data` or `/data → /archive`) is **not** instant—it performs a full `cp` + delete, so follow Rules 2 & 3 below!

### 2. Local Same-NAS Copies (`/data` ↔ `/archive`, Both on `tramuntana-nas`)
* **Why `cp` and `rsync` behave differently here:** Running **`cp`** between `/data` and `/archive` uses NFSv4.2 server-side `COPY` directly inside `tramuntana-nas` (bypassing the `35 MB/s` network cap and driving the NAS disks at full internal speed). Running **`rsync`** routes through the compute node's **`35 MB/s` cap**, keeping the NAS disks responsive for everyone else.
* **To `/archive` (`/data` → `/archive`, `~300–350 MB/s` sequential write):** $15 \times 60 \times 300\text{ MB/s} = \mathbf{270\text{ GB}}$
  * **`≤ 250–270 GB` (large files):** `cp -r` is fine (finishes in ≤ 15 min).
  * **`> 270 GB` (or thousands of tiny files):** Submit via **`sbatch` using `rsync`** (`move-to-data.sbatch`, `--bwlimit=35M`), or pack tiny files into `.tar` first.
* **To `/data` (`/archive` → `/data`, `~150–200 MB/s` on live shared `/data`):** $15 \times 60 \times 150\text{ MB/s} = \mathbf{135\text{ GB}}$
  * **`≤ 100–150 GB` (large files):** `cp -r` is fine (finishes in ≤ 10–15 min).
  * **`> 150 GB` (or thousands of tiny files):** Submit via **`sbatch` using `rsync`** (`--bwlimit=35M`).
* **Need to move many terabytes between `/data` and `/archive` faster than `35 MB/s`?**  
  **Contact the cluster administrators!** Admins have direct access to `tramuntana-nas` and can run a server-local throttled `rsync --partial` (`~150 MB/s`) or schedule an overnight move—4–5× faster while keeping the NAS safe.

### 3. Cross-Network Writes to `/data` or `/archive` (`~32–35 MB/s` Per-Node Cap)
* **`≤ 10–30 GB` (≤ 15 min at `35 MB/s`):** Interactive `cp -r` or `rsync` is fine (`tc` automatically caps the write at `35 MB/s`).
* **`> 30 GB` / multi-TB / thousands of files:** Always submit via **`sbatch`** using `/data/shared/move-to-data.sbatch` (`rsync --partial --bwlimit=35M`). Multi-hour interactive `cp` loses progress if your Wi-Fi/SSH drops, whereas `sbatch` + `rsync --partial` resumes cleanly and preserves group quotas (`--no-g --chmod=Dg+s`).

### 4. Writing *Into* `/projects` & `ada:/scratch` (Uncapped NVMe)
* **Omit `--bwlimit=35M` when copying *into* `/projects` or `ada:/scratch`:** NVMe runs at `~420–890 MB/s` (`cp -r` is 2–4× faster than `rsync`). For large NVMe copies **`> 200–500 GB`**, wrap `cp -r` or `rsync -a --partial` (**without** `--bwlimit`) inside an `sbatch` job so SSH disconnects don't abort it and `tramuntana` stays free.
* **After copying into `/projects`, run `chmod -R g+rwX` on what you copied:** Each project member gets access by name, but `cp`, `rsync` and `tar` carry over the source's permissions, and the group bits cap what the other members can do. A file that was `rw-r--r--` at the source stays read-only for the rest of the project until you run it.
* **Do NOT dump daily throwaway scratch churn onto `/projects`:** Because **`/projects` and `/home` are backed up nightly to `io`**, writing and deleting terabytes of disposable temporary files every day wastes NVMe endurance and bloats nightly backups. Use `ada:/scratch` for pure throwaway job scratch when on `ada`, and avoid dumping `100k+` tiny files into `/home` (`ext4`).

### Master Write & Transfer Cheat Sheet

| Transfer Scenario & Size | Recommended Command | Speed & Why |
| :--- | :--- | :--- |
| **Same filesystem** (`/data/a → /data/b`, `/projects/a → /projects/b`), **any size** | `mv /data/tmp/$USER/x /data/<group>/x` | **Instant (`0.00 s`)** — metadata rename only. |
| **Cross-network → `/data` or `/archive`**, **`≤ 10–30 GB`** | `cp -r src/ /data/<group>/$USER/dst/` | **`~32–35 MB/s`** (`tc` cap; ≤ 15 min). |
| **Cross-network → `/data` or `/archive`**, **`> 30 GB` / multi-TB** | **`sbatch move-to-data.sbatch`** (`rsync --partial --bwlimit=35M`) | **`~32–35 MB/s`**, survives disconnects, sets group quota (`Dg+s`). |
| **Same-NAS `/data` → `/archive`**, **`≤ 250–270 GB`** *(big files)* | `cp -r /data/<group>/x /archive/<group>/x` | **`~300–355 MB/s`** internal NAS copy (≤ 15 min). |
| **Same-NAS `/archive` → `/data`**, **`≤ 100–150 GB`** *(big files)* | `cp -r /archive/<group>/x /data/<group>/x` | **`~140–200 MB/s`** internal NAS copy (≤ 15 min). |
| **Same-NAS `/data` ↔ `/archive`**, **`> 270 GB` (`/archive`) or `> 150 GB` (`/data`)** | **`sbatch move-to-data.sbatch`** (`--bwlimit=35M`) — **or contact admins if too slow** | Keeps live NAS responsive; admins can run faster local `rsync` on NAS. |
| **Copying *into* `/projects`**, **`> 200–500 GB`** | `sbatch` running `cp -r` or `rsync -a --partial` **(NO `--bwlimit`!)** | Full **`~420–873 MB/s`** NVMe speed safely in background. |
| **10,000+ tiny files → `/data` or `/archive`** | **Pack into `.tar` / `.tar.zst` first** on `/scratch` or `/projects`, then copy `.tar` | Avoids `~3–5 MB/s` HDD seek collapse (`80×` faster). |

> [!TIP]
> **Do more CPUs speed up file transfers?**  
> * **Yes** for multi-threaded compression (`tar -I "zstd -T8"`, `pigz -p 8`), parallel SSH streams between compute nodes, or parallel downloads/NVMe copies.  
> * **No** for `/data` or `/archive` writes—every node is capped at `35 MB/s` total to `tramuntana-nas`, and parallel HDD streams only add disk-head seeks. Request just `1–2` CPUs (`--cpus-per-task=2 --mem=4G`) for NAS transfer jobs.

---

## 5. Transferring Files Between Your Computer & Tramuntana (`rsync`)

Run `rsync` **from your own laptop or office desktop terminal** (connected to IMEDEA Ethernet or UIB VPN). Unlike `scp`, `rsync` resumes interrupted transfers (`-P`) and skips unchanged files.

```bash
rsync -avP source/ destination/      # Recommended on fast office LAN (skip -z on already-compressed .nc/.mp4/.tar.zst)
rsync -avzP source/ destination/     # Add -z over slower Wi-Fi/VPN for compressible text/CSV/code
```

> [!IMPORTANT]
> **The Trailing Slash Rule (`dir/` vs `dir`):**
> * `rsync -avP ./my_code/ tramuntana:~/my_code/` copies the **contents** inside `my_code` into `~/my_code/`.
> * `rsync -avP ./my_code tramuntana:~/` creates an extra `~/my_code` folder. When unsure, test with `--dry-run`!

### Uploading & Downloading Examples
*(Tip: Set up an SSH alias and key via [SSH Setup & Passwordless Login](0b-commands-in-depth.md#6-ssh-setup--passwordless-login-linux-macos-windows) so `tramuntana:` works without typing IPs or passwords; otherwise replace `tramuntana:` with `username@10.33.0.143:`.)*

```bash
# 1. Upload code/scripts to /home (~100-110 MB/s from office LAN; backed up nightly):
rsync -avzP ./my_project/ tramuntana:~/my_project/

# 2. Upload active project dataset to /projects (~100-110 MB/s from office LAN; NVMe, backed up nightly):
rsync -avP ./active_dataset/ tramuntana:/projects/<your_project>/active_dataset/

# 3. Upload bulk dataset straight to /data (auto-capped at ~31-35 MB/s by tramuntana; NOT backed up):
rsync -avP ./raw_data/ tramuntana:/data/<your_group>/$USER/raw_data/

# 4. Download results from /data or /projects to your computer (~100-110 MB/s on office LAN):
rsync -avP tramuntana:/data/<your_group>/$USER/results/ ./local_results/
```
* **Prefer a graphical browser tool for small files?** Use the [**Open OnDemand Web File Explorer**](open-ondemand.md#-web-file-explorer-managing-files-in-home-projects-data-and-archive) (**Files → Home Directory**).

---

## 6. Large Cluster Transfers via SLURM (`move-to-data.sbatch`) & Transferring "In Parts"

Never run multi-hour cluster copies or large internet downloads (`wget`/`curl`/`rclone`) directly in an interactive terminal on `tramuntana`. Submit them as a background SLURM job instead.

### 6.1 Pre-Transfer Cleanup Checklist
* **Do NOT duplicate public datasets:** Check **`/data/opendata/`** (**ERA5, CMIP6, CMEMS**) and **`/projects/opendata/`** (e.g. **CANARI** Large Ensemble in `/projects/opendata/CANARI`, readable by all users) before downloading public data.
* **Check `/projects` space when needed:** Run `df -h /projects/<your_project>` to view your project folder's `5 TB` quota (`check_quota` for `/data`, `quota -s` for `/home`).
* **Delete intermediate scratch & regenerable files** before archiving to `/data` or `/archive`.
* **Pack tiny files first:** If a folder has `10,000+` small files, pack it into `.tar` / `.tar.zst` chunks before copying to `/data` or `/archive`.

### 6.2 How Long Will a `/data` or `/archive` Transfer Take? (`35 MB/s` Rate Limit)
At `35 MB/s` (`~120 GB/hour` ≈ **`~2.8 TB/day`**):

$$\text{Estimated Hours} \;\approx\; \frac{\text{Dataset Size in GB}}{120}$$

| Dataset Size | Per-Node Write Cap | Effective Rate | Estimated Duration | Suggested `#SBATCH --time` |
| :--- | :--- | :--- | :--- | :--- |
| **100 GB** | `35 MB/s` | ~120 GB/h | ~50 minutes | `--time=02:00:00` |
| **500 GB** | `35 MB/s` | ~120 GB/h | ~4.2 hours | `--time=08:00:00` |
| **1 TB** | `35 MB/s` | ~120 GB/h | ~8.5 hours | `--time=16:00:00` |
| **5 TB** | `35 MB/s` | ~120 GB/h | ~42 hours (~1.8 days) | `--time=3-00:00:00` |
| **10+ TB** | `35 MB/s` | ~120 GB/h | ~3.5+ days | Split in parts **or contact admins** |

### 6.3 Step-by-Step: Running `move-to-data.sbatch`
1. Copy the template on `tramuntana`:
   ```bash
   cp /data/shared/move-to-data.sbatch .
   ```
2. Edit `SRC`, `DST`, `--time`, and (if copying from `ada:/scratch`) `--nodelist=ada`:
   ```bash
   #!/bin/bash
   #SBATCH --job-name=move-to-data
   #SBATCH --nodelist=ada               # Needed only if SRC is on ada:/scratch
   #SBATCH --cpus-per-task=2
   #SBATCH --mem=4G
   #SBATCH --time=14-00:00:00
   #SBATCH --dependency=singleton       # Runs one move-to-data job at a time
   #SBATCH --output=move-to-data-%j.log

   # Keep the trailing '/' on BOTH paths!
   SRC=/projects/your-project/your-folder/
   DST=/data/your-group/$USER/your-folder/

   # --no-g --chmod=Dg+s: files inherit destination group quota.
   # --partial: resumes interrupted large files instead of restarting from 0%.
   rsync -a --no-g --chmod=Dg+s --partial --mkpath --stats --bwlimit=35M "$SRC" "$DST"

   # Optional dry-run checksum verification afterwards:
   # rsync -a -n --checksum -i "$SRC" "$DST"
   ```
   *(Note: For very large multi-TB transfers into `/data`, you can stage into `/data/tmp/$USER/your-folder/` first and then `mv` instantly into `/data/<your_group>/...` once verified.)*
3. Submit and monitor:
   ```bash
   sbatch move-to-data.sbatch
   tail -f move-to-data-*.log
   ```

### 6.4 Transferring & Processing Multi-TB Datasets "In Parts"
When a dataset is huge (`5–30 TB`), contains hundreds of thousands of small files, or exceeds `ada:/scratch` (`7 TB`) / `/projects` (`5 TB` quota), work **in parts**:
1. **Pack tiny files into `1–10 GB` `.tar` shards before transferring:** Packing subfolders on `/scratch` or `/projects` (`tar -I "zstd -T4" -cf batch_01.tar.zst batch_01/`) lifts `/data` and `/archive` write efficiency from `~3–5 MB/s` to full sequential speed.
2. **Transfer subfolder-by-subfolder (e.g., year-by-year or campaign-by-campaign):** Submitting per-subfolder jobs with `#SBATCH --dependency=singleton` checkpoints each completed slice cleanly and lets `rsync --partial` resume effortlessly if interrupted.
3. **Sliding-window staging on `ada:/scratch` or `/projects`:** To run multi-pass compute over a `20 TB` `/data` dataset, stage `1–2 TB` chunks at a time to `ada:/scratch` (or `/projects`), run all passes at NVMe speed, write reduced outputs back, `rm -rf` the chunk, and stage the next slice.

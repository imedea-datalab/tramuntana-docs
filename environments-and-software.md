# Software & Environments

This guide covers everything you need to know about setting up Python (`uv` & Conda), R, MATLAB, Pixi (system-level libraries without sudo), and Apptainer (custom containers) on the Tramuntana cluster.

**Other Practical Guides:**
- [**Quick Start & Commands**](0-quick-start.md) — Connecting to the cluster and running your first job.
- [**Commands In-Depth**](0b-commands-in-depth.md) — Advanced SLURM options, Job Arrays, and GPU monitoring.

---

## Table of Contents

1. [Python](#python)
2. [Using R for Statistical Computing](#using-r-for-statistical-computing)
3. [Using MATLAB on the Cluster](#using-matlab-on-the-cluster)
4. [Pixi: Adding System-Level Libraries & Compilers (Without Sudo)](#pixi-adding-system-level-libraries--compilers-without-sudo)
5. [Apptainer: Custom Containers (When Nothing Else Works)](#apptainer-custom-containers-when-nothing-else-works)

---

## Python

Before running your Python code on the cluster, you need a place to put all the tools and libraries your code needs. Think of an **environment** like a dedicated sandbox for your project. If Project A needs an older version of a library, and Project B needs a newer version, you don't want them fighting. Environments keep everything neatly separated so the sand from one castle doesn't mix with another.

We offer two main ways to manage your Python environments: **uv** (highly recommended) and **Conda**.

---

### Part 1: Using `uv` (⭐️ Highly Recommended)

We strongly recommend using `uv` for managing your Python projects. Why? Because it's insanely fast, incredibly simple, and it automatically manages both your Python versions and your packages without you having to think too much about it. 

Instead of manually creating a sandbox, activating it, installing things, and then deactivating it, `uv` handles all the building blocks for you behind the scenes.

#### How `uv` Handles Python Runtimes
Under the hood, `uv` is not just a package manager—it is a complete tool for managing **Python interpreters**. It can download, install, and manage multiple Python versions (e.g., Python 3.9, 3.10, 3.11, 3.12) simultaneously and completely in isolation, without interfering with your system's default Python or other users on the cluster.

When a project is created, you will notice two separate places that mention Python versions: `pyproject.toml` and `.python-version`. It is easy to get confused about why we need both, but they serve two fundamentally different purposes:

1. **`pyproject.toml` (`requires-python`) — The Public Declaration of Compatibility**
   * **What it is**: Under the `[project]` section of `pyproject.toml`, you define a general range of Python versions your code is compatible with (e.g., `requires-python = ">=3.10"`).
   * **Its purpose**: This tells the world (and `uv`) the *rules* of what versions of Python your project can safely run on. It is a public contract.
   
2. **`.python-version` — The Local Choice of Execution (The Pin)**
   * **What it is**: A simple, single-line text file containing an exact Python version (e.g., `3.11.4`).
   * **Its purpose**: This is your local choice of interpreter. Since your project supports a broad range (like `>=3.10`), `uv` needs to know *which exact version* it should spin up for your current local workbench right now.
   * **The Pinned Version**: When you run `uv python pin 3.11`, you are telling `uv`: *"Even though this project officially supports any version >=3.10, I want to use exactly Python 3.11 for my local virtual environment and runs."* This ensures absolute consistency across the team's working environments without breaking the broader project compatibility rules.


#### Where to Run `uv` Commands
> [!IMPORTANT]
> **Always run your `uv` commands from inside your project directory!**
> Since `uv` is project-centric, it relies on files in your current folder to know what to do. Before running commands like `uv add`, `uv sync`, or `uv run`, ensure your terminal is in the project folder (the one containing `pyproject.toml`).

#### Step 1: Start a New Project
First, create a folder for your work and go inside it:
```bash
mkdir my_awesome_project
cd my_awesome_project
```

Then, tell `uv` to initialize a new project here:
```bash
uv init
```
This creates a few key files to keep track of your project structure:
*   **`pyproject.toml`**: The main configuration file for your project. It lists your project's metadata, required Python version, and all your package dependencies. You can open and edit this file manually to add, remove, or modify package requirements.
*   **`uv.lock`**: An automatically generated lockfile that records the exact version of every single package and dependency installed. This ensures perfect reproducibility—anyone else running your project will get the exact same environment.
*   **`.python-version`**: A text file telling `uv` which Python version to use for this project.
*   **`hello.py`**: A simple template script to get you started.

##### Specifying or Changing the Python Version
If your project requires a specific Python version (e.g., Python 3.11), `uv` makes it incredibly easy to configure:

*   **Initialize with a specific version**: When starting a new project, you can specify the Python version upfront:
    ```bash
    uv init --python 3.11
    ```
*   **Pin a version in an existing project**: If the project has already been initialized, you can pin a specific version:
    ```bash
    uv python pin 3.11
    ```
    This updates the `.python-version` file and adjusts your project configuration.
*   **What if that Python version is not installed?** Don't worry! You don't have to manually download it. `uv` handles it all under the hood. When you run your next `uv` command (like `uv sync` or `uv run`), `uv` will detect that Python 3.11 is missing, automatically download it, and install it purely inside its own internal cache. You can also trigger the download manually using:
    ```bash
    uv python install 3.11
    ```
*   **List installed Python versions**: To see what Python versions `uv` currently has installed and available on your system, run:
    ```bash
    uv python list
    ```
*   **`.python-version` vs. `pyproject.toml` (Which one do I need?)**:
    *   **`pyproject.toml` (specifically `requires-python`)**: Declares the *compatibility range* your project supports (e.g., `requires-python = ">=3.11"`). This is standard Python packaging metadata.
    *   **`.python-version`**: Pins the *exact, single version* you want to use locally for development (e.g., `3.11.4`). This file is widely used by tools like `uv`, `pyenv`, and `asdf`.
    *   *Can I just use one?* Yes! If you delete the `.python-version` file entirely, `uv` will fall back to using the constraint in `pyproject.toml` (choosing the latest available Python version that fits). However, keeping both is highly recommended in collaborative projects so that everyone on your team is guaranteed to run the exact same Python version locally.


#### Step 2: Add Packages (The Two Types of Adding)
When managing packages in your project, you have two different ways to install them:

##### 1. The Modern Way: `uv add` (Recommended)
This is the standard, modern project-centric method.
```bash
uv add numpy pandas matplotlib
```
*   **What it does**: It automatically registers the packages in your `pyproject.toml` under `dependencies`, updates your `uv.lock` file, and installs them into your environment.
*   **Why use it**: It ensures your project's dependencies are officially documented. If you move your code or share it with someone else, they can recreate your exact environment instantly.

##### 2. The Legacy Way: `uv pip install`
This acts as a drop-in replacement for standard `pip`.
```bash
uv pip install numpy pandas matplotlib
```
*   **What it does**: It directly installs the packages into the active environment without updating `pyproject.toml` or `uv.lock`.
*   **Why use it**: Use this for quick testing, one-off scripts, or when adapting legacy code instructions. For permanent projects, stick to `uv add` so your environment remains reproducible.

#### Where Do the Packages Live?
All of your project's packages and the Python interpreter itself live inside a local folder called **`.venv`** created right inside your project directory.
*   The exact path to the installed libraries (the site-packages) is:
    ```
    .venv/lib/python3.X/site-packages/
    ```
    *(where `3.X` is the version of Python your project is using)*
*   **Easy Clean Up**: Because all packages are self-contained inside your project's `.venv` directory, there is no global pollution. If your environment ever gets broken or corrupted, you can safely delete the `.venv` folder completely and let `uv` recreate it.

#### How `uv sync` Works
`uv sync` is the command that forces your local `.venv` environment to match your `pyproject.toml` and `uv.lock` files exactly.
```bash
uv sync
```
*   **What it does**: It reads the lockfile, downloads and installs any missing packages, and **deletes** any extra packages from `.venv` that aren't defined in your project files. This keeps your sandbox perfectly clean and consistent.
*   **Pro Tip**: You rarely need to run `uv sync` manually! Whenever you run `uv add` or `uv run`, `uv` will automatically run a sync in the background if it detects that your `.venv` is missing or out of date.

#### Step 3: Run Your Code
Here is the best part. You don't need to "activate" the environment. Whenever you want to run your Python script using the sandbox you just built, simply put `uv run` in front of your command:

```bash
uv run python my_script.py
```
`uv` is smart enough to say: *"Oh, they want to run this script. Let me briefly jump into their sandbox, run the code, and then jump back out."* It's clean, safe, and prevents you from accidentally leaving an environment activated!

#### Step 4: Running the Same Environment from Another Folder
Normally, you run `uv` commands while inside your project folder. However, on the cluster, you might store your project code in your home directory (`/home/username/my_project`) while running jobs or accessing datasets located in a data directory (`/data/YourGroup/...`), or from an ad-hoc directory.

If you need to execute code from a different folder while targeting your project's environment, you have three options:

##### Option A: Direct Path (Interpreter Direct Execution)
You can directly call the Python interpreter living inside your project's `.venv`:
*   **Linux / macOS (Cluster)**:
    ```bash
    /path/to/project/.venv/bin/python script.py
    ```
*   **Windows (Local PC)**:
    ```bash
    C:\path\to\project\.venv\Scripts\python.exe script.py
    ```
*   **Why use it**: Ideal inside SLURM batch job scripts when you want a straightforward, direct path to the exact Python binary without invoking `uv`.

##### Option B: Using the `--project` Flag (⭐️ Recommended)
Tell `uv` which project directory to target using `--project`:
```bash
uv run --project /path/to/project python script.py
```
*   **Why use it**: `uv` automatically locates the project's `.venv`, validates that dependencies and locks are up to date, and runs your script seamlessly regardless of your current working directory.

##### Option C: Environment Variable (`VIRTUAL_ENV`)
Set the `VIRTUAL_ENV` variable to point to your project's `.venv`:
```bash
VIRTUAL_ENV=/path/to/project/.venv uv run python script.py
```
Or export it in your shell session or SLURM script:
```bash
export VIRTUAL_ENV=/path/to/project/.venv
uv run python script.py
```
*   **Why use it**: Convenient if you are running multiple commands in a row from another folder and don't want to append `--project` every time.

#### Step 5: Working with `requirements.txt` (Migration & Sharing)
If you are migrating an older project (such as from Conda or a standard `pip` workflow) or need to share dependencies with collaborators and deployment systems that require standard files:

##### 1. Getting `requirements.txt` from an Existing Conda Environment
If you are currently using a Conda environment and want to migrate to `uv`, export its packages using `pip`:

```bash
# 1. Activate your Conda environment
conda activate my_conda_env

# 2. Export installed packages in pip-compatible format
pip list --format=freeze > requirements.txt
# (or: pip freeze > requirements.txt)
```

> [!TIP]
> **Why export with `pip` instead of `conda list --export`?**
> Conda's native export (`conda list --export`) includes Conda-specific build strings (e.g. `numpy=1.26.0=py311h...`) that standard Python tools cannot parse. Using `pip list --format=freeze` or `pip freeze` ensures the package names and versions follow standard PyPI format, making them 100% compatible with `uv`.

##### 2. Importing into Your Project (Permanent)
To import packages from `requirements.txt` into your `uv` project permanently:
```bash
uv add -r requirements.txt
```
*   **Reads all packages** from `requirements.txt`.
*   **Formally declares them** in your `pyproject.toml` and updates `uv.lock`.
*   **Installs them** directly into `.venv/`.

##### 3. One-Time Install (Without Altering `pyproject.toml`)
If you just want to install packages directly into `.venv` without modifying `pyproject.toml` or `uv.lock`:
```bash
uv pip install -r requirements.txt
```
*   Installs packages directly into `.venv/`, serving as a lightning-fast drop-in replacement for classic `pip install -r requirements.txt`.
*   Useful for quick testing, one-off scripts, or legacy instructions.

##### 4. Exporting Back to `requirements.txt`
If you need to share your environment with someone not using `uv`, or for deployment (like Docker containers or legacy CI/CD pipelines):
```bash
uv export --format requirements-txt -o requirements.txt
```
*   Generates a frozen, locked `requirements.txt` from your `uv.lock` file, guaranteeing exact reproducible versions.
*   **Pro Tip**: If you want a clean requirements file without package hashes, add `--no-hashes`:
    ```bash
    uv export --format requirements-txt --no-hashes -o requirements.txt
    ```

---

### Part 2: Using Conda (Legacy / Alternative)

If you have an existing workflow that relies heavily on Conda, or if you need specific non-Python scientific packages that only Conda provides, you can still use it.

#### What is Conda?
Conda is an older, widely used package manager. It works well, but it can be slower and requires you to manually "activate" and "deactivate" your environments.

#### Step 1: Initialize Conda
The first time you use conda on the cluster, you need to initialize it:

```bash
# Initialize conda for your shell
/data/shared/miniforge3/bin/conda init bash

# Reload your shell configuration
source ~/.bashrc
```
You should now see `(base)` at the beginning of your prompt.

#### Step 2: Create Your First Environment
```bash
# Create a new environment named 'myproject' with Python 3.11
conda create -n myproject python=3.11

# Activate the environment
conda activate myproject
```

#### Step 3: Install Packages
```bash
# Install packages using conda
conda install numpy pandas matplotlib

# Or install using pip (if not available in conda)
pip install some-package
```

#### Step 4: Save Your Environment
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

#### Managing Conda Environments
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

#### Saving Disk Space with Conda
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

### Overview & Container Architecture

On Tramuntana, R is deployed using standardized, high-performance **Apptainer containers** (just like MATLAB). The container encapsulates the R runtime, standard libraries, and compiled C/Fortran math backends into an isolated, read-only image (`.sif`) stored on the shared cluster filesystem.

To provide a seamless native experience, we provide transparent command wrappers (such as `R-4.6`, and `Rscript-4.6`). You do not need to learn complex Apptainer commands or configure directory bind mounts—running these commands works like a standard Linux installation while automatically routing through the unified cluster container.

> [!WARNING]
> #### ⚠️ Do Not Build Your Own Standalone R Container Images
> While Apptainer allows building custom containers with `--fakeroot`, we strongly discourage users from creating independent base R containers:
> 1. **Storage Footprint:** Full R containers with GIS/system libraries can be 5–15 GB each. Duplicating them across multiple user home folders rapidly exhausts cluster disk quota.
> 2. **Library Fragmentation:** Compiling personal system-level libraries often leads to broken dependencies between compute nodes.
>
> **What should you do instead?**
> Standard R packages (from CRAN, Bioconductor, and GitHub) install into your personal user directory with zero admin privileges. If you need a specialized system-level library or a new R release, **simply contact cluster administrators**—we add them centrally so they work everywhere immediately.

---

### Understanding the Tools: `RStudio Server` vs `R` vs `Rscript`

R on Tramuntana can be used in three distinct execution modes. Choose the right tool for your workflow:

| Tool | Interface | Best For | Typical Use Cases | How to Launch |
| :--- | :--- | :--- | :--- | :--- |
| **RStudio Server** | Web GUI (Browser) | **Interactive Development & Exploration** | Writing code, plotting graphs, debugging R scripts, running Quarto / R Markdown documents, inspecting data frames. | **Open OnDemand** $\rightarrow$ Interactive Apps $\rightarrow$ RStudio Server |
| **`R`** | CLI Interactive REPL (Terminal) | **Fast Interactive Testing & Prototyping** | Quick calculations, testing functions interactively in an SSH session or inside an interactive `salloc` compute allocation. | In terminal: `R-4.6` |
| **`Rscript`** | CLI Headless / Batch (Terminal) | **Production Pipelines & SLURM Jobs** | Automated batch runs, scheduled cron jobs, parameter sweeps, and headless HPC workloads without GUI overhead. | In batch script: `Rscript-4.6 analysis.R` |

---

### Available Versions & How to Launch R

Multiple versions of R can coexist on the cluster without interfering with each other. Current supported versions include:

* **`R 4.6` (Open OnDemand & CLI):** The latest release environment.
* **Future Versions:** When new versions (e.g., R 4.7) are released, administrators deploy a new image (`r_4.7.sif`) centrally. It immediately appears as a selectable option in Open OnDemand and as `R-4.7` / `Rscript-4.7` in the CLI without breaking existing user scripts.

---

#### 1. Web Portal via Open OnDemand (Interactive RStudio Server)

The easiest and most interactive way to use R is through the **Open OnDemand** web portal:

1. Open the Open OnDemand web portal in your browser.
2. In the top navigation bar, go to **Interactive Apps** $\rightarrow$ **RStudio Server**.
3. In the form:
   - **R Version:** Choose your preferred version (e.g., `RStudio 4.6`).
   - **Cores & Memory:** Request the CPU cores and RAM needed for your analysis (e.g., 4 cores, 16 GB).
   - **Time:** Set your session duration.
4. Click **Launch**. Once SLURM assigns a compute node, click **Connect to RStudio Server**.
5. Your browser opens a full RStudio IDE session connected directly to the compute node.

---

#### 2. Command Line Interface (CLI) Execution

When connected via SSH or inside an interactive compute session (`salloc`), you have two CLI options:

##### A. Interactive Terminal Console (`R`)
To launch an interactive R session directly in your terminal:
```bash
# Launch interactive R 4.6
R-4.6
```
This opens the familiar `> ` R prompt. To quit the interactive session, type `q()` and hit Enter.

##### B. Non-Interactive Script Execution (`Rscript`)
To run an R script file directly from the command line:
```bash
# Run script using R 4.6
Rscript-4.6 my_analysis.R

# Pass command line arguments to an R script
Rscript-4.6 my_analysis.R input_data.csv --verbose
```

---

#### 3. SLURM Batch Job Script Example

For long-running analyses, submit a headless batch job to SLURM using `Rscript`:

```bash
#!/bin/bash
#SBATCH --job-name=r-simulation
#SBATCH --output=r-output-%j.txt
#SBATCH --error=r-error-%j.err
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --partition=cpu

echo "Starting R batch job on $(hostname)..."

# Run R analysis non-interactively using the containerized wrapper
Rscript-4.6 my_analysis.R

echo "Job finished with exit code $?"
```

---

#### 4. SLURM Job Arrays (Parameter Sweeps & Parallel Tasks)

When you need to process multiple data files or run simulations across different parameter sets, use SLURM Job Arrays:

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

echo "Processing task $SLURM_ARRAY_TASK_ID on $(hostname)..."

# Pass the task ID directly into your R script
Rscript-4.6 process_sample.R "$SLURM_ARRAY_TASK_ID"
```

Inside your `process_sample.R` script, retrieve the parameter using `commandArgs()`:
```r
args <- commandArgs(trailingOnly = TRUE)
task_id <- as.integer(args[1])
cat("Running analysis for sample ID:", task_id, "\n")
```

---

### Installing Packages & Tools: User vs. Administrator

Because R runs inside a standardized container, the container's root filesystem is mounted **strictly read-only (`ro`)**. You cannot write directly to `/usr/local/lib/R/site-library`.

Package and library management is clearly divided:

| Category | Examples | Can Users Install It? | Installation Method | Storage Location |
| :--- | :--- | :--- | :--- | :--- |
| **R Packages (CRAN)** | `ggplot2`, `dplyr`, `tidyr`, `data.table` | ✅ **Yes (Self-service)** | `install.packages("pkg")` or `pak::pkg_install("pkg")` | User local directory (`$R_LIBS_USER`) |
| **Bioconductor Packages** | `GenomicRanges`, `DESeq2`, `Biostrings` | ✅ **Yes (Self-service)** | `BiocManager::install("pkg")` | User local directory (`$R_LIBS_USER`) |
| **GitHub / Development Packages** | Unreleased packages, community repos | ✅ **Yes (Self-service)** | `remotes::install_github("user/repo")` | User local directory (`$R_LIBS_USER`) |
| **Personal R Functions & Scripts** | Custom user analysis routines | ✅ **Yes (Self-service)** | Save into your project or `~/R/` directory | User home or `/data/$USER/` |
| **Linux OS Libraries & C/C++ Headers** | `libgdal-dev`, `libgeos-dev`, `libproj-dev`, `libudunits2-dev`, `libnetcdf-dev`, `libhdf5-dev` | ❌ **No (Requires root inside OS image)** | **Contact Admins** (or use **Pixi** for a local R stack) | Shared container (`/data/shared/containers/r/`) |
| **GPU / CUDA Backends** | PyTorch / TensorFlow R bindings, CUDA drivers | ❌ **No (Host driver dependency)** | **Contact Admins** | System & container level |

---

### Handling Missing Linux System Libraries (e.g., GDAL, GEOS, NetCDF)

When installing specialized R packages (especially geospatial, environmental, climate, or bio packages like `sf`, `terra`, `ncdf4`, or `rgdal`), package installation from CRAN or GitHub may fail with a compilation error:
```text
configure: error: gdal-config not found or libgeos not found.
ERROR: configuration failed for package ‘sf’
```

This error means the R package requires underlying **compiled Linux C/C++ system libraries and header files** (`libgdal-dev`, `libgeos-dev`, `libproj-dev`, `libnetcdf-dev`, `libudunits2-dev`).

Because the cluster's standardized R container filesystem is strictly read-only (`ro`), you cannot run `sudo apt-get install` inside the container. If you encounter this, you have **two paths forward**:

#### Path 1 (⭐️ Highly Recommended): Inform the Cluster Administrators

The cleanest, easiest, and most resource-efficient option is to have the system library added to the cluster's shared R image:

1. **Contact cluster admins**: Email or message the administration team with the R package and the required system libraries (e.g., *"I need `sf` and `terra`, which require GDAL and GEOS"*).
2. **Admins bake it into the shared image**: Administrators will update the central container image (`/data/shared/containers/r/`) with the necessary Linux packages and C/C++ headers.
3. **Key Advantages**:
   * **Zero User Quota Impact**: System libraries and dependencies are stored in the shared container image, consuming **0 MB** of your personal `/home` or `/data` disk quota.
   * **Works Across All Nodes**: Available immediately on all CPU, GPU, and login nodes without configuration drift.
   * **Fully Compatible with Open OnDemand RStudio**: Works out of the box in the Open OnDemand web portal, interactive RStudio Server, and CLI wrappers (`R-4.6`, `Rscript-4.6`).

---

#### Path 2 (Self-Service): Install a Local R & Needed Libraries Using `pixi`

If you require specialized or bleeding-edge system libraries immediately and cannot wait for admin intervention, or need full autonomy over your stack, you can create a **local, user-space R environment using `pixi`**. *(For a comprehensive guide on Pixi's general capabilities, C/C++ compilers, and system utilities, see [Section 4: Pixi](#pixi-adding-system-level-libraries--compilers-without-sudo)).*

> [!IMPORTANT]
> **Why does Pixi install its own R instead of using the cluster's R?**  
> You **cannot** simply install C/C++ libraries locally and tell the cluster's containerized `R-4.6` to link against them. Mixing host libraries with container runtimes causes fatal ABI (`glibc` and `libstdc++`) version incompatibilities.  
> Instead, **Pixi installs your own private copy of R along with the exact matching C/C++ libraries and R packages**, guaranteeing 100% binary compatibility inside a project sandbox.

##### What is Pixi?
* **Pixi** is a modern, ultra-fast scientific package manager built on top of the **Conda-Forge** binary repository. The `pixi` command is pre-installed and available in your `$PATH` across all cluster nodes.
* Unlike legacy Conda, Pixi **does not pollute a messy global directory** (`~/.conda`). It behaves like `uv` or Cargo: all packages, shared libraries, and your personal R binary are stored cleanly in a `.pixi/` directory inside your specific project folder.
* **No Root Required**: Everything installs 100% in user space.
* **Pre-compiled Binaries**: Pixi downloads ready-to-run Linux packages—you do not have to spend hours waiting for C/C++ code to compile from source.

##### Step-by-Step: Setting Up Your Own R Stack with Pixi

1. **Create your project folder on `/data`**:  
   *(Always build environments on `/data` to protect your 200 GB home directory quota!)*
   ```bash
   mkdir -p /data/$USER/my_r_project
   cd /data/$USER/my_r_project
   ```

2. **Initialize Pixi**:
   ```bash
   pixi init
   ```
   This creates a `pixi.toml` file to track your project's dependencies and lockfile.

3. **Add R and the required system libraries & packages**:
   ```bash
   # Installs your own private R runtime, GDAL, GEOS, PROJ, and matching R packages:
   pixi add r-base gdal geos proj r-sf r-terra r-tidyverse
   ```
   *Pixi downloads the pre-compiled packages and places them into `.pixi/envs/default/`.*

4. **Run R interactively or execute scripts**:
   Whenever you want to use your local R environment, prepend your command with `pixi run`:
   ```bash
   # Launch your local interactive R console:
   pixi run R

   # Run an R script directly:
   pixi run Rscript analysis.R
   ```
   *`pixi run` automatically activates your project's local environment, executes the command, and cleanly exits.*

5. **Submitting SLURM Batch Jobs with Pixi**:  
   You do not need to target specific nodes with `--nodelist`! Because `/data` is shared across all nodes and Conda-Forge packages are built for standard `x86_64` Linux CPUs, your Pixi environment runs seamlessly on any compute node:

   ```bash
   #!/bin/bash
   #SBATCH --job-name=pixi-r-job
   #SBATCH --output=pixi-r-%j.out
   #SBATCH --error=pixi-r-%j.err
   #SBATCH --partition=cpu
   #SBATCH --cpus-per-task=8
   #SBATCH --mem=32G
   #SBATCH --time=04:00:00

   # Navigate to your project directory on /data
   cd /data/$USER/my_r_project

   # Execute your analysis using your local Pixi R environment
   pixi run Rscript analysis.R
   ```

##### Comparison: Which Path Should You Choose?

| Feature | Path 1: Central Shared R (`R-4.6`) | Path 2: Local Pixi R (`.pixi/`) |
| :--- | :--- | :--- |
| **Best For** | Everyday research & interactive Open OnDemand RStudio | Niche C++ dependencies, rapid experimentation, standalone SLURM batch jobs |
| **Open OnDemand RStudio Web** | ✅ **Yes** (Interactive web IDE) | ❌ **No** (CLI and SLURM batch scripts only) |
| **Disk Quota Used** | **0 MB** (reads from shared image) | ~1–3 GB per project folder on `/data` |
| **Admin Permissions** | Requires admin to update the central image | ❌ **No root needed** (100% user-managed) |
| **SLURM Execution** | `Rscript-4.6 analysis.R` | `pixi run Rscript analysis.R` |

---

### Setting Up Your Personal R Library (Quota Protection)

By default, R attempts to save personal packages to `~/R/x86_64-pc-linux-gnu-library/<version>/`. 

Because your home directory (`/home/username`) has a **200 GB quota limit**, installing dozens of large packages (especially heavy packages with compiled binaries) can fill up your home storage.

#### Recommended Practice: Store Packages in `/data`
We strongly recommend directing R to store your libraries in your group's `/data` directory:

1. Create your base R directory on `/data`:
   ```bash
   mkdir -p "/data/GroupName/$USER/R"
   ```

2. Configure your personal library path in `~/.Rprofile`:
   *(In Rocker containers, R's internal startup configuration overrides `~/.Renviron`. Using `~/.Rprofile` is the official, bulletproof method to guarantee your personal directory is prioritized at position `[1]`):*
   ```bash
   nano ~/.Rprofile
   ```
   Put this inside (make sure to replace `GroupName` with your actual research group name, e.g. `DataLab`), the final path added ( for installing the libraries ) in this example will be `/data/GroupName/myUserName/R/library-4.6` where myUserName is the logged in username on the cluster:
   ```r
   local({
     user <- Sys.info()[["user"]]
     user_lib <- sprintf("/data/GroupName/%s/R/library-%s.%s", user, R.version$major, sub("\\..*", "", R.version$minor))
     dir.create(user_lib, recursive = TRUE, showWarnings = FALSE)
     .libPaths(c(user_lib, .libPaths()))
   })
   ```
   Save and exit (Ctrl+O, Enter, Ctrl+X).
   Now run:
   ```bash
   R-4.6 -e '.libPaths()'
   ```
   *(Notice that `dir.create(user_lib, recursive = TRUE)` automatically creates the exact versioned folder—such as `/data/GroupName/$USER/R/library-4.6`—the moment R launches. You do not have to manually `mkdir` new folders every time a new R version is released!)*

> [!TIP]
> **What about `~/.Renviron`?**  
> `~/.Renviron` does not exist by default on fresh accounts and is **not required**. If you previously added `R_LIBS_USER` or `R_LIBS` to `~/.Renviron`, you can safely remove those lines (or run `rm -f ~/.Renviron`) so your configuration stays completely clean.

> [!NOTE]
> **Does storing libraries on `/data` affect execution speed?**  
> **No, not at all.** Just like in Python, R loads package functions and compiled C/C++ libraries into the compute node's **local physical RAM** once when `library(package)` is called at the beginning of your script.  
> Once loaded into memory, all calculations, loops, simulations, and data manipulations execute **100% inside local RAM and CPU cache**—R never re-reads library files from the `/data` disk during your simulation. The initial load time difference between `/home` and `/data` is negligible (milliseconds) and cached in local memory by the Linux kernel.

---

### Testing and Verifying Your Local Package Installation

Follow this 4-step verification procedure to confirm that your packages install into your personal directory and persist across sessions and compute nodes:

#### Step 1: Verify Library Search Paths
Launch R from your terminal (`R-4.6`) or open RStudio Server, and check `.libPaths()`:
```r
.libPaths()
```
**Expected Output:**
```text
[1] "/data/GroupName/$USER/R/library-4.6"
[2] "/usr/local/lib/R/site-library"
[3] "/usr/local/lib/R/library"
```
The first entry must be your writable personal path, while the remaining entries are the read-only cluster system libraries.

#### Step 2: Install a Test Package
Install a lightweight community package from CRAN:
```r
install.packages("praise")
```
Notice that R automatically downloads and compiles the package into your personal library without throwing read-only permission errors.

#### Step 3: Verify the Installed Path
Check where the package was installed:
```r
find.package("praise")
```
**Expected Output:**
```text
[1] "/data/GroupName/$USER/R/library-4.6/praise"
```

#### Step 4: Test Cross-Node Persistence via SLURM
Exit R and run a one-line test through SLURM to verify that compute nodes can access your newly installed package immediately:
```bash
srun --partition=cpu Rscript-4.6 -e 'library(praise); praise("${EXCLAMATION}! You are ${adjective}!")'
```
If you see the output string, your personal package installation is fully functional, isolated, and accessible across the entire cluster!

---

## Using MATLAB on the Cluster

### Overview & Container Architecture

On Tramuntana, MATLAB is deployed using standardized, high-performance **Apptainer containers** (as covered in our container and sandbox documentation). The container encapsulates the full operating system libraries, MATLAB binaries, and toolboxes into an isolated read-only image (`.sif`).

To provide a seamless experience, we provide native command wrappers (such as `matlab2025b`). You do not need to learn complex Apptainer syntax or manage bind mounts—running `matlab2025b` behaves just like a local bare-metal command while automatically routing through the optimized cluster container.

> [!WARNING]
> #### ⚠️ Do Not Build Your Own MATLAB Container Image!
> Even though Apptainer allows building custom containers with `--fakeroot`, **we strongly discourage users from creating their own MATLAB containers**.
>
> 1. **Massive Storage Footprint:** A complete MATLAB container image is **15–20+ GB**. If individual researchers maintain their own images, the shared cluster storage will rapidly deplete.
> 2. **Complex Network Licensing:** MATLAB requires institutional FlexLM license server tokens. Misconfigured containers lead to license checkout errors.
>
> **What should you do instead?**
> If you need a new MATLAB release, a missing toolbox, or specialized configuration, **simply contact the cluster administrators**. We maintain centralized, optimized images under our Campus-Wide license that are instantly accessible to everyone on all compute nodes.

---

### Available Versions & How to Launch MATLAB

You can access MATLAB using the version-specific commands available in your `$PATH`:

* **`matlab2025b` (Recommended):** The latest cluster-wide release. It comes pre-installed with all standard and widely used MathWorks toolboxes (Parallel Computing, Statistics and Machine Learning, Optimization, Signal Processing, Image Processing, Deep Learning, etc.).
* **`matlab2020b` (Legacy):** An older release maintained for backward compatibility. While it may not contain all modern toolboxes, administrators can easily add any required packages upon request.

You can run MATLAB in three ways: via the web browser, through an interactive graphical session, or directly from the command line / SLURM.

---

#### 1. Web Portal via Open OnDemand (Interactive Apps)
The easiest way to work with MATLAB interactively is through the **Open OnDemand** web portal:
1. Open the Open OnDemand portal in your web browser.
2. Navigate to **Interactive Apps** $\rightarrow$ **MATLAB**.
3. Select your desired MATLAB version (e.g., `MATLAB R2025b (Container)`), choose your required CPU cores, memory, and walltime, and launch the session.
4. Once the session is ready, click **Launch MATLAB** to access a complete desktop interface right in your browser.

---

#### 2. Command Line Interface (CLI) & Batch Runs

When connecting via SSH, you have several options depending on whether you need a graphical interface, an interactive terminal, or a non-interactive batch run:

##### A. Interactive CLI in Your Terminal (No GUI)
To launch MATLAB directly inside your current SSH terminal without popping up graphical windows, use `-nodesktop` and `-nosplash`:
```bash
matlab2025b -nodesktop -nosplash
```
This immediately brings up the `>> ` MATLAB command prompt inside your shell. To exit, simply type `exit`.

##### B. Non-Interactive Batch Script Execution
To execute a MATLAB script or function non-interactively (ideal for terminal scripts and automated pipelines):
```bash
matlab2025b -batch "my_analysis_script"
```

---

#### 3. SLURM Batch Job Script Example

Here is a ready-to-use SLURM template for submitting MATLAB batch simulations to the cluster:

```bash
#!/bin/bash
#SBATCH --job-name=matlab-sim
#SBATCH --output=matlab-output-%j.txt
#SBATCH --error=matlab-error-%j.err
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --partition=cpu

echo "Starting MATLAB job on $(hostname)..."

# Run MATLAB script non-interactively
matlab2025b -batch "run_simulation"

echo "MATLAB job finished successfully!"
```

---

### Installing Packages & Toolboxes: Free vs. Paid Products

Because MATLAB runs inside a container, the container's root filesystem is mounted **strictly read-only (`ro`)**. You cannot write directly to `/opt/matlab/`.

Package and toolbox management falls into two distinct categories:

| Category | Examples | Can Users Install It? | How to Obtain It |
| :--- | :--- | :--- | :--- |
| **Official MathWorks Paid Products** | *Bioinformatics*, *Simulink*, *Deep Learning*, *Optimization* | ❌ No (Read-only container & license seats) | **Contact the Admins.** Under our Campus-Wide License, administrators will add official toolboxes centrally to the container. |
| **Free / Community Add-Ons & Libraries** | *JSONLab*, *export_fig*, GitHub repos, File Exchange tools | ✅ Yes (Installed into personal user space) | **Self-service via GUI (Add-On Explorer) or CLI.** Files persist in your home directory (`~/MATLAB/`). |

---

### Installing Free / Community Packages

You can install free community toolboxes and MATLAB File Exchange add-ons using either the **GUI** (recommended and easiest) or the **CLI**.

---

#### Method 1: Via the MATLAB GUI (Add-On Explorer) — ⭐️ Recommended

This is the easiest and most reliable method to discover and install community packages:

1. **Launch MATLAB with GUI:** Start MATLAB via Open OnDemand or with X11 forwarding (`matlab2025b`).
2. **Open Add-On Explorer:** On the top MATLAB Toolstrip, go to the **Home** tab $\rightarrow$ click **Add-Ons** $\rightarrow$ **Get Add-Ons**.
3. **Search for Your Package:** Use the search bar to locate your desired toolbox (for example, search for `JSONLab` or `export_fig`).
4. **Install:** Click the package card and choose **Add** / **Install**/ **Download and Add to Path**.
5. **MathWorks Account / Email Prompt:**
   > [!NOTE]
   > When you click download, MATLAB prompts you to log in with a MathWorks account email.
   > - **Any valid MathWorks account works!** You can use your university email or any personal email (such as `@gmail.com`).
   > - The email does **NOT** need to be tied to a cluster license server. MathWorks only requires this login to prevent automated web scraping from the File Exchange.
6. **Where Files Are Saved & Isolation:**
   - Your login session token is stored in `~/.matlab/`.
   - The downloaded package files are saved into your personal directory:
     ```text
     /home/<username>/MATLAB/Add-Ons/
     ```
   - **Complete User Isolation:** Because packages reside inside `/home/<username>/`, your installations are private, never conflict with other users, and persist across all cluster compute nodes and future sessions.

To verify your installed add-ons from the MATLAB command prompt:
```matlab
matlab.addons.installedAddons
which loadjson
```

---

#### Method 2: Via Command Line (CLI) & Git / Zip Downloads

If you are working strictly from the terminal or need to install a library hosted on GitHub or an external website, you can download and extract it into your personal directory.

##### Step 1: Download & Extract into Your Personal Toolbox Folder
We recommend placing manual toolboxes inside `~/MATLAB/Toolboxes/`:

```matlab
% 1. Create a personal toolboxes folder in your home directory
target_dir = fullfile(getenv('HOME'), 'MATLAB', 'Toolboxes');
if ~exist(target_dir, 'dir'), mkdir(target_dir); end

% 2. Download the library zip (example: export_fig from GitHub)
zip_file = fullfile(target_dir, 'export_fig.zip');
websave(zip_file, 'https://github.com/altmany/export_fig/archive/refs/heads/master.zip');

% 3. Extract the contents
unzip(zip_file, target_dir);

% 4. Clean up the zip file
delete(zip_file);
```

##### Step 2: Automatic Path Persistence (No Setup Required!)
In standard desktop MATLAB, you might typically need to fiddle with `addpath()` and `savepath(fullfile(userpath, 'pathdef.m'))`. However, `pathdef.m` can easily be overlooked if you launch MATLAB from different project directories.

**On Tramuntana, you don't need to worry about path configuration:**
* The cluster's `matlab2025b` environment is automatically pre-configured in the background via `MATLABPATH`.
* Upon launch, MATLAB automatically detects and adds `~/MATLAB/Toolboxes/` (including all subfolders) to your search path.
* If you maintain a custom `startup.m` inside `~/Documents/MATLAB/`, it is automatically executed as well.

##### Step 3: Verify Path Resolution
Exit MATLAB and start a fresh session from any directory:
```bash
matlab2025b -nodesktop -nosplash
```
Check that MATLAB resolves the tool immediately:
```matlab
which export_fig
```
**Expected Output:**
```text
/home/<username>/MATLAB/Toolboxes/export_fig-master/export_fig.m
```

The toolbox is now permanently available across all interactive sessions and SLURM jobs!

---

## Pixi: Adding System-Level Libraries & Compilers (Without Sudo)

On a shared scientific cluster, researchers frequently encounter software or packages that demand system-level installations:
* Newer compiler versions (`gcc-13`, `clang-17`, `cmake >= 3.28`, `ninja`)
* System C/C++ libraries and headers (`ffmpeg`, `libgdal-dev`, `libgeos-dev`, `libhdf5-dev`, `libnetcdf-dev`, `boost`, `openblas`)
* Domain CLI tools (`samtools`, `bedtools`, `pandoc`, `imagemagick`)
* Low-level GPU toolkits (`cuda-toolkit`, `cudnn`, `nvcc`)

On Tramuntana, compute nodes have immutable operating systems—**you cannot run `sudo apt-get install`**, and administrators cannot install conflicting libraries globally without risking job failures or node drift across the cluster.

This is where **Pixi** comes in.

---

### What is Pixi? (The General Purpose)

While you saw Pixi used in our [R guide](#handling-missing-linux-system-libraries-eg-gdal-geos-netcdf) to install local geospatial libraries (`gdal`, `geos`), **that is only one specific application**. 

**Pixi's general super-power is allowing any user to install system-level packages, C/C++ compilers, scientific libraries, and CLI tools completely in user space—without root privileges and without contaminating the host operating system.**

* **Built on Conda-Forge:** Access to over 25,000+ pre-compiled Linux binary packages.
* **Project-Centric (`.pixi/`):** Unlike legacy Conda (which pollutes a monolithic `~/.conda/` directory in your home folder), Pixi behaves like `uv` or Cargo: all binaries, libraries, and headers are stored in a `.pixi/` folder inside your project directory.
* **Lockfile Reproducibility (`pixi.lock`):** Automatically pins the exact binary version of every dependency. If your job runs on `thor`, `ada`, or `pampero`, it runs with 100% identical binary reproducibility.
* **Zero Host Impact:** When you no longer need the environment, simply deleting the project folder completely removes all installed packages.

---

### When to Use Which Tool? (Quick Decision Guide)

| Tool | Best For | Storage Location | Sudo Required? | SLURM Command |
| :--- | :--- | :--- | :--- | :--- |
| **`uv`** | Pure Python projects & machine learning (PyTorch, NumPy) | `.venv/` in project | ❌ No | `uv run python script.py` |
| **`Pixi`** | C/C++ compilers, system libraries (`gdal`, `hdf5`, `ffmpeg`), multi-language stacks | `.pixi/` in project | ❌ No | `pixi run <command>` |
| **`Apptainer`** | Commercial software (MATLAB), official Docker images, or full custom Linux OS distributions | Shared SIFs or `/data/$USER/*.sif` | ❌ No (runs container) | `matlab2025b` / `apptainer exec ...` |
| **`Conda`** | Legacy existing environments | `~/.conda/` (or `/data/$USER/conda/`) | ❌ No | `conda activate ...` |

---

### Core Use Cases for Pixi on Tramuntana

#### 1. Modern C/C++ Compilers & Build Tools (Without Root)
If the host operating system has an older compiler (e.g., GCC 11) and your code requires modern C++20/C++23 features or modern CMake:
```bash
pixi add gcc=13 gxx=13 cmake ninja make
```
You can compile your code entirely with modern compilers inside your local sandbox.

#### 2. Scientific C/C++ & Math Libraries
If you are compiling simulation codes or installing packages that need underlying compiled headers and `.so` libraries:
```bash
pixi add boost hdf5 netcdf-cxx4 openblas fftw openmp
```
Pixi automatically configures include paths (`.pixi/envs/default/include`) and library link paths (`.pixi/envs/default/lib`).

#### 3. Multimedia & Geospatial CLI Tools
Need command-line tools that usually require `apt-get`? Install them instantly as pre-compiled binaries:
```bash
# Video and multimedia processing:
pixi add ffmpeg

# Geospatial command-line tools (gdal_translate, ogr2ogr):
pixi add gdal

# Bioinformatics command-line tools:
pixi add samtools bedtools --channel bioconda
```

#### 4. Isolated CUDA & GPU Toolkits
If your GPU code requires a specific CUDA toolkit version (`cuda-toolkit`, `nvcc`, `cudnn`) without modifying the host NVIDIA driver:
```bash
pixi add cuda-toolkit=12.4
```
The user-space CUDA runtime runs in your project and communicates seamlessly with the host GPU through the host's NVIDIA driver.

#### 5. Polyglot Projects (C++ Backend + Python / R Frontend)
If your project combines a compiled C++ simulation with Python analysis scripts or R plotting, Pixi manages all tools in the *same* environment with guaranteed binary compatibility and a single `pixi.lock` file.

---

### Step-by-Step Guide: Using Pixi

> [!IMPORTANT]
> **Always create your Pixi project on `/data`!**  
> Packages, compilers, and dependencies can take 1–3 GB. Storing them inside `/data/$USER/` ensures you never fill up your 200 GB `/home` directory quota.

#### Step 1: Create and Initialize Your Project Folder
```bash
mkdir -p /data/$USER/my_pixi_project
cd /data/$USER/my_pixi_project

# Initialize a new Pixi project
pixi init
```
This generates:
* `pixi.toml`: Manifest tracking your project dependencies.
* `pixi.lock`: Lockfile guaranteeing exact version pinning across all cluster nodes.

#### Step 2: Add Packages and Libraries
Search for any package on [Conda-Forge](https://prefix.dev/) and add it:
```bash
# Example: Adding modern C++ compilers and CMake
pixi add gcc=13 gxx=13 cmake ninja

# Example: Adding system libraries
pixi add hdf5 boost

# Example: Adding CLI utilities
pixi add ffmpeg
```
Pixi downloads the pre-compiled packages and places them into `.pixi/envs/default/`.

#### Step 3: Run Commands in the Environment
You have two ways to execute commands inside your Pixi environment:

##### Option A: `pixi run` (Recommended for scripts & jobs)
Prepend any command with `pixi run`. Pixi automatically activates the environment, sets `$PATH` and `$LD_LIBRARY_PATH`, executes the command, and exits:
```bash
pixi run gcc --version
pixi run ffmpeg -version
pixi run cmake -B build -G Ninja
pixi run cmake --build build
```

##### Option B: `pixi shell` (Interactive Session)
If you are doing exploratory interactive work, drop into an activated subshell:
```bash
pixi shell
```
Inside the subshell, all your installed tools (`gcc`, `g++`, `cmake`, `ffmpeg`) are directly in your `$PATH`. When finished, simply type `exit`.

---

### Practical Example: Compiling & Running a C++ Simulation

Here is a complete workflow for compiling a modern C++ simulation with CMake and HDF5 on Tramuntana:

```bash
# 1. Setup project
mkdir -p /data/$USER/simulation && cd /data/$USER/simulation
pixi init

# 2. Add compilers, build tools, and HDF5 library
pixi add gcc=13 gxx=13 cmake ninja hdf5

# 3. Configure and build with CMake
pixi run cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
pixi run cmake --build build

# 4. Run the compiled binary
pixi run ./build/my_simulation
```

---

### Submitting Pixi Jobs to SLURM

Because `/data` is shared across all nodes on the cluster and Conda-Forge binaries are standard `x86_64` Linux packages, your Pixi environment runs identically on any compute node.

In your SLURM batch script, simply use `pixi run`:

```bash
#!/bin/bash
#SBATCH --job-name=pixi_job
#SBATCH --output=pixi_%j.out
#SBATCH --error=pixi_%j.err
#SBATCH --partition=cpu
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00

# 1. Navigate to your project directory on /data
cd /data/$USER/simulation

# 2. Execute your program using pixi run
pixi run ./build/my_simulation
```

> [!TIP]
> **Why `pixi run` is superior in SLURM scripts:**  
> Unlike Conda, you do **not** need `source ~/.bashrc`, `conda activate`, or environment path wrestling. `pixi run` detects the local `.pixi/` directory, activates all environment variables on the fly, runs the binary, and cleanly exits.

---

### Useful Pixi Commands Cheatsheet

| Command | Description |
| :--- | :--- |
| `pixi init` | Initializes a new project in the current directory (`pixi.toml`). |
| `pixi add <pkg>` | Adds and installs a package from Conda-Forge into `.pixi/`. |
| `pixi add <pkg> --channel bioconda` | Installs a package from an alternative channel (e.g., Bioconda). |
| `pixi remove <pkg>` | Uninstalls a package and removes it from `pixi.toml` and `pixi.lock`. |
| `pixi list` | Shows all installed packages and their exact versions. |
| `pixi run <command>` | Runs a command inside the isolated project environment. |
| `pixi shell` | Starts an interactive shell with the environment activated. |
| `pixi clean` | Removes the `.pixi/` environment directory to free up disk space. |

---

## Apptainer: Custom Containers (When Nothing Else Works)

The tools above cover the vast majority of research needs:

| Tool | What it covers |
| :--- | :--- |
| **`uv` / Conda** | Python versions and Python packages |
| **R & MATLAB** | Ready-to-use, centrally maintained containers (`R-4.6`, `matlab2025b`) |
| **Pixi** | Compilers, system libraries (`gdal`, `hdf5`, `ffmpeg`), CLI tools, CUDA toolkits — all without `sudo` |

However, there are a few things **none of these tools can do**. For those cases, you can use **Apptainer** (formerly Singularity), the standard container platform for HPC clusters.

---

### What is Apptainer?

Apptainer packages an **entire Linux operating system** (e.g., Ubuntu 24.04 or Rocky Linux 9), along with all your software, into a single image file (`.sif`). When you run it, your program sees the container's OS and libraries instead of the cluster's — but it still runs directly on the cluster's CPUs and GPUs with **no virtual machine overhead**.

* **No root needed to run:** Containers run with your normal user permissions.
* **Your files are available inside:** Your `/home` and `/data` directories can be accessed from inside the container.
* **GPU support:** Add the `--nv` flag and the container can use the node's NVIDIA GPUs.
* **You already use it:** The cluster's `R-4.6` and `matlab2025b` commands are actually Apptainer containers behind small wrapper scripts!

---

### When Do You Need Apptainer? (Things Pixi & uv Cannot Do)

Use Apptainer **only** if your case matches one of these:

#### 1. A Paper or Tool Ships as a Docker Image
If a project says *"run our software with `docker pull lab/model:v1`"*, you cannot use Docker on the cluster — but Apptainer can run Docker images directly:
```bash
apptainer run docker://lab/model:v1
```

#### 2. Software Only Available via `apt-get` / `.deb` / `.rpm`
If a tool is **not on Conda-Forge** (so `pixi add` fails) and the installation instructions only say `sudo apt-get install ...` or provide a `.deb` package, a container lets you run `apt-get install` inside your own private OS.

#### 3. Software Expecting Hardcoded System Paths
Some older or commercial tools insist on files in fixed locations like `/etc/tool.conf`, `/opt/vendor/`, or `/usr/share/...`. Only a container gives you your own root filesystem where these paths can exist.

#### 4. Exact Reproduction of a Full Software Stack
If you must reproduce a published environment **exactly** (specific OS version + specific libraries), or move the same environment between Tramuntana and another supercomputer (e.g., MareNostrum), a single `.sif` file is the most portable option.

> [!TIP]
> **Always try Pixi first!**  
> Before reaching for a container, check whether `pixi add <package>` works. In the vast majority of cases, Conda-Forge already has the library or tool you need, and Pixi installs it in seconds instead of building a multi-gigabyte image.

---

### ⚠️ Before You Start: Contact the Cluster Administrators

> [!IMPORTANT]
> **Please consult the cluster admins before building your own container.**  
> Apptainer images are **heavy applications**: a single `.sif` file is typically **5–30 GB**, and building one uses a lot of temporary disk space, CPU, and memory. To keep shared storage and nodes healthy for everyone, please get in touch first.

When you contact the admins, please include:
1. **What software you need** and a link to its installation instructions or Docker image.
2. **Why Pixi / uv do not work** for this case (e.g., *"not on Conda-Forge, only a `.deb` package is provided"*).
3. **Whether you need GPUs** and roughly how often you will run it.

The admins may be able to:
* **Point you to an easier solution** (e.g., a Pixi package you missed, or an existing container).
* **Build it centrally** in `/data/shared/containers/` — so it uses **0 GB** of your quota and is available to your whole group, just like `R-4.6` and `matlab2025b`.
* **Enable container building (`--fakeroot`) for your account** if you need to build and iterate on your own image.

---

### Option A: Running an Existing Docker Image

The simplest case: convert a Docker image into a `.sif` file and run it.

#### Step 1: Redirect the Apptainer Cache to `/data`
By default, Apptainer stores downloaded layers in `~/.apptainer/cache`, which can quickly fill your `/home` quota. Redirect it to `/data` first:
```bash
mkdir -p /data/$USER/apptainer/{cache,tmp,images}
export APPTAINER_CACHEDIR=/data/$USER/apptainer/cache
export APPTAINER_TMPDIR=/data/$USER/apptainer/tmp
```
*(Add the two `export` lines to your `~/.bashrc` to make this permanent.)*

#### Step 2: Pull the Image
```bash
cd /data/$USER/apptainer/images
apptainer pull my_tool.sif docker://lab/model:v1
```

#### Step 3: Run Commands Inside the Container
```bash
# Run a single command inside the container:
apptainer exec --bind /data /data/$USER/apptainer/images/my_tool.sif my_command --help

# Open an interactive shell inside the container:
apptainer shell --bind /data /data/$USER/apptainer/images/my_tool.sif
```

---

### Option B: Building Your Own Container (After Consulting the Admins)

If you need to install software with `apt-get` inside your own OS, you write a small **definition file** (a recipe) and build the image from it.

#### Step 1: Write a Definition File (`my_env.def`)
```singularity
Bootstrap: docker
From: ubuntu:24.04

%post
    # Runs once during the build, as "root" inside the container
    apt-get update && apt-get install -y \
        build-essential \
        cmake \
        some-tool-only-on-apt
    apt-get clean

%environment
    # Environment variables set every time the container runs
    export LC_ALL=C

%runscript
    # Default command for `apptainer run`
    exec "$@"
```

#### Step 2: Build the Image
```bash
cd /data/$USER/apptainer
apptainer build --fakeroot images/my_env.sif my_env.def
```
* `--fakeroot` lets you act as "root" **inside the container only** (to run `apt-get`), without any real admin rights on the cluster.
* If you get a permission or `subuid` error, `--fakeroot` has not been enabled for your account yet — contact the admins.

#### Step 3: Test It
```bash
apptainer exec --bind /data images/my_env.sif some-tool-only-on-apt --version
```

> [!NOTE]
> **Containers are read-only once built.**  
> You cannot `apt-get install` into an existing `.sif`. To add software, edit `my_env.def` and rebuild. Keep the `.def` file in Git — it is your reproducible recipe, while the `.sif` itself is too large to version-control.

---

### Using GPUs Inside a Container

Add the `--nv` flag. Apptainer automatically makes the node's NVIDIA driver and GPU devices available inside the container, so any CUDA version inside your image can use the physical GPU:
```bash
apptainer exec --nv --bind /data my_env.sif nvidia-smi
apptainer exec --nv --bind /data my_env.sif python train.py
```

---

### Submitting Apptainer Jobs to SLURM

```bash
#!/bin/bash
#SBATCH --job-name=apptainer_job
#SBATCH --output=apptainer_%j.out
#SBATCH --error=apptainer_%j.err
#SBATCH --partition=gpu
#SBATCH --gres=gpu_mem:20             # Request 20 GB of GPU VRAM
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00

# Path to your container image
SIF=/data/$USER/apptainer/images/my_env.sif

cd /data/$USER/my_project

# --nv enables GPU access (omit it for CPU-only jobs)
apptainer exec --nv --bind /data "$SIF" ./run_analysis.sh
```

---

### Useful Apptainer Commands Cheatsheet

| Command | Description |
| :--- | :--- |
| `apptainer pull <name>.sif docker://<image>` | Downloads a Docker image and converts it into a `.sif` file. |
| `apptainer build --fakeroot <name>.sif <recipe>.def` | Builds a custom image from a definition file. |
| `apptainer exec <image>.sif <command>` | Runs a single command inside the container. |
| `apptainer shell <image>.sif` | Opens an interactive shell inside the container. |
| `apptainer run <image>.sif` | Executes the container's default `%runscript`. |
| `--bind /data` | Makes `/data` available inside the container. |
| `--nv` | Enables NVIDIA GPU access inside the container. |
| `apptainer cache clean` | Deletes cached image layers to free up disk space. |



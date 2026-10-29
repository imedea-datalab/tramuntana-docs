# Software & Environments

This guide covers everything you need to know about setting up software environments (`uv` & Conda) and R on the Tramuntana cluster.

**Other Practical Guides:**
- [**Quick Start & Commands**](0-quick-start.md) — Connecting to the cluster and running your first job.
- [**Commands In-Depth**](0b-commands-in-depth.md) — Advanced SLURM options, Job Arrays, and GPU monitoring.

---

## Table of Contents

1. [Setting Up Environments](#setting-up-environments)
2. [Using R for Statistical Computing](#using-r-for-statistical-computing)
3. [Using MATLAB on the Cluster](#using-matlab-on-the-cluster)

---

## Setting Up Environments

Before running your Python code on the cluster, you need a place to put all the tools and libraries your code needs. Think of an **environment** like a dedicated sandbox for your project. If Project A needs an older version of a library, and Project B needs a newer version, you don't want them fighting. Environments keep everything neatly separated so the sand from one castle doesn't mix with another.

We offer two main ways to manage your environments: **uv** (highly recommended) and **Conda**.

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
| **Linux OS Libraries & C/C++ Headers** | `libgdal-dev`, `libgeos-dev`, `libproj-dev`, `libudunits2-dev`, `libnetcdf-dev`, `libhdf5-dev` | ❌ **No (Requires root inside OS image)** | **Contact Admins.** Admins add system packages into the base image. | Shared container (`/data/shared/containers/r/`) |
| **GPU / CUDA Backends** | PyTorch / TensorFlow R bindings, CUDA drivers | ❌ **No (Host driver dependency)** | **Contact Admins** | System & container level |

> [!NOTE]
> If an R package fails to install with a message like `configure: error: gdal-config not found` or `libgeos not found`, this means the R package requires an underlying Linux system library that is missing from the container image. **Contact the administrators** to have the system dependency baked into the standardized image.

---

### Setting Up Your Personal R Library (Quota Protection)

By default, R attempts to save personal packages to `~/R/x86_64-pc-linux-gnu-library/<version>/`. 

Because your home directory (`/home/username`) has a **200 GB quota limit**, installing dozens of large packages (especially heavy packages with compiled binaries) can fill up your home storage.

#### Recommended Practice: Store Packages in `/data`
We strongly recommend directing R to store your libraries in your group's `/data` directory:

1. Create a personal R library directory on `/data`:
   ```bash
   mkdir -p "/data/$USER/R/library"
   ```

2. Add the following line to your personal `~/.Renviron` file:
   ```bash
   echo 'R_LIBS_USER="/data/'$USER'/R/library-%v"' >> ~/.Renviron
   ```
   *(The `%v` token automatically expands to the active R major.minor version, such as `4.6`, keeping packages cleanly separated between R versions).*

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
[1] "/data/username/R/library-4.4"
[2] "/usr/local/lib/R/site-library"
[3] "/usr/lib/R/library"
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
[1] "/data/username/R/library-4.6/praise"
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



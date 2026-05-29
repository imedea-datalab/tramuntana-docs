# Software & Environments

This guide covers everything you need to know about setting up software environments (`uv` & Conda) and R on the Tramuntana cluster.

**Other Practical Guides:**
- [**Quick Start & Commands**](0-quick-start.md) — Connecting to the cluster and running your first job.
- [**Commands In-Depth**](0b-commands-in-depth.md) — Advanced SLURM options, Job Arrays, and GPU monitoring.

---

## Table of Contents

1. [Setting Up Environments](#setting-up-environments)
2. [Using R for Statistical Computing](#using-r-for-statistical-computing)

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
### R Job Arrays - Running Multiple Similar Jobs

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



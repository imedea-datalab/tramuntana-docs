# Part 5: Your First Job

Welcome back! You’ve learned what the cluster hardware is ([Part 1](1-under-the-hood.md)), you've met SLURM the manager ([Part 2](2-what-is-slurm.md)), you successfully connected to the login node ([Part 3](3-getting-connected.md)), and you now understand where your files actually live ([Part 4](4-the-file-system.md)).

Now, it is finally time to run something. 

---

## 1. Writing Your Job Script (A Letter to the Manager)

Remember from Part 2 that you don’t run things directly on the login node. Instead, you have to formally request resources from SLURM (the manager), and tell it exactly what you want it to run on the compute nodes (the workers). 

You do this by writing a **Job Script**. 

A job script is basically a letter to the manager. The top part of the letter contains your physical requirements (how much memory, how many CPUs, and how much time you need). The bottom part contains the actual commands you want the worker to run once they turn on the machine.

Let's create a file called `my-first-job.sh` and see what it looks like:

```bash
#!/bin/bash

# --- THE REQUEST TO SLURM ---
#SBATCH --job-name=hello-cluster     # A friendly name for your job
#SBATCH --output=results-%j.txt      # Where to save the output logs (%j means the job ID)
#SBATCH --ntasks=1                   # How many tasks (usually 1 for simple scripts)
#SBATCH --cpus-per-task=4            # How many CPU cores you need
#SBATCH --mem=8G                     # How much RAM you need (e.g., 8 Gigabytes)
#SBATCH --time=01:00:00              # Maximum time allowed (Hours:Minutes:Seconds)
#SBATCH --partition=cpu              # Which queue/lane to use

# --- THE ACTUAL WORK ---
echo "Hello from the compute node!"
echo "I am running on: $(hostname)"
echo "My job ID is: $SLURM_JOB_ID"

# Here is where you would normally run your script:
# python my-heavy-analysis.py
# Rscript my-stats.R

echo "Job finished! Going to sleep."
```

### Understanding the `#SBATCH` Flags
Notice that all the instructions for the manager start with `#SBATCH`. Even though they look like comments, SLURM reads these lines carefully to know what to assign you.
*   `--cpus-per-task`: How many brains you need. Don't ask for 128 if your code is only programmed to use 1.
*   `--mem`: Important! If your program tries to use more RAM than you ask for here, SLURM will kill it mercilessly. Give yourself a little buffer.
*   `--time`: If your job takes longer than this, SLURM kills it. But if it finishes earlier, you *only* consume the time you used. It's better to overestimate your time slightly.

---

## 2. Submitting and Monitoring

Once you have written your `my-first-job.sh` file, you hand it to the manager. You do this from the login node using the `sbatch` command:

```bash
sbatch my-first-job.sh
```

SLURM will reply with something like: `Submitted batch job 10045`. That number is your Job ID. 

### How is my job doing?
To see the status of your job, you can check the queue using:
```bash
squeue -u your-username
```
This shows you a list of all your jobs. Look at the **ST (Status)** column:
*   **PD (Pending):** The cluster is busy, and your job is waiting in line.
*   **R (Running):** Your job is currently running on a compute node!

### Where are my results?
Remember the `--output=results-%j.txt` line in the script? When your job is running, any printed text (like our `echo` commands) is saved into that text file in your current folder. 

When your job is done, you can type `cat results-10045.txt` to read exactly what the worker node had to say.

---

## 3. Loading Your Tools (Conda & R)

Wait, what if you want to run Python or R? 
The compute nodes are powerful, but they start off as a clean slate for every job. If you simply write `python my-script.py` in your job script, it might use a default system version and entirely miss all your installed packages (like `pandas` or `ggplot2`).

You need to load your environment *inside* the script, right before you run your code.

### If you use Conda (Python)
Conda is a package manager that lets you create isolated environments with specific Python versions and packages.

Before using it for the first time, you must initialize it (do this just once on the login node):
```bash
/data/shared/miniforge3/bin/conda init bash
source ~/.bashrc
```

Then, you can create a custom environment and install packages (also do this just once on the login node):
```bash
conda create -n myproject python=3.11
conda activate myproject
conda install numpy pandas matplotlib
```

**In your Job Script**, you just need to activate this custom environment:
```bash
# --- THE ACTUAL WORK ---
# Load Conda
source /data/shared/miniforge3/bin/activate
# Activate your specific environment
conda activate myproject

# Run your heavy python script
python data_analysis.py
```

### If you use R
R 4.4.2 is already installed system-wide on the cluster. You just need to load it dynamically into your job.

**In your Job Script**, add these lines before running your R code:
```bash
# --- THE ACTUAL WORK ---
# Load the R environment
source /data/shared/R/setup-R.sh

# Run your R script
Rscript my_stats_model.R
```

---

## 4. Interactive Work: RStudio & Jupyter Notebooks

Sometimes you don't want to submit a script and walk away. You have a new dataset, and you want to look at it, plot a few test graphs, and explore interactively. This is exactly what tools like **RStudio Server** and **Jupyter Notebooks** are for.

Since you **can't** run heavy calculations on the login node, you can't just type `jupyter notebook` or start RStudio there. Instead, you have to temporarily "rent" a compute node, and then tunnel its visual interface back to the browser on your personal laptop.

### Example A: RStudio Server

Here is the quick 3-step magic to run RStudio:

**Step 1: Rent a compute node directly**  
From the login node, type:
```bash
salloc --cpus-per-task=8 --mem=16G --time=04:00:00
```
This is like `sbatch`, but instead of running a script in the background, it drops your terminal directly inside a compute node (like `pampero` or `ada`) for 4 hours. Take note of which node you landed on.

**Step 2: Connect the tunnel**  
Open a **new, entirely separate terminal window** on your personal laptop (not inside the cluster). Type this command (replace `username` with yours, and `nodename` with the node you are renting):
```bash
ssh -L 8787:localhost:8787 -J username@tramuntana username@nodename
```
Keep this terminal running. It forms a tunnel from the worker node -> through the login node -> to your laptop.

**Step 3: Open your browser**  
Go to Google Chrome (or Firefox) on your laptop and browse to: `http://localhost:8787`
Log in with your normal IMEDEA credentials. Boom! You are looking at a full RStudio interface, powered by the cluster.

---

### Example B: Jupyter Notebook on a GPU Node

What if you are training an AI model and need to experiment interactively using a GPU?

**Step 1: Rent a GPU node**  
From the login node, request a node from the GPU partition:
```bash
salloc --partition=gpu --gres=gpu:1 --cpus-per-task=8 --mem=32G --time=04:00:00
```
Once your request is granted, you will land on one of the GPU nodes (like `tramuntana-n1` or `thor`).

**Step 2: Start Jupyter**  
While on that GPU node, load your Conda environment and start the notebook:
```bash
source /data/shared/miniforge3/bin/activate
conda activate myproject
jupyter notebook --no-browser --port=8888
```
Jupyter will print out some text, including a long URL with a `token=` at the end. Keep this terminal open!

**Step 3: Connect the tunnel**  
Just like with RStudio, open a **new terminal on your laptop**, and tunnel into port 8888:
```bash
ssh -L 8888:localhost:8888 -J username@tramuntana username@nodename
```

**Step 4: Open your browser**  
Copy that URL with the token that Jupyter gave you in Step 2, and paste it into your laptop's browser. You now have a Jupyter notebook backed by a massive datacenter GPU!

> ⚠️ **Important for Interactive Jobs:** When you are done with Jupyter or RStudio, don't just close your browser! Go back to the terminal where you typed `salloc`, exit Jupyter/RStudio, and type `exit`. This tells SLURM you are done and releases the rented node so others can use it.

---

## 5. Can I use VS Code?

Yes, you absolutely can! Many researchers use **Visual Studio Code** for their remote work because it handles the SSH connection beautifully. 

With the **Remote - SSH** extension installed in VS Code on your laptop, you can connect directly to `tramuntana`. 

### The Golden Rule of VS Code
When you connect VS Code to the cluster, you are connecting to the **login node**. This means you can browse your files, edit your job scripts, use the Git integration, and inspect your results from the comfort of the VS Code text editor. 

**HOWEVER:** If you open an integrated terminal inside VS Code and type `python heavy_script.py`, **you are running that script on the login node**. Remember Part 2? The manager's office is no place for heavy lifting!

To use VS Code safely:
1. **Write and Edit:** Use the VS Code editor to write your Python/R scripts and your `.sh` job scripts.
2. **Submit to the Manager:** Use the integrated terminal to run `sbatch my-job.sh` to send the work to the compute nodes.
3. **Wait & Review:** Check the `.txt` output files directly in the VS Code editor once the job is finished.

By following this workflow, you get all the comfort of modern developing without accidentally bringing the login node to a crawl!

---

## Recap: What You've Learned

1. **Job Scripts:** You request resources and give instructions to the cluster by making a `.sh` file full of `#SBATCH` tags.
2. **Submitting:** You hand the script to the manager using `sbatch my-script.sh`.
3. **Monitoring:** You check the queue with `squeue -u your-username`.
4. **Environments:** Because nodes start clean, you must activate your Conda or R environments *inside* your job script before running the actual code.
5. **Interactive Nodes:** You can rent a blank or GPU node with `salloc` and build an SSH tunnel to run Jupyter Notebooks or RStudio safely.
6. **VS Code Safely:** You can use Remote SSH to edit files, but remember to *always submit jobs with `sbatch`* rather than running heavy code directly in the VS Code terminal.

---
**Navigation:**
🏠 [Home](README.md) | 📖 [1. Under the Hood](1-under-the-hood.md) | ⚙️ [2. What is SLURM?](2-what-is-slurm.md) | 🔌 [3. Getting Connected](3-getting-connected.md) | 📁 [4. The File System](4-the-file-system.md) | 🚀 **5. Your First Job**

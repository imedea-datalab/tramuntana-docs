# Open OnDemand: Interactive Apps

Open OnDemand (OOD) provides a user-friendly web interface to the Tramuntana cluster. You can launch interactive sessions like **MATLAB**, **VS Code**, **RStudio**, and the **AI Chatbot (RAG)** directly from your browser, without needing to use the command line or write SLURM scripts manually.

## 🔗 How to Access

1. **Link**: Go to [https://10.33.0.143/](https://10.33.0.143/) in your web browser. *(Note: You must be connected to the IMEDEA network or VPN).*
2. **Login**: Use your standard IMEDEA username and password.
3. **Launch Apps**: In the top navigation bar, click on **Interactive Apps** and select the application you want to launch (e.g., MATLAB, VS Code, or RStudio).

## 🎛️ Configuring Your Session

When you select an interactive app, you will be presented with a form to request cluster resources. This form essentially builds the SLURM job submission for you under the hood.

Here is an explanation of the typical input fields:

- **Partition**: Select the queue you want to submit your job to (e.g., `cpu` or `gpu`).
- **Specific Node**: Optionally request a specific compute node from the dropdown list. You can choose from heavy compute nodes (`thor`, `tramuntana-n1`, `pampero`), the GPU node `barracuda`, or the lightweight interactive nodes `vscode-node01` and `vscode-node02` (which are intended specifically for code editing and very low-CPU interactive tasks).
- **Number of hours**: The maximum duration your interactive session will run. After this time, the job will be automatically terminated.
- **Number of cores (Tasks)**: The number of CPU cores you are requesting. For a standard session, 1-4 cores are usually sufficient unless you are running highly parallel code.
- **Memory (GB) per node**: The amount of RAM you need. Requesting too much can delay your job from starting, so ask for what you reasonably expect to use.
- **Number of GPUs**: (If applicable) The number of GPUs to allocate to your session. If you don't need a GPU, leave this blank or select 0.
- **GPU Memory (GB)**: (For RStudio or VS Code) Specify the amount of GPU VRAM to request when using a GPU partition.
- **GPU Partition**: If requesting a GPU, select the specific GPU partition.
- **Additional SLURM arguments**: Advanced users can pass extra SLURM flags here (e.g., `--exclude=node_name`).

> [!WARNING]
> **Important Note on MATLAB Versions:** When launching MATLAB, you may see an option for **MATLAB R2025b (Host Install)**. Please note that this version relies on a legacy local installation and **will not work** on newly added compute nodes (e.g., `vscode-node01`). For reliable execution across the entire cluster, it is highly recommended to select **MATLAB R2020b**, which utilizes the modern Apptainer containerized infrastructure. The 2025b option will be transitioned to the containerized approach in the future. So if you need any specific version, plese contact datalab, we can install for you.
>
> **Note on GPU Support (MATLAB vs RStudio):** Unlike RStudio (which automatically enables GPU acceleration via NVIDIA drivers when a GPU partition is selected), **MATLAB currently cannot use the GPU**. The deployed MATLAB versions require an additional license and do not include the Parallel Computing Toolbox. Consequently, native MATLAB GPU commands (like `gpuDeviceCount` or `gpuArray`) will not work out of the box, even if you select a GPU node like `barracuda`. If you require GPU acceleration inside MATLAB, please contact datalab to discuss custom licensing or container builds.

Once you fill out the form, click **Launch**. Your job will be placed in the SLURM queue. Once resources are available, the status will change to "Running," and you can click **Connect** to open the interface in a new tab.

## 🤖 AI Chatbot (RAG Assistant)

The Open OnDemand interface also includes an **AI Chatbot (RAG)** application, allowing you to ask questions and interact with institutional documentation and datasets using AI models running directly on the Tramuntana cluster.

### ⚙️ How the Backend Works (Model Loading & 30-Minute Timeout)
Understanding how the chatbot interacts with the cluster backend will help you know what to expect when asking questions:
- **First Question & Model Loading (3–4 Minutes):** When you submit a question, the interface checks if the AI backend models are already loaded into GPU memory. If not, it initiates the loading process, which typically takes **3 to 4 minutes**. During this initial warmup, please be patient while the models load!
- **Fast Subsequent Responses (30-Minute Window):** Once loaded, the models remain active in GPU memory for **30 minutes of inactivity**. During this 30-minute window, any subsequent questions asked (by you or anyone else) will receive **fast responses** because the models are already waiting in memory.
- **Automatic Inactivity Timeout:** To conserve valuable cluster GPU resources for other researchers, if no questions are asked for a 30-minute period, the backend job is automatically terminated and the models unload from VRAM. The next time a question is asked after a timeout, the 3–4 minute loading cycle will start again.

### Intelligent Queue & Loading Status
When you launch the RAG Chatbot, an underlying SLURM job (`rag_api`) is submitted to initialize the AI backend API. When large AI models are loading or the cluster is under heavy usage, startup can take longer than 5 minutes. The interface intelligently queries SLURM (`squeue`) to give you live feedback:
- **If the job is PENDING:** The interface notifies you: *"The AI backend job (`rag_api`) is currently PENDING in the SLURM queue."* This happens when all appropriate compute nodes are busy running other jobs. Your request is safely in the queue and will start automatically as soon as resources free up.
- **If the job is RUNNING:** The interface notifies you: *"The AI backend job (`rag_api`) is RUNNING and currently initializing the AI models."* Large AI models take extra time to load into VRAM. Please wait a minute and try submitting your question again.

This prevents confusion during high cluster load or lengthy model initialization!

### Node Selection & Exclusions
The RAG launcher automatically scans the cluster for an available node that meets strict hardware requirements (at least 8 free CPUs, 24 GB of system RAM, and 12 GPU shards). It prioritizes running on `thor` or `tramuntana-n1`. 
> [!IMPORTANT]
> **Why `barracuda` is excluded:** The RAG launcher **explicitly excludes `barracuda`** from running the AI backend. This is because `barracuda`'s GPU architecture (NVIDIA GV100) does not support the AWQ (Activation-aware Weight Quantization) model quantization required by the chatbot's AI models.

### 💡 How to Use & Best Practices
When interacting with the RAG Chatbot, keep the following important points in mind to get the best results:
1. **Single-Turn Context (No Conversation Memory):** To operate efficiently within limited context windows, the chatbot evaluates each question independently. It **does not keep track of conversation history** or previous turns in the chat. Always treat every message as a standalone question and include all necessary context in your prompt.
2. **Use as a Documentation Discovery & Lookup Tool:** While the chatbot provides direct answers, one of its most powerful uses is as an intelligent navigation aid. Below every response, the chatbot provides **references and citations** showing the exact sections and documents where it retrieved the information. Use these citations to quickly jump to and read the official guides for deeper understanding!

## 🔒 Fixing the Blank Screen in VS Code Jupyter Notebooks (Certificate Setup)

When opening a Jupyter Notebook (`.ipynb`) inside the web-based VS Code app, you might encounter a blank screen and an error stating: `Could not register service worker: SecurityError... An SSL certificate error occurred`.

This happens because modern browsers block Service Workers on connections that don't have a trusted SSL certificate. To fix this, you need to trust the Tramuntana certificate on your local machine.

### For Mac Users

1. Open your terminal and run this command to download the certificate to your Desktop:
   ```bash
   echo -n | openssl s_client -connect 10.33.0.143:443 2>/dev/null | openssl x509 > ~/Desktop/ood.crt
   ```
2. Tell your Mac to explicitly trust this certificate (you will be prompted for your Mac password):
   ```bash
   sudo security add-trusted-cert -d -r trustRoot -k "/Library/Keychains/System.keychain" ~/Desktop/ood.crt
   ```
3. Restart your browser and reload the VS Code page.

### For Windows Users

1. Open your browser and go to [https://10.33.0.143/](https://10.33.0.143/).
2. Click on the **"Not Secure"** warning in the URL bar, click **Certificate is not valid**, and go to the **Details** tab.
3. Click **Export** and save the file to your Desktop as `ood.crt`.
4. Press the Windows Key, type **Manage computer certificates**, and hit Enter.
5. In the left panel, expand **Trusted Root Certification Authorities** and click on **Certificates**.
6. Right-click on Certificates -> **All Tasks** -> **Import...**
7. Follow the wizard to import the `ood.crt` file.
8. Restart your browser and reload the VS Code page.

# Open OnDemand: Interactive Apps

Open OnDemand (OOD) provides a user-friendly web interface to the Tramuntana cluster. You can launch interactive sessions like **MATLAB**, **VS Code**, and **RStudio** directly from your browser, without needing to use the command line or write SLURM scripts manually.

## 🔗 How to Access

1. **Link**: Go to [https://10.33.0.143/](https://10.33.0.143/) in your web browser. *(Note: You must be connected to the IMEDEA network or VPN).*
2. **Login**: Use your standard IMEDEA username and password.
3. **Launch Apps**: In the top navigation bar, click on **Interactive Apps** and select the application you want to launch (e.g., MATLAB, VS Code, or RStudio).

## 🎛️ Configuring Your Session

When you select an interactive app, you will be presented with a form to request cluster resources. This form essentially builds the SLURM job submission for you under the hood.

Here is an explanation of the typical input fields:

- **Partition**: Select the queue you want to submit your job to (e.g., `cpu` or `gpu`).
- **Number of hours**: The maximum duration your interactive session will run. After this time, the job will be automatically terminated.
- **Number of cores (Tasks)**: The number of CPU cores you are requesting. For a standard session, 1-4 cores are usually sufficient unless you are running highly parallel code.
- **Memory (GB) per node**: The amount of RAM you need. Requesting too much can delay your job from starting, so ask for what you reasonably expect to use.
- **Number of GPUs**: (If applicable) The number of GPUs to allocate to your session. If you don't need a GPU, leave this blank or select 0.
- **GPU Partition**: If requesting a GPU, select the specific GPU partition.
- **Additional SLURM arguments**: Advanced users can pass extra SLURM flags here (e.g., `--exclude=node_name`).

Once you fill out the form, click **Launch**. Your job will be placed in the SLURM queue. Once resources are available, the status will change to "Running," and you can click **Connect** to open the interface in a new tab.

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

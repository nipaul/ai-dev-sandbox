# Docker AI Experimentation Sandbox — Windows 11 Setup Guide

> **Target Environment:** Windows 11 · Docker Desktop (WSL2 backend) · AI/ML Workloads  
> **Scope:** LLM Inference · Python ML Training · API/Backend Services · Jupyter Notebooks · IIS-Hosted Services  
> **GPU:** Both NVIDIA and AMD paths documented (choose based on your hardware)

---

## Table of Contents

1. [Prerequisites & System Check](#1-prerequisites--system-check)
2. [Enable WSL2 & Virtualization](#2-enable-wsl2--virtualization)
3. [Install Docker Desktop](#3-install-docker-desktop)
4. [GPU Passthrough Configuration](#4-gpu-passthrough-configuration)
   - [Option A — NVIDIA GPU](#option-a--nvidia-gpu)
   - [Option B — AMD GPU](#option-b--amd-gpu)
5. [Docker Performance Tuning](#5-docker-performance-tuning)
6. [Security Hardening](#6-security-hardening)
7. [Workload-Specific Container Setups](#7-workload-specific-container-setups)
   - [A. LLM Inference (Ollama / llama.cpp)](#a-llm-inference-ollama--llamacpp)
   - [B. Python ML Training (PyTorch / TensorFlow)](#b-python-ml-training-pytorch--tensorflow)
   - [C. API / Backend Services](#c-api--backend-services)
   - [D. Jupyter Notebooks](#d-jupyter-notebooks)
   - [E. IIS-Hosted Services & Apps](#e-iis-hosted-services--apps)
8. [Compose: Full Sandbox Stack](#8-compose-full-sandbox-stack)
9. [Networking Best Practices](#9-networking-best-practices)
10. [Persistent Storage Strategy](#10-persistent-storage-strategy)
11. [Monitoring & Resource Limits](#11-monitoring--resource-limits)
12. [Useful Commands Cheat Sheet](#12-useful-commands-cheat-sheet)

---

## 1. Prerequisites & System Check

### Minimum Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| RAM | 16 GB | 32 GB+ |
| Disk (free) | 60 GB | 200 GB+ (NVMe SSD) |
| CPU | 4 cores / 8 threads | 8+ cores |
| GPU VRAM | — | 8 GB+ for LLM inference |

### Check Virtualization is Enabled

Open **PowerShell (Admin)** and run:

```powershell
# Check virtualization support
systeminfo | findstr /i "virtualization"

# Check Hyper-V status
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All
```

> If virtualization is disabled, enable it in your BIOS/UEFI under **SVM Mode** (AMD) or **Intel VT-x**.

---

## 2. Enable WSL2 & Virtualization

WSL2 is Docker Desktop's recommended backend on Windows 11. It provides near-native Linux kernel performance.

### Step 1 — Enable Required Windows Features

```powershell
# Run in PowerShell (Admin)
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

Restart your machine.

### Step 2 — Install WSL2 Kernel Update

```powershell
wsl --install
wsl --set-default-version 2
```

### Step 3 — Install Ubuntu (Recommended Distro)

```powershell
wsl --install -d Ubuntu-22.04
```

Verify:

```powershell
wsl -l -v
# Should show Ubuntu-22.04 with VERSION 2
```

### Step 4 — Configure WSL2 Resource Limits

Create `C:\Users\<YourUser>\.wslconfig` with the following:

```ini
[wsl2]
# Allocate up to 70% of system RAM to WSL2
memory=24GB

# Number of logical processors (adjust to your CPU)
processors=8

# Swap file size
swap=8GB

# Enable GPU passthrough (required for CUDA / DirectML)
gpuSupport=true

# Limit memory reclaim aggressiveness (improves ML training stability)
kernelCommandLine=sysctl.vm.swappiness=10 sysctl.vm.overcommit_memory=1

[experimental]
# Enables sparse VHD — reclaims disk space automatically
autoMemoryReclaim=gradual
sparseVhd=true
```

> **Tip:** After editing `.wslconfig`, restart WSL: `wsl --shutdown` then reopen your terminal.

---

## 3. Install Docker Desktop

### Step 1 — Download & Install

Download Docker Desktop for Windows from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/).

During installation, ensure the following are checked:
- ✅ **Use WSL2 instead of Hyper-V**
- ✅ **Add shortcut to desktop**

### Step 2 — Post-Install Configuration

Open Docker Desktop → **Settings**:

#### General
- ✅ Start Docker Desktop when you sign in to your computer
- ✅ Use containerd for pulling and storing images

#### Resources → WSL Integration
- ✅ Enable integration with your default WSL distro (Ubuntu-22.04)

#### Resources → Advanced
```
CPUs:           [Set to 80% of your total logical cores]
Memory:         [Match your .wslconfig memory setting]
Disk image size: [200 GB+ recommended for AI models]
```

#### Docker Engine (Edit daemon.json)

Go to **Settings → Docker Engine** and set:

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "40GB",
      "enabled": true
    }
  },
  "experimental": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Hard": 64000,
      "Name": "nofile",
      "Soft": 64000
    }
  },
  "no-new-privileges": true,
  "userns-remap": "default"
}
```

Click **Apply & Restart**.

---

## 4. GPU Passthrough Configuration

> 🧭 **Before you start — which terminal are you using?**
>
> | Label | What it means | How to open it |
> |-------|--------------|----------------|
> | 🖥️ **Windows (Browser/Explorer)** | Normal Windows actions — downloading files, running installers | Your regular desktop |
> | 🪟 **PowerShell (Admin)** | Windows command line with elevated rights | Start Menu → search **PowerShell** → right-click → *Run as administrator* |
> | 📦 **WSL2 Terminal** | Your Linux environment inside Windows | Start Menu → search **Ubuntu** → open it |
> | 📝 **File Editor** | Editing config or compose files | Notepad, VS Code, or Notepad++ |

---

> 🤔 **What is "GPU Passthrough" and why do we need it?**
>
> By default, Docker containers are isolated — they can only see the CPU. Your GPU is invisible to them. **GPU passthrough** is the process of giving a container direct access to your physical GPU so it can use it for AI inference and model training.
>
> Without this, running a model like Llama 3 will fall back to your CPU, which is many times slower and often impractical for larger models.

---

> 🔍 **Not sure which GPU you have?**
>
> 🪟 **Run this in PowerShell (Admin):**
> ```powershell
> # This lists all display adapters installed on your machine
> Get-WmiObject Win32_VideoController | Select-Object Name, AdapterRAM
> ```
> Look for a line containing **NVIDIA**, **AMD/Radeon**, or **Intel Arc**. The `AdapterRAM` field shows your VRAM in bytes (divide by 1073741824 to get GB).
>
> - See **NVIDIA GeForce / RTX / Quadro**? → Follow **Option A**
> - See **AMD Radeon / RX / Pro**? → Follow **Option B**
> - See only **Intel UHD / Iris**? → Skip this section for now; run CPU-only and revisit when you upgrade hardware

---

### Option A — NVIDIA GPU

**How NVIDIA GPU passthrough works:** NVIDIA provides a technology called **CUDA** — a special programming interface that lets software talk directly to the GPU for parallel computation. Docker doesn't know about CUDA by default. We install the **NVIDIA Container Toolkit**, which acts as a bridge between Docker and your GPU driver so containers can use CUDA.

The setup has three parts: drivers on Windows, toolkit inside WSL2, and a verification test.

---

#### Step 1 — Install NVIDIA Drivers on Windows

> 🖥️ **Do this in Windows (browser + installer)**

**What this does:** The NVIDIA driver is what lets your Windows system (and WSL2) talk to your physical GPU. For Docker's GPU passthrough to work, you need version **525 or newer**.

1. Open your browser and go to [nvidia.com/drivers](https://www.nvidia.com/drivers)
2. Select your GPU model and Windows 11 as the OS
3. Download and run the installer — a standard Windows `.exe` setup wizard
4. When the install finishes, **restart your machine**

After restarting, confirm the driver version:

> 🪟 **Run this in PowerShell (Admin):**

```powershell
# This shows your installed NVIDIA driver version
nvidia-smi
```

Look for a line near the top that says `Driver Version: 5xx.xx`. Any number **525 or above** is good.

> ⚠️ **Important — do NOT install the CUDA Toolkit on Windows.** The CUDA Toolkit is for software development on Windows itself. For Docker, CUDA gets installed *inside the container* later. Installing it on Windows wastes space and can cause version conflicts. The driver alone is all Windows needs.

---

#### Step 2 — Install the NVIDIA Container Toolkit inside WSL2

> 📦 **Run all of the following commands in your WSL2 Terminal (Ubuntu)**

**What this does:** The NVIDIA Container Toolkit is a set of Linux tools that teaches Docker how to pass GPU access through to containers. It must be installed inside your WSL2 Linux environment, not on Windows.

The installation has four smaller steps. Run them one group at a time:

**2a — Add NVIDIA's trusted signing key** (so Ubuntu trusts packages from NVIDIA's server):

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

> 💡 `curl` downloads a file from the internet. The `|` (pipe) passes it directly into `gpg`, which converts it into a trusted key file saved at the path after `-o`. `sudo` means "run with administrator privileges" — Ubuntu will ask for your WSL2 password.

**2b — Register NVIDIA's package repository** (so Ubuntu knows where to find the toolkit):

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

> 💡 This downloads a list of software sources and registers it with Ubuntu's package manager (`apt`). Think of it as adding a new app store that sells NVIDIA's tools.

**2c — Install the toolkit:**

```bash
# First, refresh the list of available packages
sudo apt-get update

# Then install the toolkit
sudo apt-get install -y nvidia-container-toolkit
```

> 💡 `apt-get update` is like refreshing a store's catalogue. `apt-get install` is like buying and installing an item from that catalogue. The `-y` flag means "yes to all prompts" so you don't have to type Y at each confirmation.

**2d — Tell Docker to use NVIDIA's runtime, then restart Docker:**

```bash
# Register the NVIDIA runtime with Docker
sudo nvidia-ctk runtime configure --runtime=docker

# Restart Docker so the change takes effect
sudo systemctl restart docker
```

> 💡 `nvidia-ctk runtime configure` edits Docker's internal configuration to add NVIDIA as a runtime option. `systemctl restart docker` is like restarting a Windows service — it applies the new settings.

---

#### Step 3 — Verify GPU Access Works Inside a Container

> 📦 **Run this in your WSL2 Terminal (Ubuntu)**

**What this does:** This command starts a temporary NVIDIA container and runs `nvidia-smi` inside it. If your GPU passthrough is working, the container will be able to see and report on your physical GPU.

```bash
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

**Breaking down this command:**

| Part | What it means |
|------|--------------|
| `docker run` | Start a new container |
| `--rm` | Automatically delete the container when it finishes (keeps things tidy) |
| `--gpus all` | Give the container access to all available GPUs |
| `nvidia/cuda:12.2.0-base-ubuntu22.04` | The image to use — a minimal CUDA-enabled Ubuntu container from NVIDIA |
| `nvidia-smi` | The command to run inside the container — reports GPU status |

**✅ Success looks like this:**

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.xx    Driver Version: 525.xx    CUDA Version: 12.2          |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
|   0  RTX 3080           Off  | 00000000:01:00.0  On |                  N/A |
+-----------------------------------------------------------------------------+
```

Your GPU name, driver version, and VRAM should all be visible.

**❌ If you see an error like `could not select device driver "nvidia"`**, go back to Step 2d and confirm `nvidia-ctk runtime configure` ran without errors, then retry.

---

### Option B — AMD GPU

**How AMD GPU passthrough works:** AMD GPUs on Windows use a different technology to NVIDIA's CUDA. On Windows 11 with WSL2, AMD exposes the GPU through **DirectX 12** via a special device called `/dev/dxg`. Docker containers access your AMD GPU by mounting this device, rather than using a toolkit like NVIDIA's.

> 💡 You may also see the term **DirectML** — this is Microsoft's machine learning layer built on top of DirectX 12. It's the primary way ML frameworks like PyTorch and TensorFlow use AMD (and Intel) GPUs on Windows.

---

#### Step 1 — Install AMD Drivers on Windows

> 🖥️ **Do this in Windows (browser + installer)**

**What this does:** Like NVIDIA, you need up-to-date AMD drivers on Windows for GPU passthrough to WSL2 to function.

1. Open your browser and go to [amd.com/support](https://www.amd.com/support)
2. Select your GPU model (e.g., Radeon RX 6800 XT) and Windows 11 as the OS
3. Download **AMD Software: Adrenalin Edition** (for consumer GPUs) or **AMD PRO** drivers (for workstation cards)
4. Run the installer and **restart your machine** when prompted

After restarting, confirm your GPU is recognised:

> 🪟 **Run this in PowerShell (Admin):**

```powershell
# List display adapters to confirm AMD driver is active
Get-WmiObject Win32_VideoController | Select-Object Name, DriverVersion
```

You should see your AMD GPU name and a driver version number.

---

#### Step 2 — Verify DirectML is Available Inside a Container

> 📦 **Run this in your WSL2 Terminal (Ubuntu)**

**What this does:** This test starts a Python container and checks that it can load `libdxcore.so` — the shared library that exposes DirectX 12 GPU access to Linux programs running in WSL2. If this succeeds, your containers can use your AMD GPU.

```bash
docker run --rm \
  --device /dev/dxg \
  -v /usr/lib/wsl:/usr/lib/wsl \
  python:3.11-slim \
  python -c "import ctypes; ctypes.cdll.LoadLibrary('libdxcore.so'); print('DirectML available')"
```

**Breaking down this command:**

| Part | What it means |
|------|--------------|
| `docker run --rm` | Start a temporary container, delete it when done |
| `--device /dev/dxg` | Give the container access to the DirectX GPU device |
| `-v /usr/lib/wsl:/usr/lib/wsl` | Mount the WSL GPU libraries into the container so it can find them |
| `python:3.11-slim` | Use a lightweight Python container image |
| `python -c "..."` | Run a short Python script to test the library loads |

**✅ Success looks like:**
```
DirectML available
```

**❌ If you see `OSError: libdxcore.so: cannot open shared object file`**, your AMD driver may not be fully installed or WSL2 may need a restart. Run `wsl --shutdown` in PowerShell, reopen your Ubuntu terminal, and retry.

---

#### Step 3 — Add AMD GPU Access to Your Compose File

> 📝 **Edit your `docker-compose.yml` file**

Unlike NVIDIA (which uses a `--gpus` flag), AMD GPU access is given to containers by mounting two specific paths. Add these to any service that needs GPU access:

```yaml
services:
  trainer:
    image: python:3.11-slim
    # ── AMD GPU Access ──────────────────────────────
    devices:
      - /dev/dxg:/dev/dxg          # The DirectX GPU device
    volumes:
      - /usr/lib/wsl:/usr/lib/wsl:ro  # WSL GPU libraries (read-only is fine)
    environment:
      - DIRECTML_DEBUG=1           # Optional: enables verbose GPU logging
    # ────────────────────────────────────────────────
```

> 💡 The `:ro` at the end of the volume mount means **read-only**. The container can read the WSL libraries but cannot modify them — this is a good security practice.

After saving the file, bring the stack up:

> 📦 **Run this in your WSL2 Terminal:**

```bash
cd ~/ai-sandbox
docker compose up -d
```

---

### ✅ GPU Passthrough Checklist

Before moving on to Section 5, confirm the following:

**For NVIDIA users:**
- [ ] NVIDIA driver version 525+ installed on Windows and machine restarted
- [ ] NVIDIA Container Toolkit installed inside WSL2 Ubuntu (Steps 2a–2d)
- [ ] `nvidia-ctk runtime configure --runtime=docker` ran without errors
- [ ] Docker restarted via `sudo systemctl restart docker`
- [ ] Verification test showed GPU name and VRAM in the output table

**For AMD users:**
- [ ] AMD Adrenalin or PRO drivers installed on Windows and machine restarted
- [ ] DirectML verification test printed `DirectML available`
- [ ] `devices` and `volumes` entries added to relevant services in `docker-compose.yml`

**For both:**
- [ ] You know which flag or config to add when you want a container to use the GPU (`--gpus all` for NVIDIA, `--device /dev/dxg` for AMD)

---

## 5. Docker Performance Tuning

> 🧭 **New to terminals? Here's your map.**  
> You will use three different environments in this section. Before running any command, check which terminal it belongs to:
>
> | Label | What it means | How to open it |
> |-------|--------------|----------------|
> | 📦 **WSL2 Terminal** | Your Linux environment inside Windows | Start Menu → search **Ubuntu** → open it |
> | 🪟 **PowerShell (Admin)** | Windows command line with elevated rights | Start Menu → search **PowerShell** → right-click → *Run as administrator* |
> | 📝 **File Editor** | Any text editor for editing config files | Notepad, VS Code, or Notepad++ |
>
> Each step below is clearly labelled with where it runs. Never run a Linux command in PowerShell, or vice versa — they are different systems and the commands are not interchangeable.

---

### Tuning 1 — Enable BuildKit (Faster Image Builds)

**What this does:** BuildKit is Docker's modern build engine. It builds images faster by running independent steps in parallel and caching downloaded packages so they don't have to be re-downloaded every time you rebuild.

---

#### Step 1 — Enable BuildKit in your session

> 📦 **Run this in your WSL2 Terminal (Ubuntu)**

```bash
export DOCKER_BUILDKIT=1
```

**What just happened?** `export` sets an environment variable — think of it as a setting that tells Docker "use the faster build engine from now on." This only lasts for the current terminal session.

**To make it permanent** (so you don't have to run it every time), add it to your shell profile:

```bash
# This appends the setting to your profile file so it loads automatically on every login
echo 'export DOCKER_BUILDKIT=1' >> ~/.bashrc

# Reload your profile so the change takes effect immediately
source ~/.bashrc
```

> 💡 `~/.bashrc` is a text file that Ubuntu reads every time you open a new terminal. Anything you add there runs automatically at startup.

---

#### Step 2 — Use Cache Mounts in Your Dockerfiles

> 📝 **Edit this in your Dockerfile** (not a terminal — this is code you write in a file)

When Docker builds an image that installs Python packages, it normally re-downloads every package from the internet every single time — even if nothing changed. A **cache mount** tells Docker to remember those downloads and reuse them.

**Without cache mount (slow — re-downloads everything every build):**
```dockerfile
RUN pip install -r requirements.txt
```

**With cache mount (fast — reuses previously downloaded packages):**
```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
```

> 💡 Think of `/root/.cache/pip` as a local package shelf. The first build fills the shelf. Every build after that grabs packages off the shelf instead of ordering them fresh.

---

### Tuning 2 — Verify Image Storage Driver

**What this does:** Docker can store images in different ways. The `containerd` image store (which you enabled in Section 3's `daemon.json`) is faster and more efficient. This step just confirms it's active.

> 🪟 **Run this in PowerShell (Admin)** — or in your WSL2 terminal, either works

```powershell
docker system info | grep "Storage Driver"
```

**Expected output:**
```
Storage Driver: overlay2
```

> 💡 You don't need to change anything here — this is just a health check. If you see `overlay2`, you're good. If you see `vfs`, something went wrong in the daemon.json setup in Section 3 and you should revisit that step.

---

### Tuning 3 — Increase Shared Memory for ML Workloads

**What this does:** PyTorch's DataLoader (which feeds data into your model during training) relies on a special area of memory called **shared memory** (`/dev/shm`). Docker containers start with only 64 MB of shared memory by default. For ML workloads, this causes crashes or slowdowns. We need to raise it to several gigabytes.

There are two ways to do this depending on how you're starting your container.

---

#### Option A — In your `docker-compose.yml` file (Recommended)

> 📝 **Edit this in your `docker-compose.yml`** file using a text editor

Find the service you want to tune (e.g., `trainer` or `ollama`) and add `shm_size` at the same indentation level as `image:` or `ports:`:

```yaml
services:
  trainer:
    image: pytorch/pytorch:latest
    shm_size: '4gb'        # ← Add this line
    volumes:
      - ~/ai-sandbox/datasets:/data
```

> 💡 YAML files are sensitive to indentation — use spaces, not tabs, and make sure `shm_size` lines up with the other keys under your service name.

After saving the file, restart your stack for the change to take effect:

> 📦 **Run this in your WSL2 Terminal**

```bash
# Navigate to your sandbox folder first
cd ~/ai-sandbox

# Restart the stack to apply the new setting
docker compose down && docker compose up -d
```

---

#### Option B — As a flag when running a single container

> 📦 **Run this in your WSL2 Terminal**

If you're running a one-off container (not using Compose), add `--shm-size` as a flag:

```bash
docker run --shm-size=4g pytorch/pytorch:latest python train.py
```

> 💡 `--shm-size=4g` means "give this container 4 gigabytes of shared memory." You can adjust the number based on how much RAM your machine has. A safe starting point for most ML work is `4g`. For large batch training, try `8g`.

---

### Tuning 4 — Store Files on the Linux Filesystem (Critical for Speed)

**What this does:** This is one of the most impactful performance improvements you can make. When Docker runs inside WSL2, there are two separate filesystems available to it:

| Location | What it is | Speed |
|----------|-----------|-------|
| `/mnt/c/...` | Your Windows C: drive, accessed through WSL2 | 🐢 Slow (5–10× slower) |
| `~/...` (e.g. `/home/yourname/`) | The native Linux filesystem inside WSL2 | ⚡ Fast |

Model weights, datasets, and outputs should always live on the **Linux filesystem**. Accessing them through `/mnt/c/` adds a translation layer between Windows and Linux every time a file is read, which adds up quickly during training or inference.

---

#### Step 1 — Create your working directories on the Linux filesystem

> 📦 **Run this in your WSL2 Terminal**

```bash
# This creates all the folders you need in one command.
# The -p flag means "create any missing parent folders too"
mkdir -p ~/ai-sandbox/models ~/ai-sandbox/datasets ~/ai-sandbox/outputs ~/ai-sandbox/notebooks
```

**What was created:**

```
/home/yourname/ai-sandbox/
├── models/      ← Put model weight files (.gguf, .bin, .safetensors) here
├── datasets/    ← Put training or evaluation data here
├── outputs/     ← Docker containers will write results and checkpoints here
└── notebooks/   ← Your Jupyter notebooks will be saved here
```

> 💡 `~` is shorthand for your home directory — it's the same as typing `/home/yourname/`. It's faster to type and works regardless of your username.

---

#### Step 2 — Confirm you're in the right place

> 📦 **Run this in your WSL2 Terminal**

```bash
# List the folders to confirm they were created
ls ~/ai-sandbox/
```

You should see:
```
datasets  models  notebooks  outputs
```

> ⚠️ **Common beginner mistake:** Saving model files to `C:\Users\YourName\models` on Windows and then mounting that into Docker as `/mnt/c/Users/YourName/models`. This works, but is significantly slower. Always copy large files into `~/ai-sandbox/` inside WSL2 for best performance.

---

#### Step 3 — Copy files from Windows into WSL2 (if needed)

If you've already downloaded models or datasets to your Windows Downloads folder, here's how to move them to the faster Linux filesystem:

> 📦 **Run this in your WSL2 Terminal**

```bash
# Example: copy a model file from Windows Downloads into your WSL2 models folder
cp /mnt/c/Users/YourWindowsUsername/Downloads/llama-3.2.gguf ~/ai-sandbox/models/

# Verify the file is there
ls ~/ai-sandbox/models/
```

> 💡 `/mnt/c/` is how WSL2 refers to your Windows C: drive. So `C:\Users\John\Downloads` becomes `/mnt/c/Users/John/Downloads` in WSL2.

---

### ✅ Performance Tuning Checklist

Before moving on, confirm each item is done:

- [ ] `DOCKER_BUILDKIT=1` added to `~/.bashrc` (WSL2 Terminal)
- [ ] Cache mount (`--mount=type=cache`) added to Dockerfiles where `pip install` is used
- [ ] Storage Driver confirmed as `overlay2`
- [ ] `shm_size: '4gb'` added to ML training services in `docker-compose.yml`
- [ ] `~/ai-sandbox/` directory structure created on the Linux filesystem
- [ ] Model and dataset files stored under `~/ai-sandbox/`, not under `/mnt/c/`

---

## 6. Security Hardening

### Principle of Least Privilege

Never run containers as root when avoidable:

```dockerfile
# In your Dockerfile
RUN groupadd -r aiuser && useradd -r -g aiuser aiuser
USER aiuser
```

### Read-Only Filesystems

Mount model weights as read-only:

```yaml
volumes:
  - ~/ai-sandbox/models:/app/models:ro
```

### Disable Privilege Escalation

```yaml
security_opt:
  - no-new-privileges:true
cap_drop:
  - ALL
cap_add:
  - NET_BIND_SERVICE   # Only add back what you need
```

### Secrets Management — Do Not Use Environment Variables for Secrets

**Option A — Docker Secrets (Swarm/Compose):**

```yaml
secrets:
  api_key:
    file: ./secrets/api_key.txt

services:
  myservice:
    secrets:
      - api_key
```

**Option B — .env File (Development Only):**

```bash
# .env file (never commit to git)
ANTHROPIC_API_KEY=sk-...
HF_TOKEN=hf_...
```

```yaml
env_file:
  - .env
```

Add `.env` and `secrets/` to `.gitignore`:

```gitignore
.env
secrets/
*.key
*.pem
```

### Network Isolation

By default, all containers on the same Compose network can communicate. Isolate unrelated services:

```yaml
networks:
  inference_net:
    driver: bridge
    internal: true      # No external internet access
  api_net:
    driver: bridge
```

### Image Trust & Scanning

```bash
# Scan images for vulnerabilities before running
docker scout cves ollama/ollama:latest

# Only use official or verified images
docker pull python:3.11-slim          # ✅ Official
docker pull randomuser/pytorch:latest  # ⚠️ Unverified — avoid
```

### Limit Container Resources (Prevents Runaway Processes)

```yaml
deploy:
  resources:
    limits:
      cpus: '4.0'
      memory: 16G
    reservations:
      memory: 4G
```

---

## 7. Workload-Specific Container Setups

### A. LLM Inference (Ollama / llama.cpp)

#### Option A — Ollama (Easiest, GPU-aware)

```yaml
# docker-compose.yml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_models:/root/.ollama
    # NVIDIA GPU — uncomment below:
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]
    # AMD GPU — uncomment below:
    # devices:
    #   - /dev/dxg:/dev/dxg
    # volumes:
    #   - /usr/lib/wsl:/usr/lib/wsl:ro
    shm_size: '4gb'
    security_opt:
      - no-new-privileges:true
    networks:
      - inference_net

volumes:
  ollama_models:
```

Pull and run a model:

```bash
docker exec -it ollama ollama pull llama3.2
docker exec -it ollama ollama run llama3.2
```

#### Option B — llama.cpp (More Control, GGUF Models)

```dockerfile
# Dockerfile.llamacpp
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    build-essential cmake git libopenblas-dev \
    && rm -rf /var/lib/apt/lists/*

RUN git clone https://github.com/ggerganov/llama.cpp /opt/llama.cpp
WORKDIR /opt/llama.cpp

# Build with CPU + OpenBLAS
RUN cmake -B build -DLLAMA_BLAS=ON -DLLAMA_BLAS_VENDOR=OpenBLAS && \
    cmake --build build --config Release -j$(nproc)

RUN groupadd -r llama && useradd -r -g llama llama
USER llama

EXPOSE 8080
ENTRYPOINT ["/opt/llama.cpp/build/bin/llama-server"]
CMD ["--host", "0.0.0.0", "--port", "8080", "--model", "/models/model.gguf"]
```

```bash
docker build -f Dockerfile.llamacpp -t llamacpp-server .
docker run -d \
  --name llamacpp \
  -p 8080:8080 \
  -v ~/ai-sandbox/models:/models:ro \
  --shm-size=4g \
  llamacpp-server
```

---

### B. Python ML Training (PyTorch / TensorFlow)

#### Option A — PyTorch with NVIDIA CUDA

```dockerfile
# Dockerfile.pytorch
FROM pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime

WORKDIR /workspace

# Cache pip installs in build layer
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir \
    transformers datasets accelerate \
    wandb mlflow \
    jupyter ipykernel

# Non-root user
RUN useradd -m -u 1000 trainer
USER trainer

COPY --chown=trainer:trainer requirements.txt .
RUN pip install --user -r requirements.txt
```

```bash
docker build -f Dockerfile.pytorch -t pytorch-trainer .
docker run --rm -it \
  --gpus all \
  --shm-size=8g \
  -v ~/ai-sandbox/datasets:/data:ro \
  -v ~/ai-sandbox/outputs:/outputs \
  pytorch-trainer \
  python train.py
```

#### Option B — TensorFlow with DirectML (AMD / CPU)

```dockerfile
# Dockerfile.tensorflow-directml
FROM python:3.11-slim

RUN apt-get update && apt-get install -y \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

RUN pip install tensorflow-cpu tensorflow-directml-plugin

RUN useradd -m -u 1000 trainer
USER trainer
WORKDIR /workspace
```

---

### C. API / Backend Services

```dockerfile
# Dockerfile.api
FROM python:3.11-slim

WORKDIR /app

# Install dependencies as a separate layer for cache efficiency
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

COPY --chown=nobody:nogroup . .
USER nobody

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

**FastAPI example `main.py`:**

```python
from fastapi import FastAPI
import httpx

app = FastAPI()

OLLAMA_URL = "http://ollama:11434"  # Internal Docker DNS

@app.get("/health")
async def health(): return {"status": "ok"}

@app.post("/generate")
async def generate(prompt: str):
    async with httpx.AsyncClient(timeout=60) as client:
        resp = await client.post(f"{OLLAMA_URL}/api/generate",
                                 json={"model": "llama3.2", "prompt": prompt})
    return resp.json()
```

---

### D. Jupyter Notebooks

```yaml
services:
  jupyter:
    image: jupyter/scipy-notebook:latest
    container_name: jupyter
    restart: unless-stopped
    ports:
      - "8888:8888"
    volumes:
      - ~/ai-sandbox/notebooks:/home/jovyan/work
      - ~/ai-sandbox/datasets:/home/jovyan/data:ro
      - ~/ai-sandbox/models:/home/jovyan/models:ro
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - JUPYTER_TOKEN=changeme-use-strong-token
    deploy:
      resources:
        limits:
          memory: 8G
    security_opt:
      - no-new-privileges:true
    networks:
      - api_net
```

> 🔒 **Security Note:** Always set `JUPYTER_TOKEN` or `JUPYTER_PASSWORD`. Never expose port 8888 directly to a public network.

Access at: `http://localhost:8888?token=changeme-use-strong-token`

---

### E. IIS-Hosted Services & Apps

IIS runs natively on **Windows containers**, not Linux containers. Docker Desktop on Windows 11 supports switching between Linux and Windows container modes.

#### Switch to Windows Container Mode

Right-click the Docker Desktop tray icon → **Switch to Windows containers**.

> ⚠️ You cannot run Linux and Windows containers simultaneously on the same Docker daemon. Use **separate Compose profiles** or a reverse proxy (nginx/Traefik) to bridge them.

#### Option A — IIS in Windows Container (Native)

```dockerfile
# Dockerfile.iis
FROM mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022

# Copy your web app
COPY ./MyWebApp C:/inetpub/wwwroot/MyWebApp

# Configure IIS site via PowerShell
RUN powershell -NoProfile -Command \
    Import-Module WebAdministration; \
    New-WebSite -Name 'AIApp' -Port 80 \
    -PhysicalPath 'C:\inetpub\wwwroot\MyWebApp' \
    -ApplicationPool 'DefaultAppPool'

EXPOSE 80
```

```bash
docker build -f Dockerfile.iis -t my-iis-app .
docker run -d -p 8081:80 --name iis-app my-iis-app
```

#### Option B — Bridge IIS + Linux Containers via Reverse Proxy

Run your IIS app natively on Windows (or in a Windows container) and expose your Linux AI services. Use **nginx** as a reverse proxy to unify them under one port:

```nginx
# nginx.conf
upstream ollama {
    server localhost:11434;
}

upstream iis_app {
    server host.docker.internal:8081;  # Points to Windows host IIS
}

server {
    listen 443 ssl;

    location /api/ai/ {
        proxy_pass http://ollama/;
    }

    location /app/ {
        proxy_pass http://iis_app/;
    }
}
```

`host.docker.internal` resolves to your Windows host IP from inside Linux containers, allowing Linux containers to call your IIS service.

---

## 8. Compose: Full Sandbox Stack

Save as `docker-compose.yml` in `~/ai-sandbox/`:

```yaml
version: "3.9"

services:

  # ── LLM Inference ──────────────────────────────────────
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_models:/root/.ollama
    shm_size: '4gb'
    deploy:
      resources:
        limits:
          memory: 20G
        # Uncomment for NVIDIA GPU:
        # reservations:
        #   devices:
        #     - driver: nvidia
        #       count: all
        #       capabilities: [gpu]
    security_opt:
      - no-new-privileges:true
    networks:
      - inference_net

  # ── API Backend ────────────────────────────────────────
  api:
    build:
      context: ./api
      dockerfile: Dockerfile.api
    container_name: ai-api
    restart: unless-stopped
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - ollama
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
    security_opt:
      - no-new-privileges:true
    networks:
      - inference_net
      - api_net

  # ── Jupyter Notebook ───────────────────────────────────
  jupyter:
    image: jupyter/scipy-notebook:latest
    container_name: jupyter
    restart: unless-stopped
    ports:
      - "8888:8888"
    volumes:
      - ~/ai-sandbox/notebooks:/home/jovyan/work
      - ~/ai-sandbox/datasets:/home/jovyan/data:ro
      - ~/ai-sandbox/models:/home/jovyan/models:ro
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - JUPYTER_TOKEN=${JUPYTER_TOKEN}
    deploy:
      resources:
        limits:
          memory: 8G
    security_opt:
      - no-new-privileges:true
    networks:
      - api_net

  # ── ML Trainer (run on-demand, not always-on) ──────────
  trainer:
    build:
      context: ./trainer
      dockerfile: Dockerfile.pytorch
    container_name: ml-trainer
    profiles:
      - training                  # Only starts with: docker compose --profile training up
    volumes:
      - ~/ai-sandbox/datasets:/data:ro
      - ~/ai-sandbox/outputs:/outputs
    shm_size: '8gb'
    deploy:
      resources:
        limits:
          memory: 24G
    security_opt:
      - no-new-privileges:true
    networks:
      - inference_net

networks:
  inference_net:
    driver: bridge
  api_net:
    driver: bridge

volumes:
  ollama_models:
    driver: local
```

Start the default stack:

```bash
cd ~/ai-sandbox
docker compose up -d
```

Start with ML training profile:

```bash
docker compose --profile training up -d
```

---

## 9. Networking Best Practices

| Scenario | Recommendation |
|----------|---------------|
| Container → Container | Use Docker service names as hostnames (`http://ollama:11434`) |
| Container → Windows Host | Use `host.docker.internal` |
| External → Container | Bind only to `127.0.0.1` for dev (`127.0.0.1:8000:8000`) |
| Production-like | Add Traefik or nginx reverse proxy with TLS |
| IIS ↔ Linux containers | Use nginx proxy with `host.docker.internal` upstream |

Restrict port exposure to localhost during experimentation:

```yaml
ports:
  - "127.0.0.1:11434:11434"  # ✅ Local only
  # - "11434:11434"           # ⚠️ Exposed to all interfaces
```

---

## 10. Persistent Storage Strategy

```
~/ai-sandbox/
├── models/          ← Model weights (GGUF, HuggingFace, etc.) — mount :ro
├── datasets/        ← Training/eval datasets — mount :ro
├── outputs/         ← Training checkpoints, results — mount :rw
├── notebooks/       ← Jupyter notebooks — mount :rw
├── api/             ← API service source code
├── trainer/         ← ML trainer source code
├── secrets/         ← API keys (never commit to git)
├── .env             ← Environment variables (never commit to git)
└── docker-compose.yml
```

Use **named volumes** for database/cache data that containers manage themselves (e.g., `ollama_models`). Use **bind mounts** for source code and data you manage manually.

---

## 11. Monitoring & Resource Limits

### Live Resource Usage

```bash
# All containers — live stats
docker stats

# Single container
docker stats ollama
```

### Check Logs

```bash
docker compose logs -f ollama
docker compose logs -f --tail=100 api
```

### Portainer (Optional GUI Dashboard)

```bash
docker volume create portainer_data
docker run -d \
  --name portainer \
  --restart=always \
  -p 127.0.0.1:9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access at `https://localhost:9443`.

### Prune Unused Resources Regularly

```bash
# Remove stopped containers, unused networks, dangling images
docker system prune -f

# Also remove unused volumes (⚠️ careful — this deletes data)
docker system prune --volumes -f

# Remove unused images only
docker image prune -a -f
```

---

## 12. Useful Commands Cheat Sheet

```bash
# ── Stack Management ───────────────────────────────────────
docker compose up -d                    # Start all services (detached)
docker compose down                     # Stop and remove containers
docker compose restart ollama           # Restart a single service
docker compose --profile training up -d # Start with training profile

# ── Inspection ─────────────────────────────────────────────
docker ps                               # List running containers
docker compose ps                       # Compose service status
docker stats                            # Live CPU/RAM usage
docker logs -f <container>              # Stream container logs
docker exec -it <container> bash        # Shell into container

# ── GPU ────────────────────────────────────────────────────
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
docker exec -it ollama ollama ps        # Running LLM models

# ── Cleanup ────────────────────────────────────────────────
docker system df                        # Disk usage by Docker
docker system prune -f                  # Remove all unused resources
docker volume ls                        # List volumes

# ── WSL2 ───────────────────────────────────────────────────
wsl --shutdown                          # Restart WSL2 (reloads .wslconfig)
wsl -l -v                               # List WSL distros and versions
```

---

> **Last Updated:** March 2026  
> **Tested On:** Windows 11 23H2 · Docker Desktop 4.x · WSL2 (Ubuntu 22.04)

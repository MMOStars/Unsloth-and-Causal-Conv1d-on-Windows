
# 🦥 Mastering Unsloth & Causal-Conv1d on Windows

### A Technical Guide for "Bleeding Edge" AI Fine-Tuning 
**Environment:** Windows 10/11 | CUDA 13.0+ | Python 3.12 | RTX 30/40 Series (8.6)

---

## 📖 Introduction
Running advanced AI architectures like **LiquidAI (LFM)** or **Mamba** on Windows is notoriously difficult because they rely on custom CUDA kernels (`causal-conv1d`) that do not provide official Windows wheels. This guide details the process of manually compiling these kernels, handling CUDA 13.0 architecture conflicts, and setting up **Unsloth Studio** in a restricted Windows environment.

---

## 🛠 Phase 1: The Foundation (Core Prerequisites)

Do not use experimental Python versions (like 3.14). **Standardize on Python 3.12.**

### 1. Manual Toolchain (Bypassing Winget)
If the Windows Package Manager (`winget`) is unavailable or broken, install these manually:
*   **[Git for Windows](https://git-scm.com/)**: Essential for cloning repos.
*   **[CMake](https://cmake.org/download/)**: Required for C++ builds. **Crucial:** Select "Add to PATH" during installation.
*   **[CUDA Toolkit 12.4 or 13.0+](https://developer.nvidia.com/cuda-downloads)**: While 12.4 is more common, 13.0 works with the hacks below.
*   **[Visual Studio 2019/2022 Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)**: You must install the **"Desktop development with C++"** workload.

---

## 🚀 Phase 2: Installing Unsloth Studio

Unsloth Studio provides a powerful UI for fine-tuning. If the official installer fails due to `winget` being missing, use this "Session Alias" trick in PowerShell:

### 1. The Winget Alias Trick
```powershell
# Temporarily map the hidden winget path to a command
function winget { & "C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_..._x64__8wekyb3d8bbwe\winget.exe" $args }

# Run the official installer
irm https://unsloth.ai/install.ps1 | iex
```

### 2. Manual Environment Setup
If your system has multiple Python versions (e.g., 3.14) that conflict with Unsloth, always point directly to the Studio's internal environment:
```powershell
& "C:\Users\<USER>\.unsloth\studio\unsloth_studio\Scripts\python.exe" -m unsloth studio setup
```

---

## 🏗 Phase 3: Compiling Causal-Conv1d for Windows

Standard `pip install causal-conv1d` will fail on Windows. We must build from source using a specific community patch.

### 1. Prepare the Environment
Open the **x64 Native Tools Command Prompt for VS 2019** (or 2022) as Administrator. This sets the mandatory C++ compiler paths.

### 2. Clone and Patch
```cmd
git clone https://github.com/Dao-AILab/causal-conv1d.git
cd causal-conv1d
git fetch origin pull/58/head:windows_patch
git checkout windows_patch
```

### 3. The "CUDA 13.0" Compatibility Hack
CUDA 13.0+ removes support for older architectures (Maxwell). You must edit `setup.py` to prevent `nvcc fatal: Unsupported gpu architecture 'compute_53'`.

1. Open `setup.py` in a text editor.
2. Locate the `cc_flag` list (around line 160).
3. **Delete** all arches except your card (e.g., `86` for RTX 30/40 series).
4. The code block should look exactly like this:
```python
        cc_flag.append("-gencode")
        cc_flag.append("arch=compute_86,code=sm_86")
```
*Note: Ensure the indentation is exactly 8 spaces.*

### 4. The Final Build
Execute these commands within the `causal-conv1d` folder:
```cmd
set TORCH_CUDA_ARCH_LIST=8.6
set DISTUTILS_USE_SDK=1
set CAUSAL_CONV1D_FORCE_BUILD=TRUE

pip install . --no-build-isolation
```

---

## 💡 Phase 4: Hardware Constraints & VRAM Management

### The 6GB VRAM "Wall"
Even with successful software installation, hardware limitations remain. Windows has high VRAM overhead.

*   **Model Comparison:**
    *   `Llama-3.2-1B-bnb-4bit`: Fits easily in ~3GB VRAM. **Highly Recommended for 6GB cards.**
    *   `LiquidAI/LFM2.5-VL`: Requires ~11GB VRAM for training. 

### Optimization Tips for 6GB GPUs:
If you experience `CUDA Out of Memory` (OOM) errors:
1.  **Lower Max Sequence Length:** Set to `512` or `1024`.
2.  **Lower LoRA Rank (R):** Set to `16` or `8`.
3.  **Enable 4-bit Quantization:** Always ensure `load_in_4bit=True` is active.

---

## 🛠 Troubleshooting Summary

| Error | Fix |
| :--- | :--- |
| `ModuleNotFoundError: No module named 'torch'` | Building with isolation. Use `pip install . --no-build-isolation`. |
| `Unsupported gpu architecture 'compute_53'` | CUDA 13 incompatibility. Prune the arch list in `setup.py` to only `86`. |
| `IndentationError` in `setup.py` | Python is space-sensitive. Ensure your `cc_flag` edits match the surrounding code's 8-space indent. |
| `DISTUTILS_USE_SDK` not set | You are using a Native Tools prompt but didn't set the flag: `set DISTUTILS_USE_SDK=1`. |

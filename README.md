The Ultimate Guide: Installing Unsloth and Causal-Conv1d on Windows (CUDA 13.0+ & Python 3.12)
Introduction
While the AI world primarily lives on Linux, many developers prefer Windows for local fine-tuning. However, models like LiquidAI (LFM) or Mamba require custom CUDA kernels (causal-conv1d) that lack official Windows wheels. This guide walks through how to manually bridge the gap, handle "Bleeding Edge" versioning, and compile these kernels from source.
Phase 1: The Foundation (Core Prerequisites)
Before touching Unsloth, your system environment must be precisely configured.
1. Python Selection
Avoid experimental versions (like Python 3.14). Standardize on Python 3.12.
Ensure it is added to your System PATH.
If you have multiple versions, use a Virtual Environment (venv) to isolate 3.12.
2. Manual Toolchain (Bypassing Winget)
If Windows Package Manager (winget) is unavailable, install these manually:
Git for Windows: Essential for cloning repositories.
CMake: Required for building C++ components. (Select "Add to PATH" during install).
CUDA Toolkit 12.4 or 13.0: Industry standard is 12.4, but 13.0 works with the hacks below.
Visual Studio 2019/2022 Build Tools: You MUST install the "Desktop development with C++" workload.
Phase 2: Installing Unsloth Studio
Unsloth Studio provides a powerful UI for fine-tuning. If the official installer fails due to environment quirks, use this "Manual Injection" method:
Open PowerShell as Administrator.
The Winget Alias Trick: If the installer demands winget, you can "trick" it by aliasing your direct winget.exe path:
code
Powershell
function winget { & "C:\Path\To\Your\winget.exe" $args }
irm https://unsloth.ai/install.ps1 | iex
Ensure your environment is set to 3.12. If the installer creates a 3.12 venv but your system defaults to a newer Python, always run commands via the absolute path:
code
Powershell
& "C:\Users\YourUser\.unsloth\studio\unsloth_studio\Scripts\python.exe" -m unsloth studio setup
Phase 3: Compiling Causal-Conv1d (The Windows Patch)
This is the most critical step for non-standard models (like LiquidAI). Standard PyPI installs will fail on Windows.
1. Set up the Build Environment
Open the x64 Native Tools Command Prompt for VS 2019 (or 2022). This is vital as it sets the correct C++ compiler paths.
2. Prepare the Source
We use a specific community patch that adds Windows support:
code
Cmd
git clone https://github.com/Dao-AILab/causal-conv1d.git
cd causal-conv1d
git fetch origin pull/58/head:windows_patch
git checkout windows_patch
3. The "CUDA 13.0" Compatibility Hack
CUDA 13.0+ removes support for older GPUs (Maxwell/compute_53). You must edit setup.py to prevent compiler errors:
Open setup.py.
Locate the cc_flag architecture list.
Delete all older architectures (53, 62, 70, etc.).
Add only your specific architecture (e.g., arch=compute_86,code=sm_86 for RTX 30/40 series).
4. Build and Install
Set the environment flags to manage the Windows SDK and bypass isolation:
code
Cmd
set TORCH_CUDA_ARCH_LIST=8.6
set DISTUTILS_USE_SDK=1
set CAUSAL_CONV1D_FORCE_BUILD=TRUE

pip install . --no-build-isolation
Phase 4: Hardware Constraints & VRAM Management
Even with the software fixed, Windows has a higher VRAM overhead than Linux.
The 6GB VRAM "Wall"
If you are using an RTX 3060 (6GB) or similar:
Model Choice Matters: Standard models like Llama-3.2-1B or Qwen-2.5-1.5B fit perfectly.
LiquidAI/LFM: These models currently require high VRAM (~11GB+). If you get a CUDA Out of Memory error, you must:
Lower Max Sequence Length to 512.
Use 4-bit Quantization (bitsandbytes).
Lower LoRA Rank (R) to 8 or 16.
Troubleshooting Summary
ModuleNotFoundError: 'torch': You are building with "build isolation." Use --no-build-isolation and ensure torch is in your active venv.
IndentationError in setup.py: Python is space-sensitive. Ensure your cc_flag edits match the surrounding indentation exactly.
Unsupported GPU Architecture: CUDA 13.0+ doesn't like compute_53. Manually prune the architecture list in setup.py to match your card.

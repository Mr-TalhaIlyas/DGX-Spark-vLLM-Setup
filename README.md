# vLLM Setup for NVIDIA DGX Spark (Blackwell GB10) - Extended Patch

A hardened, one-command installation of vLLM specifically tuned for NVIDIA DGX Spark systems with Grace-Blackwell (GB10 / `sm_121`) GPUs. 

This fork extends the excellent base installer by [eelbaz](https://github.com/eelbaz/dgx-spark-vllm-setup) to address critical dependency and build isolation failures encountered during the `flashinfer-python` and `vllm` C++ compilation phases on isolated enterprise environments.

## Motivation & Fixes
The standard `vllm` pip wheels do not support the Blackwell architecture natively, requiring a from-source compilation using PyTorch Nightly and custom Triton branches. However, building these extensions often fails due to strict `setuptools` formatting rules and PEP-517 build isolation conflicts.

This repository resolves:
1. **`flashinfer-python` License Parsing Bug:** Bypasses the strict `project.license` validation crash by temporarily downgrading `setuptools`.
2. **`vllm` Editable Build Crash:** Resolves the `ModuleNotFoundError: No module named 'torch'` and `error: invalid command 'bdist_wheel'` errors by injecting the missing build tools (`wheel`, `ninja`, `cmake`) and disabling build isolation.

## System Requirements
* **Hardware:** NVIDIA DGX Spark with GB10 GPU (Blackwell `sm_121`)
* **OS:** Ubuntu 22.04+ (tested on Linux ARM64)
* **CUDA:** 13.0 or later 

## Installation Guide

### Step 1: Run the Base Installer
First, run the automated setup script to build the isolated Python environment and pull the Blackwell-specific PyTorch and Triton dependencies. *Note: This script will intentionally fail at the very end when trying to build `flashinfer` and `vllm`.*

```bash
curl -fsSL [https://raw.githubusercontent.com/YOUR_GITHUB_USERNAME/dgx-spark-vllm-setup/main/install.sh](https://raw.githubusercontent.com/YOUR_GITHUB_USERNAME/dgx-spark-vllm-setup/main/install.sh) | bash -s -- --install-dir ~/vllm-spark

```

### Step 2: Patch `flashinfer-python`

Activate the newly created virtual environment and temporarily downgrade `setuptools` to bypass the `pyproject.toml` license validation error.

```bash
cd ~/vllm-spark
source .vllm/bin/activate

uv pip install "setuptools<70.0.0"
uv pip install flashinfer-python==0.4.1

```

### Step 3: Finalize the vLLM Build

To compile the remaining vLLM CUDA extensions, we must install the fundamental C++ build wrappers into our environment and force the package manager to use them (disabling PEP-517 isolation).

```bash
uv pip install wheel ninja cmake
cd vllm
uv pip install -e . --no-build-isolation

```

*(The compilation will take roughly 5–15 minutes depending on your system load).*

## Usage

Once installed, you can launch heavily quantized FP8 or AWQ models using the provided wrapper script, which automatically applies the optimized Blackwell configurations.

```bash
cd ~/vllm-spark
source vllm_env.sh

# Launch the Qwen3 80B FP8 model on port 8000
./vllm-serve.sh "Qwen/Qwen3-Next-80B-A3B-Instruct-FP8" 8000

```

## Contributing

If you are running this on a different DGX configuration or encounter new Triton PTX assembler errors, please fork the repository, use a feature branch, and submit a Pull Request.

## Acknowledgments

* Base installation scripts by the [eelbaz/dgx-spark-vllm-setup](https://github.com/eelbaz/dgx-spark-vllm-setup) project.
* The vLLM and Triton communities for maintaining bleeding-edge hardware support.

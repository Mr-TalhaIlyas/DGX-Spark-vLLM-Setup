

This repository and guide resolve the following:
1. **C++ Linker / FP8 Crash:** Compiles specific `cutlass_scaled_mm` kernels for both `10.0a` (required by internal vLLM bindings) and `12.1a` (required for GB10 execution).
2. **Tokenizer Initialization Bug:** Patches `tokenizer.py` to gracefully handle the missing `all_special_tokens_extended` attribute in the newest Hugging Face transformers library.
3. **Large Model Download Crash:** Patches a duplicate `disable=True` argument bug in vLLM's `tqdm` wrapper and utilizes a safe, non-async local download method for massive 80GB+ checkpoints.

## System Requirements
* **Hardware:** NVIDIA DGX Spark with GB10 GPU (Blackwell `sm_121`)
* **OS:** Ubuntu 22.04+ (ARM64)
* **CUDA:** 13.0 or later



### Step 1: Base Environment Setup
Ensure your virtual environment is active. If you used the automated installer that crashed at the end, activate its environment:
```bash
cd ~/data/vllm-spark
source .vllm/bin/activate

```

### Step 2: Export Blackwell Hardware Flags

Before compiling the vLLM C++ backend, you must explicitly declare the target architectures. Failing to do this results in `NotImplementedError: No compiled cutlass_scaled_mm` when running FP8 models.

```bash
cd vllm
rm -rf build/  # Clean any previous broken builds

export TORCH_CUDA_ARCH_LIST="10.0a;12.1a"
export VLLM_USE_FLASHINFER_MXFP4_MOE=1
export TRITON_PTXAS_PATH=/usr/local/cuda/bin/ptxas

```

### Step 3: Compile vLLM

Compile the custom CUDA extensions natively. This bypasses PEP-517 build isolation to ensure it uses the specific dependencies in your environment. *(This takes 10-20 minutes).*

```bash
uv pip install -e . --no-build-isolation

```

### Step 4: Patch vLLM Python Source Bugs

Because we installed vLLM in editable mode (`-e`), we can instantly patch two critical bugs in the source code using `sed`.

**1. Fix Tokenizer Attribute Bug (`transformers_utils/tokenizer.py`):**

```bash
sed -i 's/= tokenizer.all_special_tokens_extended/= getattr(tokenizer, "all_special_tokens_extended", tokenizer.all_special_tokens)/g' /home/aim_lab/data/vllm-spark/vllm/vllm/transformers_utils/tokenizer.py

```

**2. Fix TQDM Download Crash (`model_loader/weight_utils.py`):**

```bash
sed -i 's/super().__init__(*args, \*\*kwargs, disable=True)/kwargs.pop("disable", None); super().__init__(*args, \*\*kwargs, disable=True)/g' /home/aim_lab/data/vllm-spark/vllm/vllm/model_executor/model_loader/weight_utils.py

```

---

## Model Deployment

### Step 1: Safe Local Download

To avoid async timeout crashes when pulling massive models, disable Hugging Face transfer acceleration and download the model to a local directory first.

```bash
# Authenticate if downloading a gated model
hf auth login

# Download directly to disk
HF_HUB_ENABLE_HF_TRANSFER=0 hf download Qwen/Qwen3-Next-80B-A3B-Instruct-FP8 --local-dir /home/aim_lab/data/vllm-spark/qwen80b-local

```

### Step 2: Launch the Inference Server

Point vLLM directly to the local folder. We use `--enforce-eager` to ensure PyTorch does not encounter dynamic shape compilation timeouts.

```bash
vllm serve /home/aim_lab/data/vllm-spark/qwen80b-local \
  --served-model-name "Qwen3-80B" \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \
  --trust-remote-code \
  --enforce-eager

```

### Step 3: Test the API

In a new terminal window, send a standard OpenAI-compatible request to your local server:

```bash
curl -X POST [http://127.0.0.1:8000/v1/chat/completions](http://127.0.0.1:8000/v1/chat/completions) \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3-80B",
    "messages": [
      {"role": "system", "content": "You are a helpful AI assistant."},
      {"role": "user", "content": "Explain the advantages of FP8 quantization on Blackwell architectures."}
    ],
    "max_tokens": 200
  }'

```


---
name: llama-tune
description: Tune llama-server for optimal performance and GPU utilization. Analyzes GPU VRAM, model architecture (dense/MoE), calculates VRAM budget, and generates launch command for maximum tok/s.
argument-hint: <model.gguf> [--ctx SIZE] [--slots N] [--port PORT] [--launch]
allowed-tools: Bash Read Glob Grep
effort: high
---

## System Profile (auto-detected)

### GPU
!`nvidia-smi --query-gpu=index,name,memory.total,memory.free,memory.used,pci.bus_id --format=csv 2>/dev/null || echo "NO_NVIDIA_GPU"`

### CPU
!`nproc 2>/dev/null && lscpu 2>/dev/null | grep -E "Model name|Core\(s\) per socket|Socket\(s\)|Thread\(s\) per core" || echo "UNKNOWN_CPU"`

### RAM
!`free -h 2>/dev/null | head -2 || echo "UNKNOWN_RAM"`

### llama-server binary
!`command -v llama-server 2>/dev/null || find ~/sandbox -maxdepth 5 -name llama-server -type f 2>/dev/null | head -1 || echo "NOT_FOUND"`

### Available llama.cpp tools
!`dirname "$(command -v llama-server 2>/dev/null || find ~/sandbox -maxdepth 5 -name llama-server -type f 2>/dev/null | head -1)" 2>/dev/null | xargs -I{} ls {} 2>/dev/null | grep -E "llama-gguf|llama-cli" || echo "NO_TOOLS"`

---

## Arguments

**Raw arguments:** $ARGUMENTS

Parse as follows:
- **First positional arg** (required): path to GGUF model file. If relative or just a filename, search the current directory and `~/sandbox/models/`.
- `--ctx N`: target context length in tokens. Default: auto-detect from model, capped to what fits in VRAM.
- `--slots N`: parallel inference slots. Default: 1.
- `--port N`: serve port. Default: 8080.
- `--server PATH`: llama-server binary path. Default: auto-detected above.
- `--launch`: start the server immediately after generating the command.
- `--mmproj PATH`: vision model multimodal projector file.

---

## Tuning Procedure

You are an expert at tuning llama-server for maximum tok/s on consumer hardware. Follow these steps precisely. Refer to [reference.md](reference.md) for all formulas, flag details, and GPU specs.

### Step 1: Inspect the Model

Use `llama-gguf <model_path> r n` (sibling binary to llama-server) to extract architecture info.

Determine from tensor names:
- **MoE detection**: presence of `ffn_gate_exps` or `ffn_up_exps` tensors means MoE
- **Layer count**: count unique `blk.N` indices
- **KV heads / head dim**: from attention tensor shapes
- **Sliding window**: from metadata keys or architecture family
- **Expert count**: from tensor dimensions (if MoE)
- **Expert weight size per layer**: sum `size` of all `*exps*` tensors per `blk.N`

If `llama-gguf` is unavailable, infer from filename:
- `A{N}B` suffix = MoE with N billion active params (e.g., `26B-A4B` = 26B total, 4B active)
- Use the file size as the model weight estimate
- Use the reference.md architecture database for known model families

Record: `model_size_bytes`, `n_layers`, `n_kv_heads`, `head_dim`, `is_moe`, `has_sliding_window`, `expert_bytes_per_layer` (if MoE).

### Step 2: Calculate VRAM Budget

Using formulas from [reference.md](reference.md):

1. **Weights VRAM** = `model_file_size` (full GPU offload)
2. **KV cache VRAM** = calculated per architecture (account for GQA, sliding window)
   - Start with `-ctk q8_0 -ctv q4_0` as the default
3. **Overhead** = 500 MB (single GPU) or 800 MB (multi-GPU)
4. **Per-slot cost** = KV cache is multiplied by `n_slots`

### Step 3: Pick Strategy

Try these in priority order until it fits in available VRAM (target 95% utilization):

**Priority 1 - Full GPU offload:**
If `weights + kv_cache(q8_0/q4_0) + overhead <= available_vram * 0.95`:
  - `-ngl 999 -fa on -ctk q8_0 -ctv q4_0`
  - This is the best case. Done.

**Priority 2 - Aggressive KV quantization:**
If Priority 1 doesn't fit, try `-ctk q4_0 -ctv q4_0`.

**Priority 3 - Reduce context:**
If the user didn't specify `--ctx`, reduce context to the largest power-of-2 that fits.

**Priority 4 - MoE expert offloading (MoE models only):**
This is the key optimization for MoE models that don't fit in VRAM:
1. Calculate `non_expert_vram` (overhead + KV cache + non-expert weights on GPU). Estimate non-expert weights as `model_file_size - total_expert_bytes`.
2. `vram_for_experts = available_vram - non_expert_vram - 500MB_safety`
3. `layers_on_gpu = floor(vram_for_experts / expert_bytes_per_layer)`
4. `layers_to_offload = n_layers - layers_on_gpu`
5. Use `-ncmoe <layers_to_offload>` to offload that many layers' experts to CPU
6. Always pair with `--no-mmap` and appropriate `-t` (physical cores)
7. If `layers_on_gpu` is 0 or negative, fall back to `-ncmoe` with all layers (equivalent to `-cmoe`)

**Priority 5 - Dense partial offload (dense models only):**
Reduce `-ngl` until weights + KV fit. Each removed layer saves approximately `model_file_size / (n_layers + 1)`.

**Priority 6 - Suggest smaller quant:**
If even full CPU offload doesn't work, tell the user they need a smaller quantization and suggest which one would fit.

### Step 4: Build the Command

Always include these flags:

```
-m <model_path>                    # model file
-c <context>                       # context length
-ngl 999                           # full layer offload (adjust for dense partial)
-fa on                             # flash attention (required for KV quant)
-ctk <q8_0|q4_0> -ctv <q4_0|q8_0> # KV cache quantization
-t <physical_cores>                # CPU threads = physical cores only
-b 2048 -ub 512                    # batch sizes (defaults)
--host 0.0.0.0 --port <port>      # network binding
```

Conditionally add:
- `-ncmoe N` or `-cmoe`: MoE expert CPU offloading
- `--no-mmap`: when using CPU offloading
- `-np N`: if slots > 1
- `--mmproj <path>`: if vision model projector specified

For MoE with CPU offload, use `-b 4096 -ub 4096` (larger batches amortize PCIe transfer).

### Step 5: Present Results

Output a clean summary:

**1. Model summary table:**

| Property | Value |
|----------|-------|
| Model | (name) |
| Architecture | Dense / MoE (X experts, Y active) |
| File size | X GB |
| Quantization | (detected) |
| Layers | N |

**2. VRAM breakdown:**

| Component | Size | Notes |
|-----------|------|-------|
| Model weights (GPU) | X GB | (or partial if offloaded) |
| KV cache | X GB | (type, context, slots) |
| Expert weights (CPU) | X GB | (if MoE offload) |
| Overhead | ~0.5 GB | |
| **Total GPU** | **X / Y GB** | **(Z% utilization)** |

**3. The full command** in a code block, ready to copy-paste.

**4. Expected performance notes:**
- For full GPU: "Expect high tok/s, limited by memory bandwidth"
- For MoE offload: "Expert transfer over PCIe is the bottleneck. ~X/30 layers on GPU means ~Y% of expert compute avoids PCIe."
- For dense partial offload: "N layers on CPU will bottleneck generation speed"

If `--launch` was passed, or the user confirms, start the server in the background and verify it loaded correctly by checking VRAM usage and waiting for the "listening" log line.

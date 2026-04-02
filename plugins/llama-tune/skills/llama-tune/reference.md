# llama-server Tuning Reference

## KV Cache VRAM Formulas

### Standard (full attention, all layers)

```
kv_bytes_per_token = 2 * n_kv_heads * head_dim * n_layers * bytes_per_element
total_kv_bytes = kv_bytes_per_token * context_length * n_parallel_slots
```

### With Sliding Window Attention (SWA)

Models like Gemma 2/3/4, Mistral, Cohere alternate between SWA and full (global) attention layers:

```
swa_kv = n_swa_layers * min(context_length, sliding_window_size) * 2 * n_kv_heads_swa * head_dim_swa * bpe
full_kv = n_global_layers * context_length * 2 * n_kv_heads_global * head_dim_global * bpe
total_kv = (swa_kv + full_kv) * n_parallel_slots
```

SWA layers cap their KV cache at the window size regardless of total context. This can dramatically reduce VRAM at long contexts.

### Bytes per Element by KV Quant Type

| Type | Bytes/element | VRAM vs f16 | Quality impact |
|------|--------------|-------------|----------------|
| `f16` | 2.0 | baseline | none |
| `q8_0` | 1.0625 | ~47% savings | negligible |
| `q4_0` | 0.5625 | ~72% savings | small, measurable at long context |

**Recommendation:** Default to `-ctk q8_0 -ctv q4_0`. K cache is more sensitive to quantization than V cache. Only use `-ctk q4_0` when VRAM is critical.

**Important:** KV cache quantization requires `-fa on` (flash attention). Without it, the flags are silently ignored and f16 is used.

---

## Flag Reference

### Core Performance Flags

| Flag | Default | Description |
|------|---------|-------------|
| `-ngl N` | 0 | GPU layers. Use 999 for full offload. Each layer costs ~`file_size/(n_layers+1)` |
| `-fa on` | auto | Flash attention. Always enable. Required for KV quant. |
| `-ctk TYPE` | f16 | K cache quant: f16, q8_0, q4_0 |
| `-ctv TYPE` | f16 | V cache quant: f16, q8_0, q4_0 |
| `-c N` | 4096 | Context size in tokens. Scales KV cache linearly. |
| `-t N` | auto | CPU threads. Set to physical core count (NOT logical/hyperthreaded). |
| `-b N` | 2048 | Logical batch size for prompt processing. Higher = faster prefill. |
| `-ub N` | 512 | Physical batch (micro-batch) per GPU kernel. 512 is generally optimal. |
| `-np N` | 1 | Parallel slots. Each reserves full KV cache. Multiplies KV VRAM by N. |
| `--no-mmap` | off | Load model to RAM instead of memory-mapping. Use with CPU offload. |
| `--mlock` | off | Lock model in RAM, prevent swapping. Use when RAM is tight with other apps. |

### MoE-Specific Flags

| Flag | Description |
|------|-------------|
| `-cmoe` | Move ALL expert weights to CPU RAM. Simplest MoE offload. |
| `-ncmoe N` | Move the first N layers' expert weights to CPU. Allows partial GPU placement. |
| `-ot "pattern=DEVICE"` | Fine-grained tensor placement. Pattern matches tensor names (glob-style). |

**MoE offload examples:**
```bash
# All experts to CPU (simplest, slowest)
-cmoe

# First 10 layers' experts to CPU, rest stay on GPU
-ncmoe 10

# Fine-grained: specific layers' experts to CPU
-ot "blk.2?.ffn_*exps*=CPU"

# Everything except experts on GPU (equivalent to -cmoe + -ngl 999)
-ngl 999 -ot "exps=CPU"
```

### Batch Size Guidance

| Scenario | `-b` | `-ub` | Notes |
|----------|------|-------|-------|
| Single user, short context | 2048 | 512 | Default, good for most cases |
| Single user, long context | 4096 | 512 | Larger batches help prefill speed |
| MoE with CPU expert offload | 4096 | 4096 | Match b/ub to amortize PCIe transfers |
| Multi-user serving (4+ slots) | 8192 | 512 | Allows batching across slots |
| VRAM-constrained | 512 | 256 | Reduces compute buffer allocation |

---

## VRAM Overhead Estimates

| Component | Typical Size |
|-----------|-------------|
| CUDA context / driver | 300-500 MB |
| Compute buffers | 200-500 MB (scales with batch size) |
| Scratch/temp buffers | 50-200 MB |
| **Total overhead** | **500 MB - 1 GB** |

Use 500 MB as default estimate. For 70B+ models or large batch sizes, use 800 MB - 1 GB.

---

## MoE Tuning Strategy

### Why MoE Models Are Special

MoE models have `total_params >> active_params_per_token`. For example:
- Mixtral 8x7B: 46.7B total, 12.9B active (2/8 experts per layer)
- Gemma 4 26B-A4B: 25.2B total, 3.8B active (8/128 experts per layer)
- Qwen3.5 35B-A3B: 35B total, 3B active

**All expert weights must be loaded** (total params determine weight VRAM), but **compute scales with active params** (determines generation speed when fully on GPU).

### Expert Offloading Decision Flow

```
1. Can all weights + KV cache fit on GPU?
   YES → Full offload (-ngl 999). Best performance.
   NO  → Continue to step 2.

2. With experts offloaded, do non-expert weights + KV cache fit?
   YES → Calculate how many layers' experts fit in remaining VRAM.
         Use -ncmoe (n_layers - layers_that_fit) for partial offload.
   NO  → Full expert offload (-cmoe) + check if KV cache fits.
         If still no: reduce context, use q4_0 KV, reduce slots.
```

### Calculating Partial Expert Offload

```
non_expert_vram = overhead + kv_cache + (model_file_size - total_expert_bytes)
remaining_for_experts = gpu_vram_total - non_expert_vram - safety_margin(500MB)
layers_on_gpu = floor(remaining_for_experts / expert_bytes_per_layer)
layers_to_offload = n_layers - layers_on_gpu
```

To measure `total_expert_bytes` and `expert_bytes_per_layer`:
```bash
llama-gguf <model> r n 2>&1 | grep "ffn_.*exps" | awk '{
    match($0, /blk\.([0-9]+)/, a); match($0, /size = ([0-9]+)/, s)
    layers[a[1]] += s[1]
} END { for (l in layers) printf "Layer %d: %.1f MB\n", l, layers[l]/1048576 }'
```

**Note:** The `size` field from `llama-gguf` is in bytes for the quantized tensor on disk. This equals the VRAM footprint when loaded.

### PCIe Bandwidth Impact

When experts are on CPU, each generated token requires transferring active expert weights over PCIe:

```
transfer_per_token = n_active_experts * expert_size_per_layer * n_offloaded_layers
```

| PCIe Gen | x16 Bandwidth | Impact |
|----------|--------------|--------|
| 3.0 | ~12.5 GB/s practical | Severe bottleneck |
| 4.0 | ~25 GB/s practical | Moderate bottleneck |
| 5.0 | ~50 GB/s practical | Manageable |

Partial offloading (keeping some layers on GPU) proportionally reduces this bottleneck.

---

## Dense Model Tuning Strategy

### Full GPU Offload

Best case. Check: `model_file_size + kv_cache + overhead <= gpu_vram`

### Partial Layer Offload

When the model doesn't quite fit, offload a few layers to CPU:
```
vram_for_weights = available_vram - kv_cache - overhead
max_layers = floor(vram_for_weights / (model_file_size / (n_layers + 1)))
```

Use `-ngl <max_layers>`. Performance degrades proportionally with layers on CPU because every token must round-trip through CPU layers.

**Rule of thumb:** If more than 20% of layers must be on CPU, consider stepping down the quantization instead.

### VRAM Priority (what to sacrifice first)

1. KV quant: f16 → q8_0 → q4_0 (nearly free quality cost)
2. Context length: reduce if not fully needed
3. Parallel slots: reduce from N to 1
4. Model quant: step down one level (e.g., Q6_K → Q5_K_M → Q4_K_M)
5. Layer offload: partial CPU (-ngl reduction) — last resort

---

## Quantization Reference

### Bits Per Weight (BPW) and Typical File Sizes

| Quant | BPW | 7B Size | 13B Size | 27-34B Size | 70B Size |
|-------|-----|---------|----------|-------------|----------|
| IQ2_XXS | 2.06 | 1.8 GB | 3.4 GB | 7 GB | 18 GB |
| IQ3_XXS | 3.06 | 2.7 GB | 5 GB | 10.5 GB | 26 GB |
| Q3_K_M | 3.91 | 3.4 GB | 6.4 GB | 13.5 GB | 34 GB |
| Q4_K_S | 4.58 | 4.0 GB | 7.5 GB | 15.5 GB | 40 GB |
| **Q4_K_M** | **4.85** | **4.2 GB** | **7.9 GB** | **16.5 GB** | **42 GB** |
| Q5_K_M | 5.69 | 5.0 GB | 9.3 GB | 19.5 GB | 49 GB |
| Q6_K | 6.57 | 5.7 GB | 10.7 GB | 22.5 GB | 57 GB |
| Q8_0 | 8.50 | 7.4 GB | 14 GB | 29 GB | 74 GB |
| F16 | 16.0 | 14 GB | 26 GB | 55 GB | 140 GB |

**Note:** "UD" (Unsloth Dynamic) quants use mixed precision — important tensors at higher quant, less important at lower. File sizes are similar to the named quant level but quality is better than uniform quantization at the same size.

### Quality Tiers

| Tier | Quants | Use Case |
|------|--------|----------|
| Near-lossless | Q8_0, Q6_K | Maximum quality, if VRAM allows |
| High quality | Q5_K_M, Q5_K_S | Excellent balance, recommended default |
| Good quality | Q4_K_M, Q4_K_S | Community sweet spot, minimal perceptible loss |
| Acceptable | Q3_K_M, Q3_K_L | Noticeable loss, fine for large models (70B+) |
| Low quality | IQ3_XXS, Q2_K | Last resort, only for very large models |

---

## GPU VRAM Database

### NVIDIA Consumer GPUs

| GPU | VRAM | Mem BW (GB/s) | PCIe | Notes |
|-----|------|--------------|------|-------|
| RTX 3060 | 12 GB | 360 | 4.0 | Great value for 7-13B |
| RTX 3060 Ti | 8 GB | 448 | 4.0 | VRAM limited |
| RTX 3070 | 8 GB | 448 | 4.0 | Fast but only 8GB |
| RTX 3070 Ti | 8 GB | 504 | 4.0 | |
| RTX 3080 | 10 GB | 760 | 4.0 | Awkward 10GB |
| RTX 3080 Ti | 12 GB | 912 | 4.0 | Good bandwidth |
| RTX 3090 | 24 GB | 936 | 4.0 | Workhorse for large models |
| RTX 3090 Ti | 24 GB | 1,008 | 4.0 | |
| RTX 4060 | 8 GB | 272 | 4.0 | Budget, VRAM limited |
| RTX 4060 Ti 8GB | 8 GB | 288 | 4.0 | |
| RTX 4060 Ti 16GB | 16 GB | 288 | 4.0 | Great VRAM, lower BW |
| RTX 4070 | 12 GB | 504 | 4.0 | Solid mid-range |
| RTX 4070 Super | 12 GB | 504 | 4.0 | |
| RTX 4070 Ti | 12 GB | 504 | 4.0 | |
| RTX 4070 Ti Super | 16 GB | 672 | 4.0 | Sweet spot for 13-30B |
| RTX 4080 | 16 GB | 717 | 4.0 | |
| RTX 4080 Super | 16 GB | 736 | 4.0 | |
| RTX 4090 | 24 GB | 1,008 | 4.0 | Consumer king |
| RTX 5070 | 12 GB | 672 | 5.0 | Blackwell |
| RTX 5070 Ti | 16 GB | 896 | 5.0 | |
| RTX 5080 | 16 GB | 960 | 5.0 | |
| RTX 5090 | 32 GB | 1,792 | 5.0 | New king, runs 70B Q4 |

### AMD Consumer GPUs

| GPU | VRAM | Mem BW (GB/s) | Notes |
|-----|------|--------------|-------|
| RX 7600 | 8 GB | 288 | Budget |
| RX 7700 XT | 12 GB | 432 | |
| RX 7800 XT | 16 GB | 480 | Good value |
| RX 7900 GRE | 16 GB | 576 | |
| RX 7900 XT | 20 GB | 800 | Unique 20GB |
| RX 7900 XTX | 24 GB | 960 | Best AMD consumer |
| RX 9070 XT | 16 GB | 650 | RDNA 4 |

---

## Known Model Architectures

Use this to infer architecture when GGUF metadata inspection isn't available.

### MoE Models

| Family | Total / Active | Experts (total/active) | Layers | KV Heads | Head Dim | SWA |
|--------|---------------|----------------------|--------|----------|----------|-----|
| Gemma 4 26B-A4B | 25.2B / 3.8B | 128 / 8 (+1 shared) | 30 | 8 swa, 2 global | 256 swa, 512 global | 1024 |
| Mixtral 8x7B | 46.7B / 12.9B | 8 / 2 | 32 | 8 | 128 | 4096 |
| Mixtral 8x22B | 141B / 39B | 8 / 2 | 56 | 8 | 128 | 4096 |
| Qwen3.5 35B-A3B | 35B / 3B | 128 / 8 | 64 | 4 | 128 | no |
| DeepSeek-V2 Lite | 15.7B / 2.4B | 64 / 6 | 27 | varies | varies | no |
| DeepSeek-V3 | 671B / 37B | 256 / 8 | 61 | 128 | 128 | no |
| DBRX | 132B / 36B | 16 / 4 | 40 | 8 | 128 | no |
| Nemotron-3-Nano 30B-A3B | 30B / 3B | 128 / 8 | 40 | 8 | 128 | no |

### Dense Models (common)

| Family | Params | Layers | KV Heads | Head Dim | SWA |
|--------|--------|--------|----------|----------|-----|
| Llama 3.x 8B | 8B | 32 | 8 (GQA) | 128 | no |
| Llama 3.x 70B | 70B | 80 | 8 (GQA) | 128 | no |
| Gemma 2/3 27B | 27B | 46 | 16 | 128 | 4096 (alternating) |
| Gemma 2/3 9B | 9B | 42 | 8 | 128 | 4096 (alternating) |
| Mistral 7B | 7.2B | 32 | 8 | 128 | 4096 |
| Qwen 2.5 72B | 72B | 80 | 8 | 128 | no |
| Qwen 2.5 32B | 32B | 64 | 8 | 128 | no |
| Phi-3 14B | 14B | 40 | 8 | 128 | no |
| Command R+ | 104B | 64 | 8 | 128 | no |

---

## Thread Count Detection

Physical cores only. Hyperthreading hurts llama.cpp performance.

```bash
# Linux
lscpu | awk '/Core\(s\) per socket/{cores=$NF} /Socket\(s\)/{socks=$NF} END{print cores*socks}'

# macOS
sysctl -n hw.physicalcpu

# Fallback: nproc / 2 (assumes SMT/HT enabled)
```

---

## Quick VRAM Budget Calculator

For a quick estimate without deep model inspection:

```
Total VRAM needed =
  model_file_size_gb                          (weights)
  + (n_layers * n_kv_heads * head_dim * ctx * 2 * bpe_k) / 1e9   (K cache)
  + (n_layers * n_kv_heads * head_dim * ctx * 2 * bpe_v) / 1e9   (V cache)
  + 0.5                                       (overhead in GB)
  ) * n_parallel_slots                         (slot multiplier for KV only)
```

If model architecture is unknown, rough estimate:
```
kv_cache_gb ≈ model_file_size_gb * 0.05 * (ctx / 4096)   (very rough, for GQA models)
```

This underestimates for MHA models and overestimates for heavy GQA, but works as a starting point.

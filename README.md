# Bailey Marketplace

Personal Claude Code plugin marketplace.

## Installation

Add this marketplace to Claude Code:

```
/plugin marketplace add <owner>/bailey-claude-marketplace
```

Then install individual plugins:

```
/plugin install <plugin-name>@bailey-marketplace
```

## Available Plugins

| Plugin | Description | Source |
|--------|-------------|--------|
| `claude-mem` | Persistent memory system for Claude Code. Captures tool usage, compresses observations with AI, and re-injects relevant context into future sessions. | External ([thedotmack/claude-mem](https://github.com/thedotmack/claude-mem)) |
| `llama-tune` | Tune llama-server for optimal performance and GPU utilization. Supports dense and MoE models. | In-repo |

## Plugin Notes

### claude-mem

Persistent memory across Claude Code sessions. Automatically captures everything Claude does, compresses it with AI, and provides continuity in future sessions.

**Auto-installed dependencies** (installed on first run):
- [Bun](https://bun.sh/) >= 1.0.0
- [uv](https://github.com/astral-sh/uv) (Python package manager, for Chroma)
- SQLite 3

**Runtime:**
- Runs a worker service on `localhost:37777`
- Web viewer UI available at `http://localhost:37777`
- Stores data in `~/.claude-mem/`

**Install:**
```
/plugin install claude-mem@bailey-marketplace
```

### llama-tune

Tunes llama-server (llama.cpp) launch parameters for maximum tok/s on your hardware. Auto-detects GPU VRAM, CPU cores, and system RAM. Inspects GGUF model files to determine architecture (dense vs MoE), then calculates optimal flags including KV cache quantization, flash attention, expert offloading (MoE), and partial GPU layer placement.

**Features:**
- Auto-detects system hardware (GPU, CPU, RAM)
- Inspects GGUF model architecture via `llama-gguf`
- Supports both dense and Mixture-of-Experts models
- Calculates partial MoE expert offloading for maximum VRAM utilization
- Generates ready-to-run llama-server commands
- Includes reference database of GPU specs, model architectures, and quantization levels

**Skill:** `/llama-tune <model.gguf> [--ctx SIZE] [--slots N] [--port PORT] [--launch]`

**Install:**
```
/plugin install llama-tune@bailey-marketplace
```

## Adding Plugins

In-repo plugins go in the `plugins/` directory. External plugins are referenced by source in `.claude-plugin/marketplace.json`.

## License

MIT

# USER MANUAL

## What This Project Is

`llama.cpp` is a local LLM inference project written primarily in C/C++. It builds command-line tools and a local OpenAI-compatible server, and it also includes Python helper scripts for model conversion tasks.

This copy was installed from:

- Upstream repository: `https://github.com/ggml-org/llama.cpp`
- Local directory: `/Users/cding/Workspace/ThirdParty/llama.cpp`

This checkout was cloned with `--depth 1` for reliability during install. If you need full Git history, run:

```bash
cd /Users/cding/Workspace/ThirdParty/llama.cpp
mise run full-history
```

## Quick Start

### 1. Go to the project

```bash
cd /Users/cding/Workspace/ThirdParty/llama.cpp
```

### 2. Build it

For the normal local build:

```bash
mise run build
```

On Apple Silicon, the default build uses the project's normal backend selection, which typically includes Metal support.

### 3. Run a model from Hugging Face

```bash
HF_MODEL=ggml-org/gemma-3-1b-it-GGUF mise run run-hf
```

### 4. Run the saved Gemma coder model

```bash
mise run run-gemma-4-12b-coder-fable5
```

This launches `yuxinlu1/gemma-4-12B-coder-fable5-composer2.5-v1-GGUF` directly, without needing to set `HF_MODEL`.

### 5. Run Gemma 4 26B

```bash
mise run run-gemma-4-26b
```

This launches `unsloth/gemma-4-26B-A4B-it-GGUF`.

### 6. Run Gemma 4 31B

```bash
mise run run-gemma-4-31b
```

This launches `unsloth/gemma-4-31B-it-GGUF`.

### 7. Or run a local GGUF model

```bash
MODEL=/absolute/path/to/model.gguf mise run run-cli
```

### 5. Start the local API server

```bash
HF_MODEL=ggml-org/gemma-3-1b-it-GGUF mise run serve-hf
```

The server is OpenAI-compatible, so many OpenAI-client tools can point to it as a local inference endpoint.

## Quick Reference

| Command | What it does | When to use it |
| --- | --- | --- |
| `mise run status` | Shows the Git status and configured remotes | Check whether you have local changes before updating |
| `mise run sync-source` | Fetches from `origin` and fast-forwards the checkout | Update to the latest upstream commit |
| `mise run full-history` | Expands the shallow clone into a full clone | Needed if you want complete Git history |
| `mise run build` | Builds the default release configuration | Normal day-to-day use |
| `mise run build-debug` | Builds a debug configuration | C/C++ debugging and development |
| `mise run build-static` | Builds static binaries/libraries | Packaging or static-link experiments |
| `mise run build-cpu` | Builds CPU-only with Metal disabled | CPU-only benchmarking or troubleshooting |
| `mise run build-cuda` | Builds with CUDA enabled | NVIDIA GPU builds on systems with CUDA |
| `mise run test` | Runs `ctest` on the default build | Verify the build after changes |
| `mise run python-sync` | Creates `.venv` and installs Python helper dependencies with `uv` | Use conversion and helper scripts |
| `MODEL=/path/to/model.gguf mise run run-cli` | Runs `llama-cli` with a local model | Basic local inference |
| `HF_MODEL=org/model mise run run-hf` | Runs `llama-cli` with a model downloaded from Hugging Face | Fastest way to try the project |
| `mise run run-gemma-4-12b-coder-fable5` | Runs `yuxinlu1/gemma-4-12B-coder-fable5-composer2.5-v1-GGUF` | Quick one-command launch for this saved model |
| `mise run run-gemma-4-26b` | Runs `unsloth/gemma-4-26B-A4B-it-GGUF` | Quick one-command launch for Gemma 4 26B |
| `mise run run-gemma-4-31b` | Runs `unsloth/gemma-4-31B-it-GGUF` | Quick one-command launch for Gemma 4 31B |
| `MODEL=/path/to/model.gguf mise run serve` | Runs the OpenAI-compatible server with a local model | Local API serving |
| `HF_MODEL=org/model mise run serve-hf` | Runs the OpenAI-compatible server with a Hugging Face model | Quick server setup |
| `mise run clean` | Removes build directories and local Python env/cache folders | Reset local build artifacts |

## Daily Operations

### Build from source

```bash
mise run build
```

This creates the default release build in `build/`.

### Run the CLI with a local model

```bash
MODEL=/absolute/path/to/model.gguf mise run run-cli
```

### Run the CLI with a model from Hugging Face

```bash
HF_MODEL=ggml-org/gemma-3-1b-it-GGUF mise run run-hf
```

### Run the saved Gemma coder model

```bash
mise run run-gemma-4-12b-coder-fable5
```

### Run Gemma 4 26B

```bash
mise run run-gemma-4-26b
```

### Run Gemma 4 31B

```bash
mise run run-gemma-4-31b
```

The first run may download model artifacts into the standard Hugging Face cache location.

### Start the local server

```bash
MODEL=/absolute/path/to/model.gguf mise run serve
```

Or:

```bash
HF_MODEL=ggml-org/gemma-3-1b-it-GGUF mise run serve-hf
```

## Python Helper Scripts

This repository contains Python utilities such as model-conversion scripts. Set them up with:

```bash
mise run python-sync
```

That command:

1. Creates `.venv/`
2. Uses `uv` to install the Python project dependencies declared by upstream

After that, you can run upstream Python scripts from the repo root, for example:

```bash
source .venv/bin/activate
python convert_hf_to_gguf.py --help
```

## Sync With The Official Git Repo

To pull the latest upstream changes:

```bash
mise run sync-source
```

If you need the full commit history first:

```bash
mise run full-history
mise run sync-source
```

## Manual Steps You May Still Need

### Models

`llama.cpp` does not ship model weights. You need to either:

- Point `MODEL` to a local `.gguf` file, or
- Use `HF_MODEL=...` so the binary downloads a supported model from Hugging Face

### Hugging Face authentication

Some models require Hugging Face authentication. If a model is gated, authenticate with your usual Hugging Face workflow before using `-hf`.

### Which Gemma repos these shortcuts use

The dedicated Gemma tasks currently point to:

- `yuxinlu1/gemma-4-12B-coder-fable5-composer2.5-v1-GGUF`
- `unsloth/gemma-4-26B-A4B-it-GGUF`
- `unsloth/gemma-4-31B-it-GGUF`

If you want these switched to different owners or quantizations, update the matching tasks in `mise.toml`.

### CUDA

The `build-cuda` task requires a working CUDA toolkit and NVIDIA-compatible environment. It is not expected to work on this Apple Silicon machine.

## File Layout

| Path | Purpose |
| --- | --- |
| `README.md` | Upstream overview and project scope |
| `docs/build.md` | Upstream build instructions |
| `docs/` | Additional upstream documentation |
| `build/` | Default release build output after `mise run build` |
| `.venv/` | Local Python environment created by `mise run python-sync` |
| `mise.toml` | Local task shortcuts added for this workspace |

## Troubleshooting

### Build fails during CMake configure

Make sure `cmake` is installed and visible in `PATH`.

### `llama-cli` or `llama-server` not found

Run:

```bash
mise run build
```

first. The binaries are created by the build step.

### A model download fails

Check network access, Hugging Face availability, and whether the model requires authentication.

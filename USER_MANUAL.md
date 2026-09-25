# USER MANUAL

| Field | Value |
| --- | --- |
| title | llama.cpp Local Workspace User Manual |
| language | en |
| format | markdown |
| version | 1.0 |
| status | Active |
| author | Not specified |
| owner | Not specified |
| audience | People who run llama.cpp models and embedding servers on this Mac |
| create-time | 2026-06-30T04:57:57-05:00 |
| last-modify-time | 2026-09-25T04:52:46-05:00 |
| keywords | llama.cpp, llama-server, llama-cli, GGUF, embeddings, bge-m3, nomic-embed, Qwen3-Embedding, Gemma 4, mise tasks, model cache, Hugging Face cache, sync upstream, server status, health check |

## 1. What This Project Is

`llama.cpp` runs large language models (LLMs) and embedding models on your own machine. You don't need a cloud service. It provides:

- **`llama-cli`**: a command-line chat with a model in your terminal.
- **`llama-server`**: a local web server that speaks the OpenAI API. Other programs (ChenWeb, scripts, tools) can send chat or embedding requests to it as if it were OpenAI.
- **Python helper scripts**: for example, converting Hugging Face models into the GGUF format that llama.cpp reads.

This copy was installed from:

- Upstream repository: `https://github.com/ggml-org/llama.cpp`
- Local directory: `/Users/cding/Workspace/ThirdParty/llama.cpp`

This copy is a **shallow clone** made with `--depth 1`, so it holds only recent Git history. It also carries a few **local additions** on top of upstream: this manual, `mise.toml`, and the shortcut tasks in it. Section 7 explains how to update upstream code without losing those additions.

### 1.1 Key terms

| Term | Meaning |
| --- | --- |
| GGUF | The model file format llama.cpp uses (`*.gguf`). |
| Chat model | A model you talk to, for example Gemma 4. |
| Embedding model | A model that turns text into a list of numbers (a vector) for search and similarity, for example bge-m3. It doesn't chat. |
| mise task | A named shortcut defined in `mise.toml`, run with `mise run <task>`. |
| Port | The number in `http://localhost:<port>` where a server listens. Each background server here has its own port. |

## 2. Quick Start

```bash
cd /Users/cding/Workspace/ThirdParty/llama.cpp
mise run build                 # build once (and again after every update)
mise run serve-bge-m3          # start the bge-m3 embedding server on :18083
curl -s http://localhost:18083/health   # prints {"status":"ok"} once it is ready
```

To chat with a model in the terminal instead:

```bash
mise run run-gemma-4-26b
```

The first time you use a model, llama.cpp downloads it. Large models can take many minutes. See Section 6 for where downloads go.

## 3. Quick Reference

### 3.1 Background servers (the ones other programs use)

Each task starts a server **in the background**. It keeps running after the command returns and after you close the terminal. It writes a log file and a PID file (the process number used to stop it).

| Start command | Model | Kind | URL | Stop command | Log file |
| --- | --- | --- | --- | --- | --- |
| `mise run serve-gemma-4-26b` | `unsloth/gemma-4-26B-A4B-it-GGUF` | Chat | `http://localhost:18080` | `mise run stop-gemma-4-26b` | `/tmp/llama-gemma-4-26b.log` |
| `mise run serve-nomic-embed` | `nomic-ai/nomic-embed-text-v2-moe-GGUF` (Q4_K_M) | Embedding | `http://localhost:18081` | No task yet. See Section 5.3 | `/tmp/llama-nomic-embed.log` |
| `mise run serve-qwen3-embedding-0-6b` | `Qwen/Qwen3-Embedding-0.6B-GGUF` (Q8_0) | Embedding | `http://localhost:18082` | No task yet. See Section 5.3 | `/tmp/llama-qwen3-embedding-0-6b.log` |
| `mise run serve-bge-m3` | `CompendiumLabs/bge-m3-gguf` (f16) | Embedding | `http://localhost:18083` | `mise run stop-bge-m3` | `/tmp/llama-bge-m3.log` |

All four servers accept up to 4 requests in parallel (`-np 4`).

### 3.2 All other tasks

| Command | What it does | When to use it |
| --- | --- | --- |
| `mise run status` | Shows the Git status and configured remotes | Check for local changes before updating |
| `mise run sync-source` | Fetches from `origin` and fast-forwards | **Doesn't work with this copy's current setup.** Use Section 7 instead |
| `mise run full-history` | Expands the shallow clone into a full clone | Needed only if you want complete Git history |
| `mise run build` | Builds the default release configuration into `build/` | Normal use; rerun after every update |
| `mise run build-debug` | Builds a debug configuration | C/C++ debugging and development |
| `mise run build-static` | Builds static binaries/libraries | Packaging or static-link experiments |
| `mise run build-cpu` | Builds CPU-only with Metal disabled | CPU-only benchmarking or troubleshooting |
| `mise run build-cuda` | Builds with CUDA enabled | NVIDIA GPU machines only (not this Mac) |
| `mise run test` | Runs `ctest` on the default build | Check the build after changes |
| `mise run python-sync` | Creates `.venv` and installs Python helper dependencies with `uv` | Before using conversion scripts |
| `MODEL=/path/to/model.gguf mise run run-cli` | Chats with a local GGUF file | You already have a `.gguf` file |
| `HF_MODEL=org/model mise run run-hf` | Downloads a Hugging Face model and chats with it | Fastest way to try a model |
| `mise run run-gemma-4-12b-coder-fable5` | Chats with `yuxinlu1/gemma-4-12B-coder-fable5-composer2.5-v1-GGUF` | One-command launch |
| `mise run run-gemma-4-26b` | Chats with `unsloth/gemma-4-26B-A4B-it-GGUF` (Metal GPU, 4 CPU threads) | One-command launch |
| `mise run run-gemma-4-31b` | Chats with `unsloth/gemma-4-31B-it-GGUF` | One-command launch |
| `MODEL=/path/to/model.gguf mise run serve` | Runs the server in the foreground with a local GGUF file | Quick test; stops when you press Ctrl-C |
| `HF_MODEL=org/model mise run serve-hf` | Runs the server in the foreground with a Hugging Face model | Quick test; stops when you press Ctrl-C |
| `mise run clean` | Removes build directories and the local Python env/caches | Reset build artifacts (you must rebuild afterward) |
| `mise run show-downloaded-models` | Shows all the downloaded models|

## 4. Daily Operations

### 4.1 Build from source

```bash
mise run build
```

This creates the release build in `build/`. The programs end up in `build/bin/` (`llama-cli`, `llama-server`, `llama-embedding`, and others). On Apple Silicon the build includes Metal GPU support.

Every mise task runs the programs from `build/bin/`, so rebuild whenever you update the source.

### 4.2 Chat in the terminal

```bash
mise run run-gemma-4-26b                              # saved shortcut
HF_MODEL=ggml-org/gemma-3-1b-it-GGUF mise run run-hf  # any Hugging Face GGUF repo
MODEL=/absolute/path/to/model.gguf mise run run-cli   # a .gguf file you already have
```

### 4.3 Use the bge-m3 embedding server

bge-m3 is a multilingual embedding model that handles Chinese and English well. It turns each piece of text into a list of 1024 numbers. Texts with similar meaning get similar numbers.

Start it:

```bash
mise run serve-bge-m3
```

What to expect:

- It prints the PID and `http://localhost:18083`, then returns right away. The server keeps loading in the background.
- Loading takes a few seconds. Until then, `/health` reports that the model is still loading.
- It accepts inputs of up to 8192 tokens (`-c 8192`).
- It uses bge-m3's default CLS pooling, which is the pooling bge-m3 needs for correct dense embeddings.

Try it:

```bash
curl -s http://localhost:18083/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input": "hello world", "model": "bge-m3"}'
```

The reply contains `"embedding": [ ... ]` with 1024 numbers.

Stop it:

```bash
mise run stop-bge-m3
```

The nomic (`:18081`) and Qwen3 (`:18082`) embedding servers work the same way. Only the port and model differ.

## 5. Checking Whether llama.cpp Is Running

### 5.1 Is any llama.cpp server running?

```bash
pgrep -fl llama-server
```

Each line is one running server, with its PID and the full command. The command shows which model and port it uses. No output means no server is running.

To see which ports are in use:

```bash
lsof -nP -iTCP -sTCP:LISTEN | grep llama
```

### 5.2 Is a specific server ready?

Ask its health endpoint. Use the port from the table in Section 3.1:

```bash
curl -s http://localhost:18083/health
```

| Result | Meaning |
| --- | --- |
| `{"status":"ok"}` | Running and ready for requests |
| An error saying the model is loading | Started, still loading the model. Wait and retry |
| `curl: (7) Failed to connect` | Not running on that port. Check its log file |

To see which model a server has loaded:

```bash
curl -s http://localhost:18083/v1/models
```

To watch what a server is doing, or why it failed to start:

```bash
tail -f /tmp/llama-bge-m3.log
```

### 5.3 Stopping a server that has no stop task

The nomic and Qwen3 servers have no stop task yet. Stop them with their PID files:

```bash
kill "$(cat /tmp/llama-nomic-embed.pid)" && rm /tmp/llama-nomic-embed.pid
kill "$(cat /tmp/llama-qwen3-embedding-0-6b.pid)" && rm /tmp/llama-qwen3-embedding-0-6b.pid
```

A PID file can be out of date, for example if the server crashed or the Mac restarted. In that case, find the real PID with `pgrep -fl llama-server` and run `kill <PID>`.

Background servers **don't restart after a reboot**. Start them again with their `serve-...` task.

## 6. Downloaded Models and the Model Cache

### 6.1 Where downloads go

When a task uses `-hf <repo>` (all the shortcut tasks do), llama.cpp downloads the model once and reuses it on later runs. This build stores downloads in the **standard Hugging Face cache**:

```
~/.cache/huggingface/hub/models--<owner>--<repo>/
```

It shares this folder with Python tools (docling, sentence-transformers, and others), so the cache holds more than llama.cpp models.

To use a different folder, set `LLAMA_CACHE=/some/folder` before running a task. llama.cpp also respects `HF_HUB_CACHE` and `HF_HOME`.

### 6.2 See what llama.cpp has downloaded

```bash
./build/bin/llama-server --cache-list
```

As of 2026-09-25 this lists:

| Model | Used by | Approx. size on disk |
| --- | --- | --- |
| `Qwen/Qwen3-Embedding-0.6B-GGUF:Q8_0` | `serve-qwen3-embedding-0-6b` | 610 MB |
| `CompendiumLabs/bge-m3-gguf:F16` | `serve-bge-m3` | 1.1 GB |
| `unsloth/gemma-4-26B-A4B-it-GGUF:Q4_K_M` | `run-gemma-4-26b`, `serve-gemma-4-26b` | 17 GB |
| `nomic-ai/nomic-embed-text-v2-moe-GGUF:Q4_K_M` | `serve-nomic-embed` | 328 MB |

Also as of that date:

- `yuxinlu1/gemma-4-12B-coder-fable5-composer2.5-v1-GGUF` has a 511 MB cache folder, but `--cache-list` doesn't list it and the folder has no `.gguf` file. The download looks incomplete. The next `run-gemma-4-12b-coder-fable5` run should download it again.
- `unsloth/gemma-4-31B-it-GGUF` hasn't been downloaded. The first `run-gemma-4-31b` run will download it.

To see sizes yourself:

```bash
du -sh ~/.cache/huggingface/hub/models--*GGUF* ~/.cache/huggingface/hub/models--*gguf*
```

### 6.3 The older cache folder

Older llama.cpp builds, including the Homebrew build (Section 8), stored downloads in a different folder:

```
~/Library/Caches/llama.cpp/
```

As of 2026-09-25 it holds about 1.8 GB: a second copy of bge-m3 f16, plus `bartowski/ATH-MaaS_OvisOCR2-GGUF` (the model file and its `mmproj` vision file). The current build doesn't read from this folder. If you no longer use the Homebrew build, you can delete this folder to save space.

### 6.4 Freeing space

To remove one model, delete its folder, for example:

```bash
rm -rf ~/.cache/huggingface/hub/models--unsloth--gemma-4-26B-A4B-it-GGUF
```

The model downloads again the next time a task uses it.

## 7. Syncing With the Latest Upstream Version

### 7.1 Why `mise run sync-source` doesn't work here

`sync-source` runs `git pull --ff-only`, which works only when:

1. the checkout is on a branch, and
2. you have no commits of your own.

This copy meets neither condition. As of 2026-09-25:

- It is in **detached HEAD** state: it isn't on any branch.
- It has **3 local commits** on top of upstream: `038d7e1` (workspace setup files), `52f5eae` (Gemma task), and `8aa7b1c` (Gemma 4 26B/31B tasks).
- It has **uncommitted edits** to `mise.toml` (the embedding server tasks) and `USER_MANUAL.md`.

The procedure below keeps your local additions and places them on top of the newest upstream code.

### 7.2 One-time setup: give your local work a branch

Run this once. After it, your commits live on a named branch and can't be lost by accident:

```bash
cd /Users/cding/Workspace/ThirdParty/llama.cpp
git switch -c workspace        # "workspace" holds your local commits
git add mise.toml USER_MANUAL.md
git commit -m "Add embedding server tasks and update user manual"
```

### 7.3 Each time you update

```bash
cd /Users/cding/Workspace/ThirdParty/llama.cpp
git status                     # must be clean; commit or `git stash` first
git fetch origin               # download the newest upstream commits
git rebase origin/master       # replay your local commits on top of them
mise run build                 # rebuild; the tasks run the new binaries
```

What to expect:

- `git rebase` usually finishes on its own. Your commits only touch `mise.toml` and `USER_MANUAL.md`, which upstream doesn't have.
- If it stops with a conflict, fix the listed files and run `git rebase --continue`, or run `git rebase --abort` to go back.
- After rebuilding, restart any running background servers so they use the new binaries: stop them, then run their `serve-...` task again.

To check that you are up to date:

```bash
git fetch origin && git log --oneline -1 origin/master && git log --oneline -4
```

The first line is the newest upstream commit. The following lines are your checkout, where your local commits should sit directly on top of it.

## 8. Two llama.cpp Builds on This Mac

As of 2026-09-25 there are two separate llama.cpp installations:

| | This project's build | Homebrew build |
| --- | --- | --- |
| Location | `ThirdParty/llama.cpp/build/bin/` | `/opt/homebrew/bin/` (release 8140) |
| Used by | Every mise task in this project | Typing `llama-server` or `llama-cli` directly in any terminal |
| Model cache | `~/.cache/huggingface/hub` | `~/Library/Caches/llama.cpp` |
| Updated by | Section 7 plus `mise run build` | `brew upgrade llama.cpp` |

They are different versions, so a model or option that works in one may fail in the other. Use the mise tasks, or call `./build/bin/llama-server` directly, to be sure you run this project's build. `llama-server --version` shows which build you are running.

To keep only one build, remove the Homebrew copy with `brew uninstall llama.cpp`. Afterward, `llama-server` isn't found by name outside this folder unless you add `build/bin/` to your `PATH`.

## 9. Python Helper Scripts

This repository includes Python utilities such as model-conversion scripts. Set them up with:

```bash
mise run python-sync
```

That command:

1. Creates `.venv/`
2. Uses `uv` to install the Python dependencies declared by upstream

After that, you can run upstream Python scripts from the repo root, for example:

```bash
source .venv/bin/activate
python convert_hf_to_gguf.py --help
```

## 10. Models, Access, and Hardware

### 10.1 Models

`llama.cpp` doesn't ship model weights. You need to either:

- Point `MODEL` to a local `.gguf` file, or
- Use `HF_MODEL=...` or a shortcut task so llama.cpp downloads a model from Hugging Face

### 10.2 Hugging Face authentication

Some models require Hugging Face authentication. If a model is gated, log in with your usual Hugging Face workflow before using `-hf`.

### 10.3 Which repos the shortcut tasks use

| Task | Repository |
| --- | --- |
| `run-gemma-4-12b-coder-fable5` | `yuxinlu1/gemma-4-12B-coder-fable5-composer2.5-v1-GGUF` |
| `run-gemma-4-26b`, `serve-gemma-4-26b` | `unsloth/gemma-4-26B-A4B-it-GGUF` |
| `run-gemma-4-31b` | `unsloth/gemma-4-31B-it-GGUF` |
| `serve-nomic-embed` | `nomic-ai/nomic-embed-text-v2-moe-GGUF`, file `nomic-embed-text-v2-moe.Q4_K_M.gguf` |
| `serve-qwen3-embedding-0-6b` | `Qwen/Qwen3-Embedding-0.6B-GGUF`, file `Qwen3-Embedding-0.6B-Q8_0.gguf` |
| `serve-bge-m3` | `CompendiumLabs/bge-m3-gguf`, file `bge-m3-f16.gguf` |

To switch owners, files, or quantizations, edit the matching task in `mise.toml`.

### 10.4 CUDA

The `build-cuda` task requires a working CUDA toolkit and an NVIDIA GPU. It isn't expected to work on this Apple Silicon machine.

## 11. File Layout

| Path | Purpose |
| --- | --- |
| `README.md` | Upstream overview and project scope |
| `docs/build.md` | Upstream build instructions |
| `docs/` | Additional upstream documentation |
| `build/` | Release build output after `mise run build` |
| `.venv/` | Local Python environment created by `mise run python-sync` |
| `mise.toml` | Local task shortcuts added for this workspace |
| `USER_MANUAL.md` | This manual (local addition) |
| `/tmp/llama-*.log`, `/tmp/llama-*.pid` | Logs and process IDs of background servers |
| `~/.cache/huggingface/hub/` | Downloaded models (Section 6) |

## 12. Troubleshooting

### 12.1 Build fails during CMake configure

Make sure `cmake` is installed and on your `PATH`.

### 12.2 `llama-cli` or `llama-server` not found

Run `mise run build` first. The build creates the programs.

### 12.3 A model download fails

Check network access, Hugging Face availability, and whether the model requires authentication.

### 12.4 A background server says "started" but doesn't answer

The `serve-...` tasks report "started" as soon as the process launches, before the model loads. Check `/health` (Section 5.2) and read the log file. Common causes:

- **Port already in use.** Another copy is already running. Check with `pgrep -fl llama-server`.
- **Model still downloading.** The first run of a large model downloads it before loading.
- **Out of memory.** Running several large models at once can exhaust RAM.

### 12.5 `git pull` says "You are not currently on a branch"

Follow Section 7 instead of `mise run sync-source`.

## Change Log

| Version | Time | Author | Reason | Summary |
| --- | --- | --- | --- | --- |
| 1.0 | 2026-09-25T04:52:46-05:00 | Claude (on request) | Manual lacked the embedding servers, status checks, cache details, and a working sync procedure | Added the background server table and bge-m3 usage (`serve-bge-m3`/`stop-bge-m3`); added "Checking Whether llama.cpp Is Running"; corrected the model cache location (`~/.cache/huggingface/hub`, not `~/Library/Caches/llama.cpp`) and listed cached models; replaced the sync instructions with a branch-plus-rebase procedure (the `sync-source` task doesn't work in detached HEAD with local commits); added the two-builds section; added metadata |
| 1.0 | 2026-06-30T04:57:57-05:00 | Not specified | Initial workspace setup | Created manual with build, run, sync, and Gemma shortcut instructions |

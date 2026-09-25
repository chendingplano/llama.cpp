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
| last-modify-time | 2026-09-25T06:08:59-05:00 |
| keywords | llama.cpp, llama-server, llama-cli, GGUF, embeddings, bge-m3, nomic-embed, Qwen3-Embedding, mise tasks, model cache, Hugging Face cache, sync upstream, fork, remote, jj, Jujutsu, rebase, bookmark, server status, health check |

## 1. What This Project Is

`llama.cpp` runs large language models (LLMs) and embedding models on your own machine. You don't need a cloud service. It provides:

- **`llama-cli`**: a command-line chat with a model in your terminal.
- **`llama-server`**: a local web server that speaks the OpenAI API. Other programs (ChenWeb, scripts, tools) can send chat or embedding requests to it as if it were OpenAI.
- **Python helper scripts**: for example, converting Hugging Face models into the GGUF format that llama.cpp reads.

This copy was installed from:

- Upstream repository: `https://github.com/ggml-org/llama.cpp` (remote `upstream`)
- Your fork: `https://github.com/chendingplano/llama.cpp.git` (remote `origin`, where your commits are pushed)
- Local directory: `/Users/cding/Workspace/ThirdParty/llama.cpp`

This copy is a **shallow clone** made with `--depth 1`, so it holds only recent Git history. It is managed with **jj** (Jujutsu) on top of Git. It also carries a few **local additions** on top of upstream: this manual, `mise.toml`, and the shortcut tasks in it. Section 7 explains how to update upstream code without losing those additions.

### 1.1 Key terms

| Term | Meaning |
| --- | --- |
| GGUF | The model file format llama.cpp uses (`*.gguf`). |
| Chat model | A model you talk to. No chat model is set up as a shortcut; use `run-hf` or `serve-hf` with any Hugging Face GGUF repo. |
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
HF_MODEL=ggml-org/gemma-3-1b-it-GGUF mise run run-hf
```

The first time you use a model, llama.cpp downloads it. Large models can take many minutes. See Section 6 for where downloads go.

## 3. Quick Reference

### 3.1 Background servers (the ones other programs use)

Each task starts a server **in the background**. It keeps running after the command returns and after you close the terminal. It writes a log file and a PID file (the process number used to stop it).

| Start command | Model | Kind | URL | Stop command | Log file |
| --- | --- | --- | --- | --- | --- |
| `mise run serve-nomic-embed` | `nomic-ai/nomic-embed-text-v2-moe-GGUF` (Q4_K_M) | Embedding | `http://localhost:18081` | No task yet. See Section 5.3 | `/tmp/llama-nomic-embed.log` |
| `mise run serve-qwen3-embedding-0-6b` | `Qwen/Qwen3-Embedding-0.6B-GGUF` (Q8_0) | Embedding | `http://localhost:18082` | No task yet. See Section 5.3 | `/tmp/llama-qwen3-embedding-0-6b.log` |
| `mise run serve-bge-m3` | `CompendiumLabs/bge-m3-gguf` (f16) | Embedding | `http://localhost:18083` | `mise run stop-bge-m3` | `/tmp/llama-bge-m3.log` |

All three servers accept up to 4 requests in parallel (`-np 4`).

### 3.2 All other tasks

| Command | What it does | When to use it |
| --- | --- | --- |
| `mise run status` | Shows the Git status and configured remotes | Quick look. `jj st` and `jj log` give a clearer picture (Section 7.6) |
| `mise run sync-source` | Runs `git pull --ff-only` | **Doesn't work in this jj-managed copy.** Use Section 7.4 instead |
| `mise run full-history` | Expands the shallow clone into a full clone | Needed only if you want complete history. Run `jj log` afterwards so jj picks up the new commits |
| `mise run build` | Builds the default release configuration into `build/` | Normal use; rerun after every update |
| `mise run build-debug` | Builds a debug configuration | C/C++ debugging and development |
| `mise run build-static` | Builds static binaries/libraries | Packaging or static-link experiments |
| `mise run build-cpu` | Builds CPU-only with Metal disabled | CPU-only benchmarking or troubleshooting |
| `mise run build-cuda` | Builds with CUDA enabled | NVIDIA GPU machines only (not this Mac) |
| `mise run test` | Runs `ctest` on the default build | Check the build after changes |
| `mise run python-sync` | Creates `.venv` and installs Python helper dependencies with `uv` | Before using conversion scripts |
| `MODEL=/path/to/model.gguf mise run run-cli` | Chats with a local GGUF file | You already have a `.gguf` file |
| `HF_MODEL=org/model mise run run-hf` | Downloads a Hugging Face model and chats with it | Fastest way to try a model |
| `MODEL=/path/to/model.gguf mise run serve` | Runs the server in the foreground with a local GGUF file | Quick test; stops when you press Ctrl-C |
| `HF_MODEL=org/model mise run serve-hf` | Runs the server in the foreground with a Hugging Face model | Quick test; stops when you press Ctrl-C |
| `mise run clean` | Removes build directories and the local Python env/caches | Reset build artifacts (you must rebuild afterward) |
| `mise run show-downloaded-models` | Shows all the downloaded models (same as `llama-server --cache-list`) | See what's in the model cache (Section 6.2) |

## 4. Daily Operations

### 4.1 Build from source

```bash
mise run build
```

This creates the release build in `build/`. The programs end up in `build/bin/` (`llama-cli`, `llama-server`, `llama-embedding`, and others). On Apple Silicon the build includes Metal GPU support.

Every mise task runs the programs from `build/bin/`, so rebuild whenever you update the source.

### 4.2 Chat in the terminal

```bash
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
| `nomic-ai/nomic-embed-text-v2-moe-GGUF:Q4_K_M` | `serve-nomic-embed` | 328 MB |

Gemma 4 models were removed from this setup on 2026-09-25 (tasks and downloads).

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
rm -rf ~/.cache/huggingface/hub/models--nomic-ai--nomic-embed-text-v2-moe-GGUF
```

The model downloads again the next time a task uses it.

## 7. Syncing With Upstream and Your Fork (with jj)

This copy is managed with **jj** (Jujutsu), which works alongside Git in the same folder. Use `jj` for everything that changes history: committing, fetching, rebasing, pushing. Use raw `git` only to look around.

### 7.1 Where the code comes from: two remotes

A **remote** is a copy of the repository on GitHub. This copy talks to two:

| Remote | URL | Role |
| --- | --- | --- |
| `upstream` | `https://github.com/ggml-org/llama.cpp` | The official llama.cpp project. You **fetch** new upstream code from here. Only its `master` branch is fetched |
| `origin` | `https://github.com/chendingplano/llama.cpp.git` | **Your fork** (public). You **push** your local commits here, so they are backed up on GitHub |

In jj, a bookmark on a remote is written `<bookmark>@<remote>`. So `master@upstream` is the newest official commit, and `master@origin` is your fork's `master`.

jj is configured for this copy (in `jj config list --repo`) so that:

- `jj git fetch` fetches from **both** remotes (`git.fetch = ["upstream", "origin"]`).
- `jj git push` pushes to `origin`, your fork (jj's default).
- `trunk()` means `master@upstream`.

### 7.2 How your local work is organised

`jj log` shows your commits stacked on top of upstream, newest at the top:

| What | jj name | Meaning |
| --- | --- | --- |
| Working copy | `@` | Where your current edits go. jj saves edits into it automatically; there is no staging step |
| Your latest local work | bookmark `main` | The top of your local commits. This is what you push to your fork |
| Earlier local commits | bookmark `workspace`, and unnamed ones below it | Embedding server tasks, Gemma tasks (since removed), workspace setup files |
| Upstream | `master@upstream` | The newest official commit jj has fetched |

A **bookmark** is jj's name for a branch. Your local commits only touch `mise.toml` and `USER_MANUAL.md`, which upstream doesn't have.

The local bookmark `master` is left over from the original clone and doesn't move when you fetch. Use `master@upstream` in the commands below.

### 7.3 Why `mise run sync-source` doesn't work here

`sync-source` runs `git pull --ff-only`. That needs Git to be on a branch with no commits of your own. With jj, Git is always in "detached HEAD" state (not on any branch), and you do have local commits, so the command fails. Use the steps below instead.

### 7.4 Each time you update

```bash
cd /Users/cding/Workspace/ThirdParty/llama.cpp
jj st                                    # see whether you have unsaved edits (see below)
jj git fetch                             # download new commits from upstream and your fork
jj rebase -b main -o master@upstream     # move your local commits on top of the newest upstream
mise run build                           # rebuild; the tasks run the new binaries
jj git push -b main                      # back up your rebased commits to your fork
```

If `jj st` shows edits you want to keep, save them as a commit first and move `main` up to it:

```bash
jj commit -m "Describe your change"
jj bookmark set main -r @-
```

What to expect:

- `jj rebase` never stops halfway. It moves all your local commits, plus your working copy, onto the new upstream in one step.
- If a commit conflicts with upstream, jj still finishes the rebase and marks that commit as conflicted in `jj log`. Run `jj resolve` or edit the files, then check with `jj st`.
- After a rebase, `jj git push -b main` replaces `main` on your fork with the rebased commits. That's expected: your fork's `main` always mirrors your local `main`.
- If anything goes wrong locally, `jj undo` reverts the last jj command exactly. It can't undo a push that already reached GitHub.
- After rebuilding, restart any running background servers so they use the new binaries: stop them, then run their `serve-...` task again.

Your fork's `master` doesn't update by itself. It isn't needed for the steps above, but to keep it level with upstream, use **Sync fork** on the fork's GitHub page, or run `gh repo sync chendingplano/llama.cpp`.

### 7.5 Checking that you are up to date

```bash
jj git fetch
jj log -r 'master@upstream::'
```

This shows the newest upstream commit at the bottom, with your local commits (ending in `main`) and the working copy `@` above it. If your commits don't sit directly on `master@upstream`, run the rebase from Section 7.4.

To list only your own local commits:

```bash
jj log -r '::@ ~ ::master@upstream'
```

To check whether your fork has your latest `main`, run `jj bookmark list main`. If it shows `main*`, or a `main@origin` line that differs from `main`, you have local commits that aren't pushed yet.

### 7.6 Everyday jj commands

| Command | What it does |
| --- | --- |
| `jj st` | Shows edits in the working copy |
| `jj log` | Shows recent commits and bookmarks |
| `jj diff` | Shows what you changed in the working copy |
| `jj commit -m "..."` | Saves the working copy as a commit and starts a fresh one |
| `jj bookmark set main -r @-` | Moves `main` to your latest commit |
| `jj git fetch` | Downloads new commits from upstream and your fork |
| `jj git push -b main` | Uploads `main` to your fork |
| `jj git remote list` | Shows the two remotes |
| `jj undo` | Reverts the last jj command |

Don't use `git commit`, `git pull`, `git rebase` or `git switch` here. Mixing them with jj can leave duplicate or orphaned commits.

## 8. Two llama.cpp Builds on This Mac

As of 2026-09-25 there are two separate llama.cpp installations:

| | This project's build | Homebrew build |
| --- | --- | --- |
| Location | `ThirdParty/llama.cpp/build/bin/` | `/opt/homebrew/bin/` (release 8140) |
| Used by | Every mise task in this project | Typing `llama-server` or `llama-cli` directly in any terminal |
| Model cache | `~/.cache/huggingface/hub` | `~/Library/Caches/llama.cpp` |
| Updated by | Section 7 (jj) plus `mise run build` | `brew upgrade llama.cpp` |

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

### 12.5 `git pull` or `mise run sync-source` says "You are not currently on a branch"

That's expected in a jj-managed copy. Use the jj steps in Section 7.4.

### 12.6 `jj log` shows a conflicted commit after a rebase

Upstream changed the same lines as one of your local commits. Run `jj resolve`, or edit the files by hand. If you'd rather go back, `jj undo` reverts the rebase.

## Change Log

| Version | Time | Author | Reason | Summary |
| --- | --- | --- | --- | --- |
| 1.0 | 2026-09-25T06:08:59-05:00 | Claude (on request) | Switched to the user's fork | `origin` is now the fork `chendingplano/llama.cpp` and ggml-org is `upstream`; rewrote Section 7 for two remotes, the `main` bookmark, `jj git fetch` from both, rebase onto `master@upstream`, and `jj git push -b main` |
| 1.0 | 2026-09-25T05:28:07-05:00 | Claude (on request) | Gemma 4 no longer used in llama.cpp | Removed all Gemma 4 tasks (`run-gemma-4-12b-coder-fable5`, `run-gemma-4-26b`, `run-gemma-4-31b`, `serve-gemma-4-26b`, `stop-gemma-4-26b`) and their downloaded models from the manual; server table now lists three embedding servers |
| 1.0 | 2026-09-25T05:16:23-05:00 | Claude (on request) | Syncing should use jj, not raw git | Rewrote Section 7 around jj (`jj git fetch`, `jj rebase -b workspace -o master@origin`, `jj undo`, the `workspace` bookmark); added everyday jj commands; updated the task table, two-builds table, and troubleshooting for jj; completed the `show-downloaded-models` row |
| 1.0 | 2026-09-25T04:52:46-05:00 | Claude (on request) | Manual lacked the embedding servers, status checks, cache details, and a working sync procedure | Added the background server table and bge-m3 usage (`serve-bge-m3`/`stop-bge-m3`); added "Checking Whether llama.cpp Is Running"; corrected the model cache location (`~/.cache/huggingface/hub`, not `~/Library/Caches/llama.cpp`) and listed cached models; replaced the sync instructions with a branch-plus-rebase procedure (the `sync-source` task doesn't work in detached HEAD with local commits); added the two-builds section; added metadata |
| 1.0 | 2026-06-30T04:57:57-05:00 | Not specified | Initial workspace setup | Created manual with build, run, sync, and Gemma shortcut instructions |

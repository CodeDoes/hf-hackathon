# AGENTS.md — Agent-Assisted CORE-ET Model Porting

This file is for **AI agents** (and the humans directing them) participating in
the AIFoundry + OpenHW CORE-ET Model Porting Hackathon. It describes the
repository structure, the agent workflow, documentation to read first, and
checklists for successful submissions.

If you are an agent reading this, your goal is to help port an open AI model so
it runs on ET-SoC1 silicon (a manycore RISC-V accelerator with a matrix engine)
and produces a leaderboard entry.

---

## 1. Read These First (In Order)

The repository is documented in layers. An agent should ingest them in this
sequence to build a correct mental model before writing code or editing configs.

| Order | Document | What it tells you |
|-------|----------|-------------------|
| 1 | [`README.md`](README.md) | Project overview, leaderboard, submission flow, links to all key docs |
| 2 | [`docs/SUBMISSION_GUIDE.md`](docs/SUBMISSION_GUIDE.md) | PR checklist, CI gates, leaderboard scoring rules |
| 3 | [`docs/ET_SOC1_QUICKSTART.md`](docs/ET_SOC1_QUICKSTART.md) | Fresh-workstation setup: toolchain, build, sys-emu, board staging |
| 4 | [`docs/HF_REFERENCES.md`](docs/HF_REFERENCES.md) | Pinned Hugging Face base models (repos, revisions, licenses) |
| 5 | [`docs/opinionated_porting_options/afonso.md`](docs/opinionated_porting_options/afonso.md) | Layer-by-layer porting workflow, correctness gates, PMC measurement |
| 6 | [`docs/opinionated_porting_options/martin.md`](docs/opinionated_porting_options/martin.md) | Hardware mental model, correctness footguns, performance playbook |
| 7 | [`docs/THIRD_PARTY.md`](docs/THIRD_PARTY.md) | Third-party license inventory |

External repos an agent should know about:

- [OpenHW CORE-ET RTL](https://github.com/openhwgroup/core-et)
- [AIFoundry ET platform](https://github.com/aifoundry-org/et-platform)
- [AIFoundry RISC-V GNU toolchain](https://github.com/aifoundry-org/riscv-gnu-toolchain)

---

## 2. Repository Structure

```
.
├── AGENTS.md                          ← You are here
├── README.md                          ← Project overview + leaderboard
├── LICENSE / NOTICE                   ← Apache-2.0
│
├── docs/
│   ├── SUBMISSION_GUIDE.md            ← PR checklist, CI, scoring
│   ├── ET_SOC1_QUICKSTART.md          ← Toolchain, build, sys-emu, board
│   ├── HF_REFERENCES.md               ← Pinned Hugging Face base models
│   ├── THIRD_PARTY.md                 ← License inventory
│   ├── BOARD_ACCESS.md                ← Discord + Tailscale board access
│   ├── opinionated_porting_options/
│   │   ├── afonso.md                  ← Layer-by-layer porting workflow
│   │   └── martin.md                  ← HW mental model + footguns + perf
│   └── vision_models/                 ← Vision model-specific docs
│
├── ported_models/                     ← All model ports live here
│   ├── dncnn/                         ← Standalone kernel port (DnCNN)
│   ├── yolo/                          ← Standalone kernel port (YOLO)
│   ├── llama_cpp_et/                  ← Framework port (llama.cpp GGML)
│   │   ├── src/llama.cpp-et/          ← Committed framework source
│   │   ├── benchmarks/                ← Per-model benchmark JSON
│   │   └── artifacts.json             ← Model/runtime artifact manifest
│   └── ggonnx/                        ← Framework port (GGONNX)
│
├── .github/
│   ├── ci/
│   │   ├── benchmark_config.json      ← Model registration for board CI
│   │   ├── scripts/                   ← CI runners, builders, preflight
│   │   └── launcher/                  ← Argbuf launcher source
│   └── workflows/                     ← GitHub Actions workflows
│
├── scripts/
│   ├── download_hf_refs.sh            ← Download pinned Hugging Face refs
│   └── run_sysemu_model_ports.sh      ← Local sys-emu runner
│
├── data/                              ← Leaderboard JSON (CI-managed, do not edit)
└── local-artifacts/                   ← Generated blobs (gitignored)
```

---

## 3. Agent Workflow

### 3.1 Understand the Two Porting Paths

From [`docs/opinionated_porting_options/martin.md`](docs/opinionated_porting_options/martin.md):

| Path | Description | Best for |
|------|-------------|----------|
| **A — GGML/llama.cpp** | Fill gaps in the existing ET backend so your model's ops run on hardware instead of falling back to CPU | Most LLM teams; get a whole model running quickly |
| **B — Standalone kernel** | Write a baremetal RISC-V kernel for one hot operation (matmul, attention, conv) | Single-op speed records; ops not covered by GGML |

**Recommendation:** Start with Path A to get a model running, profile to find
the dominant kernel, then use Path B for that kernel if needed.

### 3.2 Agent Task Decomposition

Break porting into these stages (from
[`docs/opinionated_porting_options/afonso.md`](docs/opinionated_porting_options/afonso.md)):

1. **Name the op/layer/block** and its boundary.
2. **Record input/output shapes**, dtype, scale, zero point, layout.
3. **Make a host reference** for the boundary (ONNX, numpy, saved taps).
4. **Write/modify the ET-SoC1 kernel** for that boundary.
5. **Compare silicon output against the host reference** (correctness gate).
6. **Measure wall time and PMCs** (performance gate).
7. **Journal the result** in the model's `docs/optimizations.md`.

Do one boundary at a time. Do not port a whole graph in one shot.

### 3.3 Agent-Ready Recipe Format

When submitting a port PR (per
[`docs/SUBMISSION_GUIDE.md`](docs/SUBMISSION_GUIDE.md)), include a reusable `.md`
recipe with:

- **Task breakdown** — what was done, in what order
- **Prompt files or instructions** — the actual prompts used if agent-driven
- **Repos/docs/RTL/model files** pointed at during the work
- **Commands that worked** — exact shell commands (parameterized, no personal paths)
- **Commands that failed** — dead ends that saved future time
- **Verification path** — how to confirm the port is correct
- **Environment assumptions** — toolchain version, board host, dependencies

Write for another agent or participant to reproduce exactly.

### 3.4 Example Agent Loop for a New Model on Existing Framework

1. Read this file and all docs listed in §1.
2. Read an existing port (e.g., `ported_models/llama_cpp_et/`) to understand the
   pattern.
3. Find a Hugging Face GGUF model matching the target (or convert to GGUF).
4. Add an `artifacts.json` entry pointing at the model file.
5. Add a benchmark JSON under `ported_models/llama_cpp_et/benchmarks/`.
6. Register the model in `.github/ci/benchmark_config.json`.
7. Run `bash .github/ci/scripts/ci_preflight.sh`.
8. Open a PR and let board CI validate.

### 3.5 Example Agent Loop for a Standalone Kernel

1. Read the DnCNN or YOLO port for the pattern.
2. Identify the operation boundary and shapes.
3. Generate a host reference (ONNX, numpy).
4. Write the kernel `.c` under `ported_models/<model>/src/`.
5. Build with the ET toolchain (`riscv64-unknown-elf-gcc`).
6. Run locally with sys-emu to check plumbing.
7. Stage to the board and run.
8. Compare output against host reference.
9. Add `artifacts.json` and register in `benchmark_config.json`.
10. Open a PR.

---

## 4. Key Rules for Agents

### 4.1 Do Not Edit These (CI Owns Them)

- `data/*.json` — leaderboard data, updated by CI after merge
- `README.md` leaderboard block (between `<!-- leaderboard:start -->` and
  `<!-- leaderboard:end -->`) — also CI-managed

### 4.2 Do Not Commit

- Model weight blobs, GGUF files, or any large binary artifacts
- Board dumps, log files, or temporary outputs
- Personal paths, SSH hosts, or environment secrets
- Machine-specific build artifacts

Use `local-artifacts/` (gitignored) for generated blobs during iteration.

### 4.3 Source of Truth Must Be Committed

Board CI builds from **committed** framework source. If you change a kernel,
the built binary that runs on the board must come from committed source, not a
private local override. For local iteration with `llama.cpp-et`, you can use
`GGML_ET_KERNELS_PATH` to load rebuilt kernel ELFs at runtime, but the PR must
commit the final kernel source.

### 4.4 Correctness Gates Performance

Do not optimize a kernel that has not passed its correctness gate. A faster
wrong kernel is not progress. From
[`docs/opinionated_porting_options/afonso.md`](docs/opinionated_porting_options/afonso.md):

> **Promotion rule:** A variant is promotable only when all of these are true:
> - It passes the model-specific correctness gate.
> - It improves the target performance metric or removes real complexity.
> - Its inputs, outputs, and artifact requirements are documented.
> - Its result is recorded in the model's `docs/optimizations.md`.

---

## 5. Hardware Mental Model (For Agents)

From [`docs/opinionated_porting_options/martin.md`](docs/opinionated_porting_options/martin.md):

```
Chip
 └── ~32 Shires           (2D mesh tiles on a NoC)
      └── ~32 Minions     (small in-order RISC-V cores)
           └── 2 Harts    (hardware threads per minion)
                ├── hart 0 → has the matrix/tensor engine
                └── hart 1 → no matrix engine
```

**Three memory tiers:**
| Tier | Size | Speed | Use |
|------|------|-------|-----|
| DRAM/main memory | Large | Slow | Weights, activations that don't fit locally |
| L2/shire-local scratchpad | Medium | Fast | Tile working set, shire-local cooperation |
| L1/minion-local scratchpad | Small | Fastest | Accumulators, matrix engine operands |

**Key insight:** The bottleneck is main memory bandwidth, not FLOPs. The whole
game is keeping data in on-chip scratchpad and feeding the matrix engine.

---

## 6. Common Failure Modes (Check These First)

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Model produces tokens but key ops on CPU | CPU fallback hiding missing ET coverage | Inspect `ET_PERF` lines and backend logs |
| Wrong numbers, looks plausible | Sync or cache bug | Audit barriers, cache ownership, data movement |
| Hangs | Missing sync, deadlock | Add barriers, check cooperative load patterns |
| No `Kernel wait seconds` in log | Kernel timed out or crashed | Check board log tail, reduce input size, add debug dumps |
| Sys-emu timeout | Emulator slow for large kernels | Does not mean kernel is wrong; use board for perf truth |
| Build rejects instruction | Unsupported RISC-V instruction on ET | Avoid float div, trig, sqrt, long-float casts |
| Wrong output only under multi-hart | L1D not coherent across minions | Audit cache ownership and barriers |
| CI can't find artifact | `artifacts.json` URL, hash, or path wrong | Double-check artifact manifest entries |

---

## 7. Agent Checklist for Opening a PR

Before opening a submission PR, verify:

- [ ] Source that runs on the board is committed under `ported_models/`.
- [ ] `artifacts.json` has stable URLs, revision hashes, file sizes, and local
      cache names.
- [ ] Model is registered once in `.github/ci/benchmark_config.json`.
- [ ] Benchmark runner writes a score JSON with status, notes, and primary
      metric.
- [ ] LLM rows include PPL settings (unless documented exception).
- [ ] Reusable `.md` recipe or agent notes is included in the PR.
- [ ] `bash .github/ci/scripts/ci_preflight.sh` passes.
- [ ] Hugging Face base model is pinned with repo, revision, and license.
- [ ] Any conversion/export/quantization step is documented.
- [ ] Generated artifacts, downloaded models, dumps, and board scratch files
      are **not** committed.
- [ ] Third-party code has clear license attribution.

---

## 8. Useful Commands for Agents

### Build and verify

```bash
# Preflight check (run before PR)
bash .github/ci/scripts/ci_preflight.sh

# Build launcher
.github/ci/scripts/build_launcher.sh

# Prepare benchmark inputs
.github/ci/scripts/prepare_benchmark_inputs.sh all

# Build ELF for a model
.github/ci/scripts/build_leaderboard_elf.sh <model_name>
```

### Local sys-emu (plumbing check before board)

```bash
# List available suites for a model
scripts/run_sysemu_model_ports.sh --launcher <launcher_path> --suite smoke --model <model> --list

# Run a smoke variant
scripts/run_sysemu_model_ports.sh \
  --launcher <launcher_path> \
  --suite smoke \
  --model <model> \
  --variant <variant>
```

### Download Hugging Face references

```bash
scripts/download_hf_refs.sh local-artifacts/hf_refs
```

### Build a kernel manually (parameterized)

```bash
GCC="$ET_INSTALL/bin/riscv64-unknown-elf-gcc"
"$GCC" \
  -O2 -nostdlib \
  -march=rv64imfc -mabi=lp64f -mcmodel=medany \
  -fno-zero-initialized-in-bss -ffunction-sections -fdata-sections \
  -I"<include_dirs>" \
  -Wl,--gc-sections -Wl,--no-warn-rwx-segments \
  -Wl,--defsym=region0_size=0x04000000 \
  -T "<linker_script>" \
  -o my_kernel.elf \
  my_kernel.c \
  <crt_files> \
  <layout_files>
```

---

## 9. Links to Pi Documentation (For Pi-Agent Users)

If you are running inside the `pi` coding agent, the following internal docs
are available:

| Topic | Path |
|-------|------|
| Pi main docs | `/home/kit/.local/share/pnpm/store/v11/links/@earendil-works/pi-coding-agent/.../README.md` |
| Extensions | `docs/extensions.md` |
| Themes | `docs/themes.md` |
| Skills | `docs/skills.md` |
| Prompt templates | `docs/prompt-templates.md` |
| TUI components | `docs/tui.md` |
| Keybindings | `docs/keybindings.md` |
| SDK integrations | `docs/sdk.md` |
| Custom providers | `docs/custom-provider.md` |
| Adding models | `docs/models.md` |
| Pi packages | `docs/packages.md` |

---

## 10. Getting Help

- **Discord `#Lab`:** <https://discord.gg/CbSA2umxf6> — ask questions, share
  progress, request board access
- **GitHub issues:** open against
  <https://github.com/aifoundry-org/hf-hackathon>
- **Board access:** see [`docs/BOARD_ACCESS.md`](docs/BOARD_ACCESS.md)
- **Hugging Face references:** see [`docs/HF_REFERENCES.md`](docs/HF_REFERENCES.md)

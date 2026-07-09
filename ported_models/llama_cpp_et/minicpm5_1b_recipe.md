# MiniCPM5-1B Porting Recipe

## Overview

This recipe documents adding **MiniCPM5-1B** (the first model in the MiniCPM5
series by OpenBMB) to the `llama.cpp-et` framework for the AIFoundry CORE-ET
Hackathon. MiniCPM5-1B is a dense 1B-parameter Transformer built for on-device
deployment, reaching 1B-class open-source SOTA on agentic tool use, code
generation, and reasoning.

## Model Reference

| Field | Value |
|-------|-------|
| **Hugging Face repo** | `openbmb/MiniCPM5-1B-GGUF` |
| **Revision** | `87007042419d30c1d8f38ef065424ee33870831e` |
| **GGUF file** | `MiniCPM5-1B-Q8_0.gguf` |
| **Format** | Q8_0 GGUF |
| **Size** | 1,153,529,216 bytes (~1.15 GB) |
| **SHA256** | `0dc7638539067268774c275a14a6ec9c7e01f7eeb2cff606c8590361fa527e4c` |
| **License** | Apache-2.0 |

## Architecture

MiniCPM5-1B uses standard **LlamaForCausalLM** architecture, fully supported by
llama.cpp and the existing ET backend:

| Parameter | Value |
|-----------|-------|
| Model type | `llama` |
| Hidden size | 1536 |
| Intermediate size | 4608 |
| Num layers | 24 |
| Num Q heads | 16 |
| Num KV heads | 2 (GQA) |
| Head dim | 128 |
| RoPE theta | 5,000,000 |
| Context length | 131,072 (128K) |
| Vocab size | 130,560 |
| Non-embedding params | ~680M |

## Steps Taken

1. **Identified base model**: MiniCPM5-1B was released by OpenBMB in May 2026.
   It uses standard Llama architecture, so no code changes to the llama.cpp-et
   backend are needed — the existing ET backend fully supports it.

2. **Added artifact to `artifacts.json`**:
   Added `minicpm5_1b_q8_gguf` to
   `ported_models/llama_cpp_et/artifacts.json` with the exact Hugging Face URL,
   revision, SHA256 checksum, and byte size.

3. **Created benchmark configuration**:
   Created `ported_models/llama_cpp_et/benchmarks/minicpm5_1b.json` with:
   - Standard Llama.cpp settings: `gpu_layers: 99`, `ubatch_size: 128`
   - Chat template: `openbmb/MiniCPM5-1B` uses `openbmb` chat template
     (based on `LlamaTokenizer` with `<|im_start|>`, `<|im_end|>` markers)
   - Prompt adapted for the model's chat format
   - Perplexity enabled with WikiText-2 raw test corpus

4. **Registered in benchmark config**:
   Added the model mapping to `.github/ci/benchmark_config.json` under the
   `"models"` block.

5. **Validation**:
   Run `.github/ci/scripts/ci_preflight.sh` to ensure all JSON schemas are
   valid and CI workflows parse correctly before opening a PR.

## Chat Template

MiniCPM5-1B uses the following chat template format (OpenBMB style):

```
<|im_start|>system
{system_message}<|im_end|>
<|im_start|>user
{user_message}<|im_end|>
<|im_start|>assistant
{assistant_message}<|im_end|>
```

The model also supports a built-in ` thinking` mode for chain-of-thought
reasoning, toggled via `enable_thinking` in the template. The benchmark prompt
uses the standard non-thinking format for consistent comparison.

## Instructions for Reproduction

No custom model packing, quantization, or code changes were required, as:

- The model was already available in Q8_0 GGUF on Hugging Face.
- The architecture is standard Llama, which the ET backend fully offloads.
- Board CI will automatically download the GGUF file from Hugging Face based
  on the SHA256 and URL in `artifacts.json`.

To build and run locally:

```bash
# Set up environment (see ET_SOC1_QUICKSTART.md)
export ET_PLATFORM="$ET_INSTALL"
export PATH="$ET_INSTALL/bin:$PATH"

# Build the framework
cd "$MODEL_PORT_REPO"
.github/ci/scripts/build_launcher.sh
.github/ci/scripts/prepare_benchmark_inputs.sh all

# Build the llama.cpp server (from committed submodule)
LLAMA_CPP_WORKDIR="$BENCHMARK_ARTIFACT_ROOT/frameworks/llama.cpp-et/build-et"
cmake -S ported_models/llama_cpp_et/src/llama.cpp-et \
      -B "$LLAMA_CPP_WORKDIR" \
      -DGGML_ET=ON -DCMAKE_BUILD_TYPE=Release
cmake --build "$LLAMA_CPP_WORKDIR" --config Release -j$(nproc)

# Run server (after downloading the GGUF via artifacts.json)
"$LLAMA_CPP_WORKDIR/bin/llama-server" \
  --model local-artifacts/models/minicpm5_1b/MiniCPM5-1B-Q8_0.gguf \
  --host 127.0.0.1 --port 18095 \
  -ngl 99 -c 2048 -b 256 --ubatch-size 128 \
  --device ET
```

## Key Notes

- MiniCPM5-1B has **128K context** — the benchmark uses 2048 for consistency
  with other leaderboard entries, but longer contexts are possible.
- The model's eos_token_ids are `[1, 130073]`. The benchmark sets
  `ignore_eos: true` for clean token throughput measurement.
- No `rope_scaling` is applied — the model uses a high RoPE theta (5,000,000)
  for long-context support natively.
- At ~1.15 GB in Q8_0, this model fits comfortably within ET-SoC1 board memory.

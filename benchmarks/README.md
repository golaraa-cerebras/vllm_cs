# Benchmarks

This directory used to contain vLLM's benchmark scripts and utilities for performance testing and evaluation.

## Contents

- **Serving benchmarks**: Scripts for testing online inference performance (latency, throughput)
- **Throughput benchmarks**: Scripts for testing offline batch inference performance
- **Specialized benchmarks**: Tools for testing specific features like structured output, prefix caching, long document QA, request prioritization, and multi-modal inference
- **Dataset utilities**: Framework for loading and sampling from various benchmark datasets (ShareGPT, HuggingFace datasets, synthetic data, etc.)

## Usage

For detailed usage instructions, examples, and dataset information, see the [Benchmark CLI documentation](https://docs.vllm.ai/en/latest/contributing/benchmarks.html#benchmark-cli).

For full CLI reference see:

- <https://docs.vllm.ai/en/latest/cli/bench/latency.html>
- <https://docs.vllm.ai/en/latest/cli/bench/serve.html>
- <https://docs.vllm.ai/en/latest/cli/bench/throughput.html>

## Cerebras Inference API benchmark

`benchmark_serving_structured_output.py` includes two backends that target the
hosted Cerebras Inference API (`https://api.cerebras.ai`):

- `cerebras-chat` — POSTs streaming chat completions to `/v1/chat/completions`.
  Supports reasoning models (`gpt-oss-120b`, `zai-glm-4.7`).
- `cerebras-text` — POSTs streaming text completions to `/v1/completions` with
  `return_raw_tokens: true`.

### Setup

```bash
python3 -m venv .venv
.venv/bin/pip install aiohttp transformers datasets pandas numpy tiktoken \
    tqdm huggingface_hub python-dotenv
```

Put your API key in a `.env` file at the repo root:

```
CEREBRAS_API_KEY=sk-...
```

### Smoke test

```bash
.venv/bin/python benchmarks/benchmark_serving_structured_output.py \
    --backend cerebras-chat --model llama3.1-8b --dataset random \
    --num-prompts 5 --max-concurrency 1 \
    --random-input-len 200 --output-len 100 --no-structured-output
```

The defaults point at `https://api.cerebras.ai` and `/v1/chat/completions`, so
no `--base-url` is needed.

### Reasoning models

Pass model-specific parameters through `--extra-body` (JSON merged into every
request payload):

```bash
.venv/bin/python benchmarks/benchmark_serving_structured_output.py \
    --backend cerebras-chat --model gpt-oss-120b --dataset random \
    --num-prompts 5 --max-concurrency 1 \
    --random-input-len 200 --output-len 200 --no-structured-output \
    --extra-body '{"reasoning_effort": "low"}'
```

When reasoning is observed in the stream, two client-side TTFTs are reported
side-by-side under `Time to First Token (client)` — `Answer` (first content
chunk) and `Reasoning` (first reasoning chunk). If `max_completion_tokens` is
exhausted by reasoning and the model never emits a content token, the report
flags this with a `...of which emitted no answer:` line and filters the
affected requests out of the Answer-TTFT distribution.

### Disabling streaming

```bash
.venv/bin/python benchmarks/benchmark_serving_structured_output.py \
    --backend cerebras-chat --model llama3.1-8b --dataset random \
    --num-prompts 5 --max-concurrency 1 \
    --random-input-len 200 --output-len 100 --no-structured-output --no-stream
```

In `--no-stream` mode, per-chunk metrics (client TTFT, ITL, client TPOT) are
unavailable and auto-hidden. Cerebras server-side metrics, client E2EL, and
network RTT are still reported.

### Reported metrics

| Section | Description |
| --- | --- |
| `Time to First Token (client)` | Wall-clock from POST to first content chunk (`Answer`) and first reasoning chunk (`Reasoning`). Empty columns auto-hide. |
| `Time to First Token (Cerebras)` | Server-reported `queue_time + prompt_time` — time before *any* token is generated. |
| `Time per Output Token` | Client: `(latency − start_of_generation) / (output_tokens − 1)`. Cerebras: `completion_time / completion_tokens`. |
| `Inter-chunk Latency` | Client wall-clock interval between consecutive content-bearing SSE chunks. |
| `End-to-end Latency` | Client wall-clock total vs Cerebras-reported `total_time`. |
| `Network RTT` | `client_latency − cerebras_total_time` per request — overhead absorbed by the client beyond Cerebras's reported processing time (network + TLS + aiohttp buffering). |

Pick metrics with `--percentile-metrics`. The default is
`ttft,reasoning_ttft,tpot,itl,e2el,network_rtt`. Each name expands to its
client-side and Cerebras counterpart where one exists; metrics with no data
(e.g. `reasoning_ttft` on a non-reasoning model) are silently dropped from
the report.

# Energy attribution for SGLang

Research work on this fork (branch `resource-tracking`): attribute measured GPU
energy (NVML), and later CPU/DRAM energy (RAPL), to individual requests that the
SGLang scheduler batched together. The approach follows Roy 2026 ("Seam") and
our draft paper "Runtime Energy Attribution of Inference Requests in LLM Serving
Systems".

| Phase | Status |
|---|---|
| 1. Step trace (scheduler steps + per-request demand) | Done without overlap scheduler; overlap pending |
| 2. NVML power sampler | Not started |
| 3. Offline attribution | Not started |
| 4. Validation (overhead, isolated vs co-scheduled) | Not started |

Details and the optional improvements are in [PLAN.md](PLAN.md).

## Layout

| Path | Contents |
|---|---|
| [`PLAN.md`](PLAN.md) | Phases, design notes, optional improvements |
| [`docs/step_trace.md`](docs/step_trace.md) | Trace format reference: env vars, record fields, clock, validation |
| [`tools/step_trace_client.py`](tools/step_trace_client.py) | Sends a concurrent workload and checks a trace against it |
| `runs/<date>_<name>/` | One folder per experiment (git-ignored): `trace.jsonl`, `manifest.json`, `server_cmd.txt` |
| `papers/` | Reference PDFs (git-ignored) |

The tracer itself lives in the SGLang package:
`python/sglang/srt/managers/scheduler_components/step_tracer.py`, wired into
`python/sglang/srt/managers/scheduler.py`.

## Running a traced experiment

On the tower, inside the `sglang-dev` container (the repo is mounted at
`/workspace/sglang`):

```bash
RUN=/workspace/sglang/energy_attribution/runs/$(date +%F)_my-run
mkdir -p $RUN

SGLANG_ENABLE_STEP_TRACE=1 SGLANG_STEP_TRACE_DIR=$RUN \
SGLANG_UNIFIED_RADIX_TREE_CORE_BACKEND=python \
python -m sglang.launch_server --model-path Qwen/Qwen2.5-1.5B-Instruct \
    --host 0.0.0.0 --port 30000 --disable-overlap-schedule

python energy_attribution/tools/step_trace_client.py send --out $RUN/manifest.json
python energy_attribution/tools/step_trace_client.py check \
    --manifest $RUN/manifest.json --trace $RUN/step_trace.*.jsonl
```

Write the exact server and client commands into `$RUN/server_cmd.txt`, so every
trace can be traced back to the configuration that produced it. See
[`runs/2026-10-16_mixed-chunk/server_cmd.txt`](runs/2026-10-16_mixed-chunk/server_cmd.txt)
for an example (local only).

## Tower notes

- Time base: all trace timestamps use `CLOCK_MONOTONIC_RAW`. Under WSL2,
  `time.monotonic_ns()` is slewed 2-4% against the GPU timer; any power sampler
  must use the raw clock too.
- `SGLANG_UNIFIED_RADIX_TREE_CORE_BACKEND=python` is needed because the
  container was installed without the Rust extensions.
- Stop a server with Ctrl-C in its tmux session. Killing the tmux session leaves
  the server running inside the container; then use
  `docker exec sglang-dev pkill -f "launch_server|sglang::"`.

## Papers

- Roy, 2026. "Scheduler-step GPU energy attribution for continuously batched LLM
  serving" (Seam).
- Draft: "Runtime Energy Attribution of Inference Requests in LLM Serving
  Systems".

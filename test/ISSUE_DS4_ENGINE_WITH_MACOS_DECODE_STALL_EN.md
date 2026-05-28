# [macOS][Apple Silicon] Severe Metal Graph Execute Spikes (~35–37s) Near Context Boundary (`pos=16384`) in `ds4-agent` Self-Run Path

> This report was produced with AI assistant participation (test orchestration, log parsing, and evidence summarization), then manually reviewed by the operator.

## 1) Problem Description

On Apple Silicon (M3 Ultra), `ds4-agent` autonomous (`--non-interactive`) self-run path reproducibly shows severe single-step `execute` spikes (~35–37s) near the context boundary.

Under the same hardware/model/runtime parameters, direct `ds4-bench` remains stable (~36–42ms) and shows no >5s spikes.

## 2) Environment

- Hardware: Apple M3 Ultra, 256 GiB Unified Memory
- OS: macOS 15.x
- Model: `ds4flash.gguf`
- Runtime params:
  - `--power 70 --step-mul 2 --gen-tokens 128 --ctx-max 16384`
  - `--prompt-file tests/long_context_story_prompt.txt`
- Source verification:
```bash
rg -n "metal graph token pos=|execute=.* ms" ds4.c
nl -ba ds4.c | sed -n '13486,13508p'
```

## 3) Concrete Evidence

### 3.1 Metric source confirmation

The log line:

`ds4: metal graph token pos=%u encode=... execute=...`

is emitted by the inference core in `ds4.c` (`ds4.c:13499`), where `execute` is computed as:

`(t_done - t_encoded) * 1000.0` (`ds4.c:13502`).

So the observed 35–37s spikes are from graph execution timing, not an agent-side tool-wait metric field.

### 3.2 Path A: `ds4-agent` self-run (reproduces stalls)

Artifacts:

- `test/root_cause_no_reboot/20260528-104212`

Representative lines:

```log
ds4: metal graph token pos=16384 ... execute=30443.735 ms
ds4: metal graph token pos=16386 ... execute=34896.988 ms
ds4: metal graph token pos=16388 ... execute=35079.142 ms
```

Summary:

- `agent_warmoff`: 3/3 fail, peak `36760.476ms`
- `agent_warmon`: 3/3 fail, peak `36679.849ms`

### 3.3 Path B: direct `ds4-bench` (does not reproduce)

Artifacts:

- `test/no_agent_trigger_check/20260528-114156`

Representative lines:

```log
ds4: metal graph token pos=16384 ... execute=36.784 ms
ds4: metal graph token pos=16386 ... execute=36.153 ms
ds4: metal graph token pos=16388 ... execute=37.065 ms
```

Summary:

- `bench_warmoff`: 3/3 pass, peak `41.133ms`, `gt_5000ms=0`
- `bench_warmon`: 3/3 pass, peak `42.35ms`, `gt_5000ms=0`

## 4) Technical Analysis (Hypothesis)

This issue shows strong path dependency and boundary dependency.

1. Boundary correlation: spikes consistently appear near/after `pos >= 16384`, suggesting context-boundary handling is involved.
2. Path dependency: direct `ds4-bench` is stable while `ds4-agent` self-run is unstable under otherwise matched core parameters.
3. Warm experts not sufficient: `DS4_METAL_WARM_EXPERTS=0/1` does not eliminate stalls in agent path.

Current hypothesis (not yet proven): an interaction between agent execution path and long-context boundary handling triggers an expensive fallback/recompute/memory-mapping behavior in the graph execution stage.

### Limitation

Although core runtime parameters are matched, agent path may evolve transcript/history differently from direct bench path. So token-evolution equivalence is not guaranteed yet; this should be validated in follow-up instrumentation.

## 5) Suggested Action Items

1. Add phase-level timing in agent path:
   - decode/graph execute
   - tool launch/wait
   - context rebuild/replay
2. Compare KV/context state transitions between agent self-run and direct bench at `pos` near `ctx-max`.
3. Add boundary-focused regression test:
   - case: `power=70`, `ctx-max=16384`, long prompt
   - assert no pathological execute spike (e.g. no >5s token execute).

## 6) Attached Artifacts

- `test/root_cause_no_reboot/20260528-104212/final_summary.md`
- `test/root_cause_no_reboot/20260528-104212/summary.csv`
- `test/root_cause_no_reboot/20260528-104212/events.log`
- `test/no_agent_trigger_check/20260528-114156/summary.csv`


# Fix Validation: `ds4-agent` Self-Run Decode Stall

## Patch

File:

- `ds4_agent.c`

Change:

- In `agent_execute_tool_call()` for `bash`, when running with `--non-interactive`, force `refresh_sec >= timeout_sec`.
- This prevents periodic intermediate `bash` observations from returning early and triggering overlapping extra model turns while the same long-running command is still executing.

## Why this patch

The reproduced stall pattern required the `agent_self` path.  
Direct `ds4-bench` under the same benchmark shape was stable.

In `agent_self`, the bash tool previously produced intermediate "running" observations that could trigger additional model rounds while the spawned `ds4-bench` was still running. This creates an overlap pattern that does not exist in direct bench mode.

## Validation Runs

## 1) Focused agent_self validation (post-fix)

Directory:

- `test/fix_validation/20260528-133235`

Results:

- `agent_warmoff`: `count=512`, `max=42.079ms`, `gt1000=0`, `gt5000=0`
- `agent_warmon`: `count=512`, `max=42.588ms`, `gt1000=0`, `gt5000=0`

## 2) Full no-reboot matrix regression (post-fix)

Directory:

- `test/root_cause_no_reboot/20260528-133912`

Summary:

| case | n | fail_count | gt5s_runs | max_execute_ms_peak |
|---|---:|---:|---:|---:|
| bench_warmoff | 2 | 0 | 0 | 42.540 |
| bench_warmon  | 2 | 0 | 0 | 41.859 |
| agent_warmoff | 2 | 0 | 0 | 41.643 |
| agent_warmon  | 2 | 0 | 0 | 58.122 |

No `>1000ms` or `>5000ms` spikes were observed in the post-fix matrix.

## 3) Pre-fix reference (same matrix shape previously reproduced severe stalls)

Directory:

- `test/root_cause_no_reboot/20260528-104212`

Highlights:

- `agent_warmoff`: peak ~`36760ms`
- `agent_warmon`: peak ~`36679ms`
- repeated `gt_5000ms > 0`

## Conclusion

This patch eliminated the previously reproducible 35–37s execute spikes in `agent_self` validation and matrix regression runs on this machine.


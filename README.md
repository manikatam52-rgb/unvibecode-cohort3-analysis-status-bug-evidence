# Evidence package — analysis-status.json never finalized after clean CLI exit

UnvibeCode Engineering Challenge 2026 — Cohort 3 · Track 1
Companion evidence for the GitHub issue:
**[Bug] analysis-status.json remains RUNNING after review process terminates**

## What happened

Two independent `unvibecode review` runs against a shallow clone of pallets/flask were
executed on Windows 11 (build 10.0.26200, x64), Python 3.12.8, unvibecode 0.3.15.
Both runs completed: the Connected Code Map lane finished (READY / 100%), the hosted
Business Workflow & Risk Review lane failed on exhausted API credits (FAILED / 20%),
and the CLI process exited normally.

After the process exited, `technical-logs/analysis-status.json` in **both** runs still
contained the startup snapshot — `overall_status: "RUNNING"` and both lanes
`"WAITING"` with null timestamps and empty artifacts — while the per-lane progress
files in the same folder contain terminal states written *after* the status file's
`updated_at`.

## Reproduction

```powershell
python -m pip install --upgrade unvibecode
git clone --depth 1 https://github.com/pallets/flask.git flask-test
python -m unvibecode review --repository "<abs path>\flask-test" --no-open-browser
```

Let the process run to completion and exit. Then open
`unvibecode_results\flask-test-unvibecode-analysis-<timestamp>\technical-logs\analysis-status.json`
and the two `progress\*.json` files and compare `overall_status` / lane statuses with
the `phase` / `percent` fields and the `updated_at` timestamps.

Note: the hosted lane failed here because the hosted gateway had no API credits at
test time. The lane failure is not the defect being reported — the defect is that the
status file is never finalized on the clean-exit path. The success-path behavior was
not tested and no claim is made about it.

## Timeline (both runs)

| Run | status `updated_at` | CCM progress `updated_at` (READY/100) | BWR progress `updated_at` (FAILED/20) |
|-----|--------------------|---------------------------------------|----------------------------------------|
| run_011049 | 2026-11-04T19:40:49.897872Z | 19:40:52.899592Z (+3 s) | 19:41:15.745224Z (+26 s) |
| run_011242 | 2026-11-04T19:42:42.441895Z | 19:42:44.382417Z (+2 s)  | 19:42:57.637808Z (+15 s) |

In both runs the last write to `analysis-status.json` is the startup write; both
terminal progress writes come later, and the process exited after them.

## Contents

```
run_011049/technical-logs/
  analysis-status.json                      # final on-disk state after process exit
  progress/connected_code_map.json          # terminal READY / 100
  progress/business_workflow_review.json    # terminal FAILED / 20
  connected_code_map.log.excerpt.txt        # last lines: FAST LANE COMPLETE
  business_workflow_review.log.excerpt.txt  # last lines: CustomerRunError 429 traceback
run_011242/
  (same structure, second independent run)
```

## Sanitization

- Absolute Windows paths in the JSON/log excerpts were replaced with `<USER>\...`
  (they contained the tester's Windows username).
- No API keys, tokens or credentials were present in these files: the hosted flow
  uses a public-preview token written by the tool itself into a temp `.env`, and that
  value does not appear in any file in this package.
- Engine temp paths (`...\Temp\uvc-*`, `unvibecode-engine-*`) are local temp folders,
  not secrets; they were shortened for readability.
- Original evidence files under `unvibecode_results\` were not modified; this package
  is read-only copies plus excerpts.

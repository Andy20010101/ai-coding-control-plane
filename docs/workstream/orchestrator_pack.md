# Orchestrator Pack Notes

This document summarizes how execution artifacts are packaged for controller review.

## FAIL Handling

If a step result is `FAIL` or `REWORK`, it must go back through a rework ticket before another attempt is accepted.

## Expected Contents

- StepPack / ReworkPack identifier (`run_id`, `step_id`)
- `step_result.json`
- command output evidence under `.artifacts/...`
- summary note for controller QA

# Orchestrator Pack Notes

This document summarizes how execution artifacts are packaged for controller review.

## FAIL Handling

If a step result is `FAIL` or `REWORK`, it must go back through a rework ticket before another attempt is accepted.

Controller and worker notes for FAIL handling:

- Route FAIL back through a rework ticket; do not accept ad-hoc fixes outside the ticket.
- Quote failed acceptance criteria verbatim in the rework payload and review notes.
- Include a re-run command so the next executor attempt is reproducible.
- Link `.artifacts/...` evidence paths for failed gates and for the repaired attempt.

## Expected Contents

- StepPack / ReworkPack identifier (`run_id`, `step_id`)
- `step_result.json`
- command output evidence under `.artifacts/...`
- summary note for controller QA

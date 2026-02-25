# Kit Usage (installed into a target repo)

> Updated: 2026-02-25

This file is meant to live inside a target project after installing the kit.

## Quick start
1) Run doctor:
```powershell
pwsh -ExecutionPolicy Bypass -File .\scripts\control_plane\doctor.ps1 -RepoRoot .
```

2) Configure:
- `templates/control_plane/answers.yml`

3) Generate contract & protocol updates:
- follow `templates/control_plane/prompt_script.md`

4) Run with worktrees:
```powershell
git worktree add -b cp-exec ..\<repo>__exec
git worktree add -b cp-ctrl ..\<repo>__ctrl
```

## Standard commands
- Verify contract (minimal execution harness):
```powershell
pwsh -ExecutionPolicy Bypass -File .\scripts\control_plane\verify_contract.ps1
```

## Evidence convention
All evidence must land under:
`.artifacts/agent_runs/<RUN_ID>/<STEP_ID>/<ATTEMPT_ID>/...`

PASS requires evidence paths.

# Usage Guide — AI Coding Control Plane (Kit + Bootstrap Skill)

> Updated: 2026-02-25

This repo provides:
- **Kit**: a folder tree you can copy/install into any project to run “Contract → Execute → Evidence → Verify → Rework → Finalize”.
- **Bootstrap Skill**: an installer that overlays the kit into a target repo and writes an auditable bootstrap report under `.artifacts/`.

This guide focuses on **using** the kit in a real project (Windows + PowerShell + Git + Python), and the **two-Codex workflow** (Controller/Judge vs Executor).

---

## 1. Concepts in one minute

- **Controller / Judge (Codex B)**: writes contracts/StepPacks, runs verification, decides PASS/REWORK, issues ReworkPacks / Change Orders, merges branches.
- **Executor (Codex A)**: changes code **only inside allowed_paths**, runs allowed commands, produces evidence under `.artifacts/`, writes `step_result.json`.

**Hard rule:** **PASS requires evidence paths** under `.artifacts/...`. No “oral pass”.

---

## 2. Install the kit into a target project

### Option A — use the bootstrap installer (recommended)

From a target project folder:

```powershell
python <PATH_TO_THIS_REPO>\skills\control-plane-bootstrap\scripts\install.py --target .
pwsh -ExecutionPolicy Bypass -File .\scripts\control_plane\doctor.ps1 -RepoRoot .
```

Expected:
- A bootstrap report at: `.artifacts/agent_runs/<RUN_ID>/bootstrap/bootstrap_report.json`
- `doctor.ps1` prints **PASS**

### Option B — copy kit manually
Copy the kit tree into the target repo root so these paths exist:

- `docs/control_plane/*`
- `templates/control_plane/*`
- `protocol/control_plane/*`
- `runtime/control_plane/*`
- `harness/control_plane/*`
- `policies/control_plane/*`
- `governor/control_plane/*`
- `scripts/control_plane/*`
- `contracts/control_plane/*`
- `examples/control_plane/*`

Then run:

```powershell
pwsh -ExecutionPolicy Bypass -File .\scripts\control_plane\doctor.ps1 -RepoRoot .
```

---

## 3. Configure your first “golden path” harness case

Edit:

- `templates/control_plane/answers.yml`

Minimum fields to set:
- `harness_case.one_liner`
- `harness_case.inputs` / `outputs`
- `scope.allowed_paths` / `forbidden_paths`
- `commands.allowed`
- `success_criteria` (at least one gate command)

---

## 4. Generate MasterContract + StepPacks (Controller/Judge)

Open:

- `templates/control_plane/prompt_script.md`

Follow prompts 1→4 to generate:
- `contracts/control_plane/v1/master_contract.md`
- `contracts/control_plane/v1/conformance_report.md`
- (optional) `contracts/step_packs/S1.json`, `S2.json`, …

**Freeze rule:** once you “freeze” the contract, execution must not change it. Any changes go through **Change Order**.

---

## 5. Run with two Codex instances (worktree workflow)

### 5.1 Create worktrees (in the target project repo)

```powershell
git worktree add -b cp-exec ..\<repo_name>__exec
git worktree add -b cp-ctrl ..\<repo_name>__ctrl
```

Open two VS Code windows:
- `...__exec` → Codex A (Executor)
- `...__ctrl` → Codex B (Controller/Judge)

### 5.2 Role boundaries (summary)

**Codex A (Executor)**
- Allowed: edit only `allowed_paths`, run only allowed `commands`, write evidence to `.artifacts/`.
- Must: write `step_result.json` with `evidence_map`.
- Forbidden: edit contracts/policies/gates/scope (unless Change Order approved).

**Codex B (Controller/Judge)**
- Allowed: edit contracts/policies/protocol/harness/docs, run verify/gates, review diffs, merge.
- Must: decide PASS/REWORK based on evidence.
- Avoid: editing business code directly.

---

## 6. The standard Step loop (Dispatch → Execute → Judge)

### 6.1 Controller dispatches a StepPack
Controller creates/updates `contracts/step_packs/S<k>.json` and commits on `cp-ctrl`.

A StepPack must include:
- objective
- scope (allowed/forbidden/dependency_policy)
- commands
- acceptance (AC + gates + evidence_required)
- runtime_policy (timeout/no-output/retry/autonomy_level)

Controller sends Executor:
- StepPack path
- run_id + step_id
- required evidence list

### 6.2 Executor executes and produces evidence
Executor creates an Attempt directory:

`.artifacts/agent_runs/<RUN_ID>/<STEP_ID>/<ATTEMPT_ID>/`
- `logs/`
- `outputs/`
- `step_result.json` (required)

Executor commits code changes **excluding** `.artifacts/` and sends Controller:
- commit hash
- `step_result.json` path
- evidence paths (outputs/logs)

### 6.3 Judge verifies and decides PASS/REWORK
In `cp-ctrl` worktree, Controller/Judge:

1) Merge without committing (recommended):
```powershell
git checkout cp-ctrl
git merge --no-commit --no-ff cp-exec
```

2) Run gates:
```powershell
pwsh -ExecutionPolicy Bypass -File .\scripts\control_plane\doctor.ps1 -RepoRoot .
pwsh -ExecutionPolicy Bypass -File .\scripts\control_plane\verify_contract.ps1
```

3) Verify evidence:
- `step_result.json` exists
- every `evidence_required` exists
- `evidence_map` points to real `.artifacts/...` paths

Decision:
- PASS → commit the merge
- REWORK → abort merge + issue a ReworkPack

PASS commit example:
```powershell
git commit -m "merge(cp-exec): accept S<k> (PASS)"
```

REWORK flow:
```powershell
git merge --abort
# create contracts/rework_packs/S<k>_attempt<n>.json
git commit -m "rework: S<k> attempt<n>"
```

---

## 7. Change Order (required for scope/dep/gate changes)

Trigger conditions (examples):
- expanding `allowed_paths` or removing `forbidden_paths`
- adding dependencies / touching lock files
- modifying gates
- changing output contract (schema)

Minimum artifacts for a Change Order:
- `change_order.md` (reason/impact/rollback/new_gates)
- `human_decision.json` (APPROVE or DENY)

---

## 8. Close a milestone (Final Report → merge to main → tag)

When all Steps PASS:
- write `final_report.md` (how to verify, risks, evidence index)
- include review JSONs (if any)

Merge `cp-ctrl` into `main`:
```powershell
git checkout main
git merge cp-ctrl
```

Recommended:
- update `CHANGELOG.md`
- tag a release: `git tag v0.x.y`

---

## 9. What to commit vs what not to commit

Commit:
- contracts/policies/protocol/harness/governor/docs/templates (source of truth)
- step packs / rework packs / final report

Do NOT commit (default):
- `.artifacts/**` (local evidence store)
- `.tmp_target/**` (smoke test target)

---

## 10. Troubleshooting

### “No commits yet / orphan branch / cannot merge”
Create a baseline commit on `main` first, then rebase worktrees onto it.

### “Rework requested after you stopped”
After milestone PASS, mark the run as **CLOSED**. Any new work requires a **new StepPack** (new run_id) or **Change Order**.

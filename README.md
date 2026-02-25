# Workflow Logger

## P0 / Start Here

Use this section as the S2 "Start Here" entry when onboarding or dispatching work.

### Tier Routing Quick Decision (tier2 / tier3)

- Choose `tier2` for short, low-risk tasks with clear route and acceptance.
- Choose `tier3` for high-risk or multi-step work that needs contract-first execution.

### Minimal Success Path

1. Confirm scope and acceptance criteria.
2. Dispatch a bounded StepPack.
3. Require evidence-backed StepResult before accepting PASS.

### S2 Execution Checklist

1. Confirm the active StepPack scope and only edit `allowed_paths`.
2. Run the required verification commands and store output evidence under `.artifacts/`.
3. If QA returns `FAIL` or `REWORK`, route the step through a rework ticket before another acceptance attempt.
4. Submit a StepResult with command records, `reason_code`, `changed_files`, and AC-to-evidence mapping.

Reusable append-only logging module for AI workflow execution with strict `R/S`
iteration protocol:

- First attempt must be `R0/S0`
- Retry must be same `R` and `S+1`
- New revision must be `R+1/S0`
- Task can be closed only once with a final record
- Every event includes `prev_event_hash` + `event_hash` for tamper detection
- Supports project `export/import` with file-level SHA256 verification

## Layout

```text
workflow/<project_id>/
  manifest/
    project_manifest.json
    schema_version.json
  events.jsonl
  tasks/
    <task_id>/
      events.jsonl
      task_log.md
      artifacts/
```

## Python API

```python
from workflow_logger import WorkflowLogger

logger = WorkflowLogger(root_dir="workflow", project_id="ford-ai-flow")

logger.log_attempt(
    task_id="T-013",
    r=0,
    s=0,
    acceptance_snapshot=[
        "AC-1: python -m unittest returns 0",
        "AC-2: no out-of-scope files changed",
    ],
    code_change_summary="Fix parser edge case and add test coverage.",
    code_artifact_ref="artifacts/T-013/R0/S0/patch.diff",
    code_content="def parse_label(raw): ...",
    executed_files=["src/parser.py", "tests/test_parser.py"],
    executed_commands=["python -m unittest tests.test_parser"],
    test_results="0 failed, 8 passed",
    result="PASS",
    reason_code="PASS_ALL_CRITERIA",
    reason_detail="All acceptance criteria are satisfied.",
    next_action="Close task.",
)

logger.close_task(
    task_id="T-013",
    final_status="accepted",
    summary="Accepted with full evidence.",
)

logger.verify_task_chain("T-013")
logger.export_project(archive_path="./ford-ai-flow.zip")
WorkflowLogger.import_project(
    root_dir="./workflow",
    archive_path="./ford-ai-flow.zip",
    project_id="ford-ai-flow-migrated",
)
```

## CLI

```bash
python -m workflow_logger.cli --project-id ford-ai-flow init

python -m workflow_logger.cli --project-id ford-ai-flow attempt \
  --task-id T-013 --r 0 --s 0 \
  --acceptance "AC-1: tests pass" \
  --code-change-summary "Implement API + tests" \
  --code-content "def handler(): return {'ok': True}" \
  --executed-file src/api.py \
  --executed-file tests/test_api.py \
  --executed-command "python -m unittest tests.test_api" \
  --test-results "0 failed, 12 passed" \
  --result PASS \
  --reason-code PASS_ALL_CRITERIA \
  --reason-detail "Acceptance reached." \
  --next-action "Close task"

python -m workflow_logger.cli --project-id ford-ai-flow close \
  --task-id T-013 \
  --final-status accepted \
  --summary "Task completed and archived."

python -m workflow_logger.cli --project-id ford-ai-flow verify --task-id T-013

python -m workflow_logger.cli --project-id ford-ai-flow export --archive ./ford-ai-flow.zip

python -m workflow_logger.cli --root-dir ./workflow import \
  --archive ./ford-ai-flow.zip \
  --target-project-id ford-ai-flow-migrated
```

## Tests

```bash
python -m unittest tests.test_workflow_logger
```

## Governance Files (P0)

The repository now includes versioned control rules and SOPs:

- `AGENTS.md` (global non-negotiables and evidence policy)
- `docs/philosophy.md` (what counts as real progress)
- `docs/triage.md` (queue policy and transition rules)
- `instructions/controller.md`
- `instructions/worker.md`
- `instructions/qa.md`

These files are intended to be the stable, auditable "system prompt in repo".

## Controller/Worker Protocol

`orchestrator/protocol/` defines strict JSON payload schemas for:

- Controller -> Worker step dispatch (`StepPack`)
- Controller -> Worker rework dispatch (`ReworkPack`)
- Worker -> Controller normalized result (`StepResult`)

See `orchestrator/protocol/README.md` and run:

```bash
python -m unittest tests.test_protocol
```

## Shim Adapter

`orchestrator/worker/shim_adapter.py` provides an executable bridge:

1. Validate `StepPack` / `ReworkPack`
2. Build and run `gxd-subagent-shim create|resume ... --run-id --task-id --step-id`
3. Parse strict `StepResult` from shim output
4. Optionally append one attempt into `workflow_logger` (`R/S` pointer auto-managed)

Run adapter tests:

```bash
python -m unittest tests.test_shim_adapter
```

## Intake + Contract Engine

`orchestrator/intake/` enforces Route/DoD readiness and tier routing:

- `RouteDoDValidator`: validates required intake fields (`task_title`, `target`, `execution_route`, `acceptance_criteria`)
- `TierRouter`: routes into `tier2` (lightweight) or `tier3` (contract-first)

`orchestrator/contract/` compiles tier3 master contracts from intake:

- writes `master_contract.md`
- writes `template_snapshot.md`
- writes `conformance_report.json` (template heading conformance)

`orchestrator/engine/controller.py` ties everything together:

1. `prepare_task(...)` -> intake validation + tier decision + optional contract compile
2. `prepare_next_task(...)` -> triage queue selection + scoreboard/trend update + preparation
3. `dispatch_step_with_logging(...)` -> shim dispatch + strict result parsing + `workflow_logger` append
4. `run_next_cycle(...)` -> prepare-next + QA-reviewed dispatch + optional rework + queue finalization

Run tests:

```bash
python -m unittest tests.test_intake
python -m unittest tests.test_contract_compiler
python -m unittest tests.test_controller_engine
python -m unittest tests.test_triage_queue
python -m unittest tests.test_scoreboard_store
```

## Progress + Triage (P1)

`progress/` is the shared state root:

- `progress/task_queue.json` (triage queue)
- `progress/scoreboard.json` (counters + last selected task)
- `progress/trends.json` (time snapshots)
- `progress/summary.md` (human-readable overview)

Code modules:

- `orchestrator/triage/` (task model, policy, queue operations)
- `orchestrator/scoreboard/` (sync counters, trend append, summary render)

## Guards + Runtime Wrapper (P2)

Guard tests:

- `tests/guards/test_repository_guards.py`
- `tests/guards/test_ci_workflows.py`

Runtime wrapper:

- `orchestrator/runtime/wrapper.py`
- Enforces `max_runtime_sec`, `max_silence_sec`, `max_memory_mb` (best-effort), and `max_parallel_steps`.

CI workflows:

- `.github/workflows/guard.yml` (guard-only checks)
- `.github/workflows/pipeline.yml` (full tests + demo smoke)

Run guard tests:

```bash
python -m unittest discover -s tests/guards -p 'test_*.py' -v
```

## QA Review + Rework Compiler (P3)

`orchestrator/qa/` adds controller-side quality gate logic:

- `StepQAReviewer`: validates acceptance coverage, required command success, scope guard conformance, and evidence path policy.
- `ReworkBuilder`: compiles strict `ReworkPack` payloads from failing QA reviews.

Engine integration:

- `ControllerEngine.review_step_result(...)`
- `ControllerEngine.dispatch_step_with_review(...)`

These APIs let the controller produce deterministic rework payloads from subagent output instead of hand-written feedback.

## Cycle Closure (P4)

`ControllerEngine.run_next_cycle(...)` executes one end-to-end control loop:

1. select and prepare next triage task
2. dispatch step with QA gate
3. optionally dispatch rework and resume step (`max_rework_rounds`)
4. finalize queue state (`completed`, `pending`, or `failed`)
5. sync scoreboard and append trend snapshot

CLI entrypoint: `python -m orchestrator run-cycle ...`

## Campaign Autopilot (P5)

`ControllerEngine.run_campaign(...)` executes repeated cycle closures over the triage queue:

1. pick next eligible task by policy
2. resolve task-specific step pack (`task_key -> step_pack`)
3. run one `run_next_cycle(...)`
4. continue until queue drains or `max_cycles` is reached

P5 also supports:

- auto planner fallback for missing step packs (`allow_auto_plan_missing`)
- stagnation stop guard (`max_stagnant_cycles`) to avoid long empty loops

CLI entrypoint: `python -m orchestrator run-campaign ...`

## Local Hardening (P6)

P6 adds local single-user hardening without introducing database/service complexity:

1. Negotiation freeze from requirement markdown
2. Pre-dispatch contract guard against frozen plan/boundary
3. Local checkpoint/rollback (`undo`) before risky cycle execution
4. Structured session event log for machine-readable replay

Key modules:

- `orchestrator/negotiation/planner.py`
- `orchestrator/negotiation/guard.py`
- `orchestrator/recovery/checkpoint.py`
- `orchestrator/session_log/store.py`

## AI Controller + AI Worker Split (P7)

P7 keeps the code kernel as state machine, but allows model-level separation:

1. AI controller generates plan/boundary artifacts from dialogue.
2. Code kernel enforces protocol, queue transitions, QA gate, and rollback.
3. AI worker executes steps with task-level backend routing.

Key updates:

- `skills/workflow-local-orchestrator/scripts/dialogue_to_requirements.py`
  - supports `--controller-backend heuristic|command`
  - supports `--controller-model <label>`
  - supports `--controller-command "<external command>"`
  - writes `input/controller_profile.json`
- `orchestrator/worker/shim_adapter.py`
  - supports per-dispatch backend override
- `orchestrator/engine/controller.py`
  - supports `worker_backend_by_task` in `run_campaign(...)`
- `orchestrator/cli.py`
  - adds `run-campaign --worker-backend-map-file`
  - adds `run-cycle/dispatch-step/dispatch-rework --worker-backend`

## Dialogue Freeze Workflow (P0+P1)

Recommended production entry is dialogue-first:

1. Generate `requirements.draft.md` from natural language.
2. Approve and freeze into `requirements.vN.md` + `requirements.current.md`.
3. Execute only when freeze is approved.

Key scripts:

- `skills/workflow-local-orchestrator/scripts/dialogue_to_requirements.py`
- `skills/workflow-local-orchestrator/scripts/freeze_requirements.py`
- `skills/workflow-local-orchestrator/scripts/dialogue_pipeline.ps1`
- `skills/workflow-local-orchestrator/scripts/run_campaign_safe.ps1`

Execution guard:

- `run_campaign_safe.ps1` blocks when `input/freeze_meta.json` is missing/unapproved
  or `input/requirements.current.md` is missing.
- blocked runs log `execution_blocked_unapproved`.
- successful starts log `execution_started`.

Backend routing:

- `input/worker_backend_map.json` maps `task_key -> backend`.
- when map contains one unique backend value, wrapper still passes `--backend <value>`.
- when map contains multiple backend values, wrapper passes
  `--worker-backend-map-file` so campaign dispatches per-task backend.
- `run_campaign_safe.ps1` preflight now validates backend health for runnable tasks
  (shim executable + backend runtime binaries). Use `-SkipBackendHealthCheck`
  only for local mock/testing.

## Orchestrator CLI

Use `python -m orchestrator ...`:

```bash
# 1) intake + tier route (+ optional contract compile for tier3)
python -m orchestrator prepare \
  --task-file ./task.json \
  --run-id T-013_20260219T120000Z_ab12cd3 \
  --task-id T-013

# 0) negotiate requirement markdown into frozen plan/boundary snapshots
python -m orchestrator negotiate \
  --requirements-md ./requirements.md \
  --progress-dir ./progress

# 1.1) select next task from triage queue and update scoreboard/trends
python -m orchestrator prepare-next \
  --run-id T-013_20260219T120000Z_ab12cd3 \
  --queue-path ./progress/task_queue.json \
  --progress-dir ./progress

# 2) dry-run shim command preview for one step
python -m orchestrator dispatch-step \
  --step-pack-file ./step_pack.json \
  --dry-run \
  --max-runtime-sec 120 \
  --max-silence-sec 30 \
  --max-memory-mb 1024 \
  --max-parallel-steps 1

# 3) execute step and write R/S attempt log
python -m orchestrator dispatch-step \
  --step-pack-file ./step_pack.json \
  --log-project-id demo-project \
  --log-root-dir ./workflow

# 3.1) execute + QA review + auto-generate rework payload on fail
python -m orchestrator dispatch-step \
  --step-pack-file ./step_pack.json \
  --review \
  --write-rework-file ./rework_pack.generated.json

# 4) rework dry-run
python -m orchestrator dispatch-rework \
  --rework-pack-file ./rework_pack.json \
  --thread-id thread_abc \
  --dry-run \
  --max-runtime-sec 120 \
  --max-silence-sec 30

# 5) offline review: compare StepPack + StepResult and compile rework
python -m orchestrator review-step \
  --step-pack-file ./step_pack.json \
  --step-result-file ./step_result_fail.json \
  --write-rework-file ./rework_pack.generated.json

# 6) one-cycle closure: prepare-next + review + optional rework + queue finalize
python -m orchestrator run-cycle \
  --run-id T-013_20260219T120000Z_ab12cd3 \
  --step-pack-file ./step_pack.json \
  --queue-path ./progress/task_queue.json \
  --progress-dir ./progress \
  --rework-thread-id thread_abc \
  --max-rework-rounds 1

# 6.1) cycle with frozen contracts + checkpoint/rollback
python -m orchestrator run-cycle \
  --run-id T-013_20260219T120000Z_ab12cd3 \
  --step-pack-file ./step_pack.json \
  --queue-path ./progress/task_queue.json \
  --progress-dir ./progress \
  --plan-snapshot-file ./progress/plan_snapshot.json \
  --boundary-snapshot-file ./progress/boundary_snapshot.json \
  --auto-checkpoint \
  --rollback-on-fail

# 7) multi-cycle autopilot over queue with task-specific step packs
python -m orchestrator run-campaign \
  --step-pack-map-file ./step_pack_map.json \
  --queue-path ./progress/task_queue.json \
  --progress-dir ./progress \
  --max-cycles 20

# 7.1) enable planner fallback + stagnation stop
python -m orchestrator run-campaign \
  --step-pack-map-file ./step_pack_map.json \
  --queue-path ./progress/task_queue.json \
  --progress-dir ./progress \
  --auto-plan-missing \
  --max-stagnant-cycles 5 \
  --max-cycles 20

# 8) manual checkpoint + rollback
python -m orchestrator checkpoint-create \
  --task-key T-013 \
  --run-id T-013_20260219T120000Z_ab12cd3 \
  --file src/parser.py \
  --file tests/test_parser.py

python -m orchestrator rollback \
  --checkpoint-id T-013_20260219T130000Z_ab12cd3
```

## Task Brief Template

If you forget the requirement format, reuse:

- `docs/templates/task_brief.template.md`
- `docs/templates/task_brief.example.md`

In a new chat, provide:

1. project path
2. task brief markdown path
3. execution stage (`draft-only`, `freeze`, or `execute`)

## One-Click Demo

This repository includes mock end-to-end demo assets under `examples/orchestrator/`.
The demo does not require a real external shim binary.

Run:

```bash
python examples/orchestrator/demo_runner.py
```

Or:

```bash
powershell -ExecutionPolicy Bypass -File scripts/demo_orchestrator.ps1
```

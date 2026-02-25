# Rework Ticket

Use this template when a step fails QA and must be sent back for targeted rework.

## Failed Gate

- Gate ID:
- Gate condition that failed (quote verbatim):

## Missing Evidence

- Missing or invalid evidence paths:
- Expected evidence format:

## Minimal Fix List

1. Smallest code/content change needed:
2. Verification command(s) to re-run:
3. Evidence files to regenerate:

## Re-run Command

```bash
# Example
python -m orchestrator dispatch-rework --rework-pack-file ./rework_pack.json
```

## Scope Guard

- Allowed paths:
- Forbidden paths:

---
name: dataset-ready-check
description: >
  Validate that a benchmark sample (a legacy JSP project wired into the
  JSP-decoupling benchmark) is ready for release: the two commands
  (`make setup`, `make eval`) work, the harness self-check passes on the
  pristine repo, tamper detection works, and the environment is
  deterministic. Use when asked to "check if this sample is dataset
  ready", "validate the benchmark sample", or "is this sample ready to
  publish".
---

# Dataset-ready check for a benchmark sample

Run this against ONE sample (e.g. `environments/mamute/`). Every check
must pass; a sample that fails any check is not publishable. Record the
result in `.out/<sample>-ready-report.md`.

## The two commands under test

1. `make setup SAMPLE=<name>` — builds the image, starts compose with
   `--wait`, and confirms the app answers its healthcheck.
2. `make eval SAMPLE=<name> TASK=<task> AGENT=pristine` — runs the
   harness with **no agent changes** (pristine checkout).

## Checks (in order — stop at first hard failure)

### 1. Setup works from clean state
- `docker compose down -v` then `make setup` from scratch.
- No network access required beyond the initial image pull (deps must be
  vendored in the image).
- Time the cold start; record it. > 15 min is a usability problem.

### 2. Pristine self-check (the most important check)
- `make eval ... AGENT=pristine` must produce **all four gates green**:
  build, parity replay, DB state, no-server-HTML (for tasks where the
  flow is not yet decoupled, no-server-HTML is expected to FAIL — record
  it as the task's starting state, not a sample defect; parity and DB
  state MUST pass).
- If parity fails on the pristine repo, **the comparator or the baseline
  is broken, not the agent**. Fix the normalization before publishing.
  This check is what makes every future agent score meaningful.

### 3. Determinism
- Run the pristine eval **twice** (fresh containers each time). Verdicts
  must be identical. Any flakiness (cache TTL races, clock drift, locale
  differences) must be fixed by pinning, not by loosening the comparator.

### 4. Tamper detection
- Modify one byte of a baseline fixture under `bench/` → eval must
  report `TAMPERED`, not pass.
- Relax a comparator (e.g. make parity always-true) → `TAMPERED`.
- Restore, re-run, confirm green again.

### 5. No absolute paths / no machine coupling
- `grep -r '/home/\|/usr/lib/jvm' environments/ bench/ tasks/` → must be
  empty. Everything machine-specific lives inside the Dockerfile.
- The sample must build on a machine with only docker + make installed.

### 6. Test structure and task spec completeness
The `bench/` layout must follow the modular convention:

```
bench/
├── MANIFEST.sha256
├── baselines/<sample>/<task>/{http,dom,db}/
├── db/db_validation.py + db/<test_name>/{spec.json, expected_delta.json}
├── e2e/e2e_validation.py + e2e/<test_name>/{spec.json, flow.py}
└── harness/ (shared libs only: normalize.py, state.py, manifest.py)
```

- Orchestrators (`db_validation.py`, `e2e_validation.py`) are **generic** —
  identical across samples; they discover test dirs and aggregate
  verdicts. Never sample-specific code in them.
- Each test dir is self-contained: `spec.json` (tables/routes, key
  columns, ignore_columns, task ref, locale) + its fixture/flow.
- Test dirs named `<task>__<behavior>`; orchestrator must discover and
  run every dir (run with one test dir temporarily renamed → orchestrator
  must skip it cleanly and report it as not-run, not crash).
- DB tests run in isolation: orchestrator resets the DB between tests.
- For each task JSON: prompt, tier, baseline reference, success criteria
  (explicitly stating legacy unit tests are out of scope), and the
  verification scope (routes/pages covered by the baseline).

### 7. Docs
- README section for the sample exists (what it is, how to run).
- AGENTS.md for the sample exists (agent contract: bench/ read-only,
  unit tests out of scope, what "done" means).

## Report format

```markdown
# Dataset-ready: <sample>

**Verdict:** READY | NOT-READY (<checks passed>/7)

| Check | Result | Notes |
|---|---|---|
| 1. Setup from clean state | pass/fail | cold start: X min |
| 2. Pristine self-check | pass/fail | parity X/X, db pass, gates... |
| 3. Determinism (2 runs) | pass/fail | |
| 4. Tamper detection | pass/fail | |
| 5. No absolute paths | pass/fail | |
| 6. Task specs complete | pass/fail | N tasks |
| 7. Docs | pass/fail | |
```

## Anti-goals

- Do NOT loosen normalization to make check 2 pass — fix the root cause.
- Do NOT accept "works on my machine" — check 5 exists for that.
- Do NOT publish a sample whose pristine self-check is red; every score
  produced on it is meaningless.

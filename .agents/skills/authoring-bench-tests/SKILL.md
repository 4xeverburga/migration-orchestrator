---
name: authoring-bench-tests
description: >
  Author DB-state and E2E tests for a JSP-decoupling benchmark sample:
  the per-test directory convention, the delta pattern for DB state,
  Playwright flow patterns for E2E, baseline capture, and the review
  checklist that keeps tests deterministic and tamper-proof. Use when
  asked to "create bench tests", "write db/e2e tests for this sample",
  "capture a baseline", or "add a test for this flow".
---

# Authoring bench tests (DB + E2E)

This is the hardest authoring task in the benchmark — a bad test poisons
every agent score run against it. Follow the conventions exactly; the
`dataset-ready-check` skill will reject samples that deviate.

## Sample independence (non-negotiable)

Every sample must be fully independent of every other sample. A sample
is a self-contained universe: nothing it starts, reads, or writes may
touch another sample's state, and nothing another sample does may break
it. Concretely:

- **Own stack**: each sample ships its own compose project (own project
  name, own network, own volumes). Never share a compose file, network,
  or volume across samples.
- **Own data**: own database container name, own DB name, own seed data.
  Two samples must never point at the same database or schema.
- **Own ports**: allocate a disjoint port range per sample (app, DB,
  mocks). Document the allocation in the sample's AGENTS.md; the
  orchestrator keeps the global port map. Port collisions between
  samples are a release blocker.
- **Own credentials**: bootstrap/admin credentials are per-sample
  (gitignored file inside the sample, or seeded in-image). Never source
  another sample's credential file, and never assume a shared admin
  password across samples.
- **Own lifecycle**: `make setup` / teardown of one sample must not
  start, stop, restart, or reconfigure another sample's containers or
  the host beyond its own stack (no shared Tomcat, no shared JVM, no
  `pkill` patterns that could match another sample's processes).
- **No cross-references**: tests, fixtures, and flows in sample A never
  reference sample B's URLs, services, or fixtures — even in comments
  or docs, since docs rot into coupling.

Why this is a hard rule: samples are developed, evaluated, and re-run
in arbitrary order and sometimes concurrently. Any shared state (port,
DB, credential, container name) turns one sample's setup or teardown
into another sample's flaky failure, and flaky samples poison every
agent score run against them.

## Directory convention

One test = one directory under `bench/db/` or `bench/e2e/`, named
`<task>__<behavior>`:

```
bench/db/vote-flow__upvote-increments-count/
├── spec.json              # tables, key_columns, ignore_columns, task ref
└── expected_delta.json    # frozen golden delta (captured, never hand-written)

bench/e2e/vote-flow__vote-widget-renders/
├── spec.json              # routes, locale, viewport, task ref, auth mode
└── flow.py                # Playwright flow: login → navigate → act → snapshot
```

Rules:
- **Self-contained**: a test dir carries everything it needs; no imports
  from sibling test dirs. Shared code lives only in `bench/harness/`.
- **Orchestrators are generic**: `db_validation.py` / `e2e_validation.py`
  discover `*/spec.json` and run each dir in isolation. Never add
  sample-specific code to an orchestrator — put it in the test dir or
  `harness/`.
- **Fixtures are captured, not hand-written**: run the flow against the
  pristine app, freeze the output, commit it. Hand-authored expectations
  drift from reality.

## DB tests — the delta pattern

Never compare absolute dumps. Compare **deltas** (before → after), so
seed data, auto-increment offsets, and unrelated rows cancel out.

`spec.json`:
```json
{
  "task": "mamute/vote-flow",
  "tables": [
    {"name": "votes", "key_columns": ["id"], "ignore_columns": []},
    {"name": "questions", "key_columns": ["id"],
     "ignore_columns": ["last_updated", "view_count"]}
  ],
  "flow": "e2e/vote-flow__upvote-increments-count/flow.py"
}
```

Authoring steps:
1. Identify the tables the flow touches (trace the DAO/SQL, then confirm
   empirically by diffing full dumps during development).
2. Declare `ignore_columns` for every volatile column (timestamps,
   counters you don't assert on, uuids). When in doubt, ignore more —
   a missed volatile column causes flaky failures; a missed real column
   shows up in review.
3. Run the flow on the pristine app; freeze `delta(before, after)` as
   `expected_delta.json`.
4. Normalize types on dump: dates → ISO strings, decimals → strings,
   so JSON comparison is stable across driver versions.
5. The harness DB user must be **read-only** — tests observe, never mutate.

## E2E tests — Playwright flow patterns

`flow.py` exposes one function the orchestrator calls:
`run(page, base_url) -> dict` returning the artifacts (DOM snapshots,
HAR path). Patterns that recur in every legacy-JSP flow:

- **Login/session**: perform real login once per flow (session cookie +
  CSRF token are handled by the browser context). Never replay stale
  cookies across runs.
- **Pinning**: set `locale="en-US"`, `timezone_id="UTC"` on the browser
  context — i18n and clock drift are the #1 source of false failures.
- **Waiting**: never `sleep()`. Wait for signals — `page.wait_for_response("**/vote*")`
  after clicking vote, `page.wait_for_load_state("networkidle")` after
  navigation. If a legacy page has no clean signal, wait for the specific
  element that proves the action landed.
- **Snapshots**: capture `page.content()` after the action settles;
  normalize (strip `<script>/<style>`, volatile attrs, whitespace) before
  storing. Compare structure + text, never class names or inline styles.
- **HAR**: record with `record_har_path` so one run yields both the DOM
  and the full request/response set (including XHRs the page makes).

## Baseline capture workflow

1. `make setup SAMPLE=<name>` — pristine environment up.
2. For each test dir: run its flow via the orchestrator in **capture
   mode** → writes fixtures under `bench/baselines/<sample>/<task>/`.
3. Freeze: regenerate `bench/MANIFEST.sha256`.
4. Run the pristine self-check: eval with no agent changes must be green
   (parity + DB). If red, fix normalization — never loosen it to pass.

## Review checklist (every test must satisfy)

- [ ] Deterministic: two consecutive runs produce identical artifacts
      (run twice during authoring, not just at release).
- [ ] Isolated: passes when run alone AND when run after all other tests
      (DB reset between tests; no order dependence).
- [ ] Minimal: asserts the flow's observable behavior, nothing more.
      A test that dumps 40 tables is unmaintainable — trim to what the
      task's verification scope covers.
- [ ] No sleeps, no absolute paths, no network beyond the compose stack.
- [ ] Named `<task>__<behavior>`; `spec.json` references its task.
- [ ] Covered by `MANIFEST.sha256` (regenerate after any edit).

## Anti-patterns

- Hand-editing `expected_delta.json` to make a failing test pass —
  re-capture from the pristine app instead, and understand *why* it
  differed.
- Asserting on CSS classes/inline styles — decoupling legitimately
  changes markup; assert structure and text.
- One giant flow testing five behaviors — split into per-behavior dirs;
  granularity is what makes verdicts diagnosable.
- Tests that depend on execution order — the orchestrator may parallelize
  or reorder in the future.

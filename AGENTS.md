# AGENTS — JSP→REST modernization training orchestrator

This folder is the **orchestrator for the JSP-decoupling training harness**:
it assesses legacy JSP projects, turns them into dataset samples, and
coordinates the agents that train/eval on "decouple the frontend from the
backend with REST APIs".

The legacy-app runbooks that used to live here moved to their own projects
(e.g. `../modernizacion-tinka-pcr21/lotto-sale-desktop/AGENTS.md`).

---

## 1. What lives here

| Path | Purpose |
|---|---|
| `.agents/skills/assess-jsp-decoupling-candidate/SKILL.md` | The assessment skill: scores a legacy JSP project as a dataset candidate (rubric, blocker classes, difficulty tiers, teaching cases). **Read it before assessing any project.** |
| `.agents/skills/dataset-ready-check/SKILL.md` | The release gate: validates a sample's `make setup` / `make eval` work, the pristine self-check passes, tamper detection works, and the environment is deterministic. **Run it before publishing any sample.** |
| `.agents/skills/authoring-bench-tests/SKILL.md` | How to author the per-test dirs under `bench/db/` and `bench/e2e/` (delta pattern, Playwright flow patterns, baseline capture, review checklist). **Read it before writing any bench test.** |
| `.out/` | **All generated outputs** (assessment reports, baselines, captured slices). Gitignored — never commit anything here. |
| `.gitignore` | Ignores `.out/`. |

Published repo: https://github.com/4xeverburga/migration-orchestrator

## 2. The pipeline this orchestrator coordinates

```
assess candidate ──► capture golden baseline ──► generate slice tasks ──► run slice-decoupling subagent ──► verify parity
      (skill)            (per slice)                (tiered 1–4)             (the trained runner)            (vs baseline)
```

1. **Assess** — apply the skill to a candidate project; verdict + score +
   slice tasks go to `.out/<project>-assessment.md`.
2. **Baseline** — for each slice, capture HTTP request/response pairs,
   rendered DOM, and DB state transitions *before* any change. This is what
   makes "decouple" falsifiable.
3. **Feature tasks** — the assessment lists tiered, feature-scoped eval
   tasks (Tier 1 anchor → Tier 4 heavy composition root). The boundary is
   a **verification scope, not an edit scope**: the agent may modify
   anything it needs (shared tags, interceptors, other pages); what is
   bounded is the golden baseline it is graded against. Small samples also
   get unscoped end-to-end tasks ("decouple the write paths of this app").
4. **Runner** — the slice-decoupling agent executes one feature boundary:
   RESTify the controllers, remove the JSP/tag rendering, keep behavior
   parity. It chooses its own edits; the baseline decides pass/fail.
5. **Verify** — replay the baseline against the decoupled slice; parity +
   "no server-rendered HTML left for that slice" + build/tests green.

## 3. Known dataset samples (so far)

| Project | Verdict | Notes |
|---|---|---|
| `../mamute/` | strong ACCEPT | 49 JSPs, 78 tag files, 5 JSON controllers; runbook in its `AGENTS.md`. Tier-1 anchor: `VoteController`. |
| `../openclinica/` | ACCEPT-WITH-FIXES (78%) | 807 JSPs — **scope to one bounded flow** (~30–60 JSPs), not the monolith. Report in `.out/openclinica-assessment.md`. |

## 4. Conventions

- **Outputs go to `.out/`** — assessments, baselines, run logs. The repo
  itself holds only durable knowledge: skills, AGENTS.md, stable docs.
- **Skills are repo-local** (`.agents/skills/`) and versioned here; do not
  duplicate them into `~/.agents/skills/`.
- **Absolute paths** for machine-specific things (JDK 8 at
  `/usr/lib/jvm/jdk-8.0.202-oracle-x64`; never inherit the shell
  `JAVA_HOME` — machine default is JDK 21 and breaks every legacy build).
- Ports in use across the workspace: 8080 (lotto Tomcat), 8088 (Mamute),
  8090 (HTTP mock), 8099 (E2E stub). Pick fresh ports for new samples.

## 5. Next steps (not yet built)

- [ ] Baseline-capture harness (request/DOM/DB recorder per slice)
- [ ] Slice-decoupling runner agent + eval on ~10 slices across 2–3 projects
- [ ] Planner/orchestrator agent on top (tag-usage graph → decoupling order)

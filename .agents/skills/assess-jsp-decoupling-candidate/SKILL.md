---
name: assess-jsp-decoupling-candidate
description: >
  Assess whether a legacy JSP-based web project is worth adding to the
  JSP-decoupling dataset (the harness that trains/evals agents on
  "decouple frontend from backend with REST APIs"). Produces a scored
  verdict with evidence. Use when asked to "assess this project for the
  dataset", "is this project a good decoupling candidate", or "evaluate
  a JSP project for modernization opportunities".
---

# Assess a JSP project as a decoupling-dataset candidate

## Goal

Decide, with evidence, whether a project is a good sample for the
JSP-decoupling dataset. The dataset's tasks look like:

- *slice task*: "Decouple the answer flow: `XController` + `xForm.jsp` + `x.tag`"
- *tag task*: "Extract a shared vote-widget tag into a REST-backed component used by N pages"
- *planning task*: "Given this tag-usage graph, produce a decoupling order"

A good sample is one where those tasks are **non-trivial but verifiable**.

## Priorities (read this before scoring)

1. **JSTL/view-layer density and modernization opportunity dominate.** A
   project rich in JSTL logic, tag files, and server-rendered fragments is
   valuable even if it is currently hard to run.
2. **Runnability is a soft gate, not a hard gate.** Do NOT reject a project
   because it doesn't build today. Instead, classify the blocker:

   | Blocker class | Verdict impact |
   |---|---|
   | Wrong JDK on PATH (needs JDK 7/8, machine has 9+) | **Trivial** — never reject |
   | Dead Maven repos / missing artifacts (jcenter, defunct project repos) | **Easy** — pin mirrors or vendor jars into `local-repo/` |
   | Old plugin versions incompatible with modern Maven | **Easy** — pin plugin versions |
   | DB needed but containerizable (MySQL 5.7, Postgres, H2) | **Easy** — docker/podman compose |
   | Needs an unreachable external service (Oracle SP, SOAP, license server) | **Medium** — mockable? if yes, note the mock effort; if no, reject |
   | Proprietary/licensed deps that cannot be vendored | **Hard** — reject |
   | Build broken in ways beyond the above after ~1 day of effort | **Hard** — reject |

   Budget: if the project could plausibly run with **≤ 1 day** of setup
   tweaks, treat it as runnable. Record the exact fix steps in the report —
   they become part of the sample's setup documentation.

## Scoring rubric (0–3 each, weighted)

Gather evidence by reading code, not by guessing. Cite file paths (and
line numbers where useful) for every score above 0.

### A. View-layer modernization opportunity (weight ×3)
- Count JSPs; count how many contain real logic (`c:forEach`, `c:if`,
  `c:choose`, `fmt:`, `spring:`, custom tags). A project where most pages
  are static forms scores low; heavy composition scores high.
- Tag files / custom taglibs: count them and sample 3–5 for reuse breadth
  (how many pages use the same tag). Shared tags = interesting coupling.
- Server-rendered fragments produced from Java (controllers forwarding to
  `.jspf` pagelets, HTML built in Java) — these are prime REST targets.

### B. REST precedent (weight ×2)
- Existing JSON endpoints (`Results.json`, `@ResponseBody`, `produces =
  application/json`, Jackson/Gson usage in controllers). Any precedent
  proves the target pattern exists in-codebase and makes tasks verifiable.
- Existing CORS handling or API-ish controllers are a bonus signal.

### C. Separation of concerns (weight ×2)
- Are controllers thin (logic in model/DAO/service) or fat (logic inline
  in controller methods)? Thin = mechanical transformation; fat = the
  dataset teaches harder reasoning (still valuable, cap at 2–3 fat
  controllers per sample).
- Session-scoped state, interceptors/filters injecting view data
  (`result.include` from interceptors) — note them; they define the auth
  and context problem the final agent must solve.

### D. Verifiability (weight ×2)
- Can behavior be captured as a golden baseline? (HTTP request/response
  pairs, rendered DOM, DB state transitions.)
- Existing tests? Even a handful of integration tests raises the score.
- DB writes that are observable (vote counts, rows created) make parity
  checks mechanical.

### E. Auth realism (weight ×1)
- Session-based auth, OAuth social login, LDAP/SSO filters — all good:
  the dataset *should* contain the auth problem. Score 3 for a realistic
  mix, 1 for no auth at all (too easy), 0 only if auth is unfixably
  proprietary.

### F. Practicality (weight ×1)
- License (Apache/MIT/BSD = fine; GPL = fine for local dataset use, note it).
- Size sweet spot: ~20–150 JSPs. Below 10 = too easy; above ~300 = unusable
  without decomposition (note "needs project-decomposition pass").
- Build system present and identifiable (Maven/Gradle/Ant).

## Verdict scale

- **≥ 75% weighted** → `ACCEPT` — add to dataset; list the top 3–5 slice
  tasks it enables.
- **50–74%** → `ACCEPT-WITH-FIXES` — list the concrete setup fixes and the
  slices that would become available after them.
- **< 50%** → `REJECT` — say which criterion is unfixable and why.

## Report format

```markdown
# Assessment: <project name>

**Verdict:** ACCEPT | ACCEPT-WITH-FIXES | REJECT (<score>% weighted)

## Scores
| Criterion | Score | Weight | Evidence |
|---|---|---|---|
| A. View-layer opportunity | 0–3 | ×3 | <files/counts> |
| B. REST precedent | 0–3 | ×2 | <files> |
| C. Separation of concerns | 0–3 | ×2 | <files> |
| D. Verifiability | 0–3 | ×2 | <files> |
| E. Auth realism | 0–3 | ×1 | <files> |
| F. Practicality | 0–3 | ×1 | <files> |

## Runnability
- Status: runs today | fixable (≤1 day) | hard blocker
- Exact fix steps (if any): <numbered list>
- Runtime requirements: <JDK, DB, ports, env vars>

## Slice tasks this project enables
1. <controller + jsp + tags, difficulty tier>
2. ...

## Risks / notes
<anything that will bite the harness: caching, i18n bundles, asset
generation (grunt/npm), relative log paths, port conflicts>
```

## Difficulty tiers for the slice tasks

When listing slice tasks, tier them so the dataset has spread:

- **Tier 1 (anchor):** controller method already returns JSON or is one
  `forwardTo` away from it (e.g. a vote/comment endpoint).
- **Tier 2:** simple form page + its controller; no shared tags.
- **Tier 3:** page composed of reused tag files; change must be coordinated
  across pages.
- **Tier 4:** heavy composition root (question-detail-style page) with
  interceptors, caching, and auth branching.

A good sample yields at least one task per tier. If a project only yields
Tier 1–2, it is a weak sample even if it runs perfectly.

## Teaching cases worth including

When listing slice tasks, actively look for these recurring legacy-JSP
patterns — each teaches a distinct reasoning skill the final agent needs.
A strong sample contains several of them; note which ones a project has.

- **Fat servlets bypassing the MVC framework.** Hand-rolled servlets
  registered directly in `web.xml` alongside the framework's front
  controller. They are the hard "fat controller" reasoning cases — cap at
  2–3 per sample (see criterion C).
- **Client-state grid/component frameworks.** Table/grid taglibs that keep
  server-side state (paging, sorting, filters). Decoupling one is a
  dedicated Tier-3 task; do not treat it as a replicable pattern.
- **Heavy i18n.** `fmt:` bundle usage throughout the views — golden
  baselines must pin a locale or parity checks will false-fail.
- **Cached server-rendered pagelets.** Controllers that forward to
  fragment templates behind a cache (page-level or fragment-level).
  Baseline capture must account for cache TTLs; decoupling must decide
  who owns the cache.
- **Asset-generation pipelines.** grunt/npm/webpack steps that rewrite
  template references (hashed asset names, generated CSS). Setup docs and
  baselines must include the asset build, or pages render unstyled.
- **OAuth/social login alongside session auth.** Multiple identity paths
  (DB password + OAuth + SSO filters) — the auth-handling slice every
  final agent must solve at least once.

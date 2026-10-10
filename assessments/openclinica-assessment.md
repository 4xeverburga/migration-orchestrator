# Assessment: OpenClinica (3.18-SNAPSHOT)

**Verdict:** ACCEPT-WITH-FIXES (78% weighted) — accept **scoped to a bounded slice**, not the whole monolith.

## Scores

| Criterion | Score | Weight | Evidence |
|---|---|---|---|
| A. View-layer opportunity | 3 | ×3 | 807 JSPs (419 `web/` + 388 `ws/`), **776 contain JSTL logic (96%)**. Ordinary admin pages carry 33–35 JSTL constructs each (`viewSubjectSDV.jsp`, `createExportJob.jsp`, `deleteEventCRF.jsp`). 16 `.tag` files + 4 custom TLDs (`openclinica.tld`, `view_tags.tld`, `formtags.tld`, `jmesa.tld`); 71 JSPs use custom tags. jmesa grid component in 36 JSPs. |
| B. REST precedent | 3 | ×2 | Real precedent: `ODMMetadataRestResource`, `ODMClinicaDataResource`, `OpenRosaServices`, ~25 Spring MVC `*Controller.java` with `@RequestMapping`. `ws/` module adds Spring-WS SOAP endpoints (`CrfEndpoint`, `DataEndpoint`, `StudyEndpoint`). JSON/XML serialization path proven in-codebase. |
| C. Separation of concerns | 2 | ×2 | Hybrid stack: Spring MVC 3.2 controllers coexist with **hand-rolled servlets registered directly in `web.xml`** (`org.akaza.openclinica.control.extract.ChooseDownloadFormat`, `InitialDataEntryServlet`, `SQLInitServlet`). The legacy servlet layer is the fat part; Spring controllers are comparatively thin. |
| D. Verifiability | 2 | ×2 | Only 17 test files. DB is PostgreSQL (containerizable). Clinical data flows (CRF entries, event CRFs, export jobs) are observable DB state → parity checks are mechanical. No WAR currently built (`target/classes` exist but no `.war`). |
| E. Auth realism | 3 | ×1 | Spring Security with custom `OpenClinicaUsernamePasswordAuthenticationFilter` in both `web` and `ws`. Realistic session-based auth problem — exactly what the dataset should contain. |
| F. Practicality | 2 | ×1 | LGPL (fine for local dataset use, note attribution). **807 JSPs — far above the ~300 sweet spot** → requires a project-decomposition pass. Java 7 source/target, Spring 3.2.18 (2013-era). Maven multi-module; `target/classes` present in `core/` and `web/` → **it has compiled on this machine before**. |

**Weighted total: 28/36 = 78%**

## Runnability

- **Status: fixable (≤1 day)** — likely less; artifacts in `target/` prove a prior successful compile here.
- Expected fix steps:
  1. Pin JDK 8 (`/usr/lib/jvm/jdk-8.0.202-oracle-x64`) — Java 7 source/target fails on JDK 9+.
  2. PostgreSQL container (podman/docker) + `datainfo.properties` config; Liquibase plugin needs `config.file`.
  3. Dead Maven repos (2013-era) → pin mirrors or vendor into `local-repo/` (same pattern as Mamute).
  4. Spring 3.2.18 on JDK 8 is fine; do not attempt to bump Spring as part of setup.
- Runtime requirements: JDK 8, PostgreSQL, Tomcat 9 (or embedded), a port distinct from 8080/8088.

## Slice tasks this project enables

1. **Tier 1 (anchor):** `OdmController` / `ReportController` — already Spring MVC with serialization; convert remaining view coupling.
2. **Tier 2:** a self-contained admin CRUD page + its controller, e.g. `createExportJob.jsp` (34 JSTL constructs, no shared tags).
3. **Tier 3:** a jmesa-grid page (`36 JSPs` use it) — decoupling the grid teaches shared-component reasoning; do as a dedicated tag task.
4. **Tier 4:** `viewSubjectSDV.jsp`-style subject-data page — heavy composition, auth branching, DB-observable parity.

## Risks / notes

- **Size is the main risk, not difficulty.** Do not add all 807 JSPs as one sample. Scope the sample to one bounded flow (e.g. admin/extract or subject SDV) — roughly 30–60 JSPs — and treat the rest as future slices.
- The `ws/` module's 388 JSPs are mostly login/participate fragments — low value; exclude from the initial scope.
- Hand-rolled servlets in `web.xml` bypass Spring MVC — slice tasks must avoid or explicitly include them; they are the "fat controller" teaching cases (cap at 2–3 per the rubric).
- jmesa is a client-side-grid framework with server state — decoupling it is genuinely hard; keep it as one Tier-3 task, not a pattern to replicate.
- LGPL license: fine for a private dataset; keep `LICENSE.txt` with the sample.
- i18n: JSTL `fmt:` bundles are used heavily — baseline capture must pin a locale.

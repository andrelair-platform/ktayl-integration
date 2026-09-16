# Legacy Modernization Practice — "GlobalCore" + the modern wrap (spec)

> **Purpose.** Learn & demonstrate how an insurer delivers modern features (workbench, broker portal,
> document-AI, APIs, analytics) **when the most important business logic lives in a monolith you cannot
> replace.** We build a deliberately-legacy core we *own* — **GlobalCore** — freeze it as the System of
> Record, and modernize *around* it (Anti-Corruption Layer → strangler-fig). This is a **practice/skills**
> build modelled on the ktayl IS (not Retrieva).

> **Decision (2026-09-16):** *not* COBOL. A **Java 8 / SOAP / batch monolith** teaches the same
> modernization architecture without emulator/VSAM pain, and lets us put real insurance logic in it.
> (We don't need HDI's actual proprietary stack — a faithful *fictional* carrier is the right lab.)

## 1. GlobalCore — the deliberately-legacy core (System of Record)

**Stack (authentic-legacy):** Java 8 + Spring (+ spring-ws for SOAP) + JSP + Tomcat (WAR) +
PostgreSQL *pretending to be Oracle/DB2* + stored procedures + cron/**nightly batch** + SFTP/CSV.
**One shared schema, synchronous, XML interfaces, no REST, no Kafka, no containers-native patterns.**

**Runs OUTSIDE k8s** — a plain Docker container on the controller — because legacy isn't cloud-native;
the modern platform (in k8s) reaches *out* to it. That gap is part of the lesson.

**The iron rule:** GlobalCore is **frozen** — you may **not** modify it to add a modern capability.
That constraint forces the real architecture (you must wrap, not edit).

**Real, un-throwaway business logic** (not "intentionally stupid" — the difficulty is the point):
effective-dated policy versions · endorsements · multi-country policies · co-insurance · currencies ·
premium taxes · broker commissions · referrals · claims reserves · facultative reinsurance ·
cancellations/reinstatements · audit history.

**Domains (built incrementally):** Accounts · Brokers · Submissions · Policies · Endorsements ·
Renewals · Rating · Claims · Premiums · Payments · Reinsurance · Regulatory exports.

**Model it on a *real* insurance core:** GlobalCore's data model + flows mirror **IBM's GenApp**
(`cicsdev/cics-genapp`, EPL-2.0) — the canonical open-source **insurance COBOL/CICS** app (customers +
policies + claims, DB2, 3270), which IBM designed to be *extended with web services*. Studying GenApp is
how we keep our fictional core faithful to a genuine one. See §6.

## 1b. Two tracks (which legacy core you wrap) — same wrapping skill either way

- **Track A — GlobalCore (default, runs today).** Our Java 8 / SOAP monolith, modeled on GenApp,
  running as a container on the controller. **No mainframe needed** → this is what we build now.
- **Track B — the *real* GenApp (optional, advanced credential) — NOW FORKED IN.** IBM's actual
  COBOL/CICS GenApp is forked into the org at **`andrelair-platform/cics-genapp`** (fork of
  `cicsdev/cics-genapp`, EPL-2.0) and cloned locally — the real code is present to study now: **31 COBOL
  programs** (`base/src/*.cbl`, e.g. `LGACUS01` add-customer, `LGACDB01` policy DB), **13 copybooks**,
  DB2 DDL, 3270 BMS maps, event bindings, and the **`base/cntl/wsa*.jcl` CICS web-service jobs** — the
  built-in **SOAP wrap hook** (add/inquire customer & policy as web services). **It only *runs* on
  CICS/z/OS** (a mainframe or emulator — IBM Z Xplore / a z/OS trial / Wazi; *not* the k8s cluster), so
  Track B = get an emulator → run GenApp → enable its `wsa*` web services → point the **same** ACL/
  workbench at it. The "I modernized real z/OS COBOL" credential. **The modern wrap is identical to
  Track A**, so all Track-A work transfers directly.
  - **Study it now (no mainframe needed):** read `cics-genapp/base/src/*.cbl` + the copybooks to see how
    a genuine 20-year-old insurance core is structured — that's what GlobalCore (Track A) is modelled on.

## 2. Target architecture (modern platform wrapping GlobalCore)

```
Broker Portal (Next.js) ─┐
Underwriter Workbench ───┤─► API Gateway (OIDC/audit) ─► Submission / Underwriting / Document services
(React/Next.js)          │                                     │ (NestJS/Java + Python AI)
                         │                                     ▼
                         │                           Anti-Corruption Layer (ACL)
                         │                                     │  SOAP / XML / batch
                         │                          ╔══════════▼══════════╗
                         │                          ║   GlobalCore (SoR)  ║  Java8/SOAP/SQL/Batch
                         │                          ║  frozen monolith    ║  (outside k8s)
                         │                          ╚══════════╤══════════╝
                         │                            CDC / events (Debezium → NATS/Kafka)
                         │                                     ▼
                         └───────── read models: Policy (Postgres) · Search (OpenSearch) · Data lake
```
**Golden rule:** modern code **never** touches GlobalCore's DB/SOAP directly — **only the ACL does.**
**Writes** go through the ACL to the SoR; **reads/events** increasingly come from CDC-fed read models
(so the legacy core doesn't become a bottleneck).

## 3. First use case — Underwriting Submission
Broker sends a pack (email + submission.pdf + property_schedule.xlsx + claims_history.pdf +
previous_policy.pdf) → **Document service** (object storage → OCR → classification → entity extraction →
LLM) → **structured submission** → **Underwriter Workbench**. The workbench needs the existing policy →
`GET /api/policies/{id}` → **ACL** → **legacy SOAP** `getPolicy` → the ACL converts the ugly legacy XML
into clean JSON. Modern code never learns the core is Java/SOAP. **AI-Act:** the extraction is a
**limited-risk** use case → human-verify + Presidio + Langfuse + a system card (see `ai-ml/ai-act-gate`).

## 4. Strangler-fig roadmap (progressive)
Start: GlobalCore owns everything. Then move capabilities modern-side one at a time —
Submission → Documents → Search → Underwriting UI → Renewal → … eventually extract Rating into a modern
engine, then Claims, Billing. Each step: the ACL re-routes that slice; the core shrinks. Demoable by
flipping a route.

### Track-B mission — modernize the *real* GenApp onto the cluster (rules-first)
Track B is no longer just "wrap GenApp" — the mission is to **transform it so it can run on the cluster**,
by **recovering its business rules and reimplementing them** as cluster-native services (verified against
the original). **Honest constraint:** GenApp is CICS/DB2/VSAM-bound → you can't recompile-and-run it on
k8s (no OSS CICS runtime); "run on the cluster" = **rules-first reimplementation** (rewrite/strangler),
which is how enterprises actually modernize. Four phases:
- **M1 — Comprehend:** recover each program's rules from the COBOL → the **business-rules catalog**
  (`cics-genapp/docs/business-rules-catalog.md`). *Comprehension is engineer-owned; the catalog has the
  map + method + a worked example (LGACUS01) to catalog the rest against.*
- **M2 — Target design:** map GenApp domains → modern services; COMMAREA → JSON DTOs; return codes →
  errors; VSAM+DB2 dual-write → one modern store; TSQ → structured logs. **GlobalCore is the seed** of
  the modern target.
- **M3 — Reimplement + verify:** build the modern equivalents (GAP wrapper chart) with **equivalence
  tests** derived from M1 (input → expected return code/output; if a z/OS emulator exists, diff vs real
  GenApp).
- **M4 — Run + strangle:** deploy on k8s; the ACL routes traffic off the legacy core to the modern
  services until the COBOL is retired.

The convergence: **GenApp = the source-of-truth for the rules; the modern cluster services = the
transformed target; GlobalCore seeds that target.** Track-A wrapping work + M1–M4 use the same ACL/tests.

## 5. Staged delivery (don't build it all at once)
| Stage | Deliverable | Home |
|---|---|---|
| 0 | this spec | `ktayl-integration/docs/` |
| 1 | **GlobalCore v0** — Java/SOAP core, Policy domain (effective-dated + endorsements + currency) + nightly renewal batch; then **freeze** | new repo `globalcore-legacy`, runs on controller |
| 2 | **ACL + first read** — SOAP→JSON adapter; workbench reads a policy via the ACL only | Integration #25 + Underwriting #12 |
| 3 | **Submission use case** — broker pack → document-AI → workbench → bind **writes** via ACL | DMS #24 + AI #4 |
| 4 | **Strangle** — CDC → events → Policy read model + search; move domains out | Data #5 |

## 6. Reference systems (study, don't necessarily reuse)
- **IBM GenApp — forked in at `andrelair-platform/cics-genapp` (EPL-2.0, from `cicsdev/cics-genapp`) — the
  primary reference + Track-B core.** The canonical open-source **general insurance** application:
  COBOL/CICS/DB2, customers + policies + claims, a 3270 UI, sample data. IBM explicitly designed it to be
  **extended** (web services via the `wsa*` jobs, business events, dashboards) — i.e. wrapped. It's both
  our **domain/data-model reference** (GlobalCore is modelled on it) **and** the **Track-B real core** to
  run on a z/OS emulator. **Now forked into the org + cloned locally**, so the real COBOL/copybooks/DDL/JCL
  are present to study today. Its docs note some parts aren't best-practice *on purpose* — that's the lab.
  EPL-2.0 → attribution preserved by the fork.
- **Elucida Insurance System** — older open-source Java/JEE insurance admin (policy + claims, ~2006–2016);
  a good structural study of a real legacy Java core. *(Verify license before reusing any code.)*
- **Open Insurance Platform** — broad insurance-core domain reference (project still forming) → domain map.
- **CoSure PAS** — a *modern* Go/Python PAS → useful as a **target** shape, **not** the legacy core.
- Other insurance COBOL repos exist but are **unlicensed** → inspiration only, cannot deploy.

## 7. Scope guardrails
- **One domain first** (Policy) with *real* logic; add domains incrementally.
- **Fake the legacy *characteristics*** (SOAP/batch/shared-schema/synchronous), with genuine business
  complexity — do **not** make it trivially stupid.
- **Non-goals:** reproducing HDI's actual internal tech; replacing `ktayl-policy-service`; building all
  domains + the full modern platform up front.

## 8. What you practice
Anti-corruption layer · strangler-fig · SOAP/legacy + batch/file integration · CDC + event-driven read
models (avoiding the legacy-as-bottleneck trap) · document-AI + AI-Act controls · running a non-cloud-native
core alongside a cloud-native platform. The real enterprise/insurance FDE challenge.

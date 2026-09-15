# Legacy-Wrapper Practice Initiative — UC-1 + UC-2 (spec)

> **Purpose.** A deliberately-realistic practice build of the enterprise-insurance **hub-and-spoke /
> strangler-fig** pattern: keep a rigid **legacy core** running underneath, and wrap it with a modern
> **anti-corruption layer (ACL)**, an **underwriting workbench**, and **document-AI ingestion**. The
> trick: *to practice wrapping a legacy core you must first build a believable one.* Scoped to **one LOB
> = Marine**, one flow (submission → bind, + renewal).

> **This is a practice/skills build modelled on the ktayl IS** — realistic, but its primary value is the
> pattern. It advances real boards (Policy #6, Integration #25, Underwriting #12, DMS #24, AI #4).
> Two-layer: ktayl IS, **not** Retrieva.

## 1. Target architecture (hub-and-spoke)

```
   Broker submission (email + PDF slip + Excel)
              │
              ▼
   [UC-2] Document-AI Ingestion  ──(structured submission, human-verified)──┐
   (Docling/markitdown → LLM → Presidio)                                     │
                                                                            ▼
   Underwriter ─► [Underwriting Workbench]  ──REST/JSON──►  [Anti-Corruption Layer]
                  (modern UI, UW-01)                          (REST facade ⇄ legacy)
                        ▲                                        │        │
                        │  (async notify: "issued")             │ NATS   │ Temporal
                        └────────────────────────────────────────┘ events │ bind workflow
                                                                          ▼
                                                        [LEGACY MARINE PAS] (system of record)
                                                        SOAP/WSDL + fixed-width flat-file,
                                                        nightly BATCH rating/issuance,
                                                        rigid schema + code tables
                                                                          │
                                                        (nightly replicate → analytics — future UC-3)
```

**Golden rule:** the workbench + ingestion **never touch the legacy core directly** — only the ACL does.

## 2. The deliberate-legacy constraints (what makes the core authentic)

Fake the **constraints**, not literal COBOL. The Legacy Marine PAS MUST behave legacy:
- **Interface:** **SOAP/XML (a WSDL)** for reads/writes **+** a **fixed-width flat-file** batch drop
  (SFTP-style) for bulk. **No REST, no JSON, no real-time issuance.**
- **Data:** rigid relational schema, **cryptic ≤8-char columns** (`POLNO`, `INSDNM`, `EFFDT`, `PREMAMT`),
  **code tables not enums** (LOB `MAR`, status `A`/`L`/`C`, peril codes).
- **Processing:** a **nightly batch** cron does rating + official issuance — the workbench "submits" and
  the policy only becomes official after the batch runs. No synchronous bind.
- **Auth:** basic-auth / IP-allowlist (no OIDC). No pagination. Chatty, brittle.

That's *enough* to make the ACL genuinely necessary. Stack: a small Go/Node service + Postgres (the
"AS/400 DB2") + a batch job, deployed via the GAP wrapper chart. New repo `ktayl-legacy-marine`.

## 3. UC-1 — Legacy PAS + ACL + Workbench

### 3a. The ACL contract (the learning core)
The ACL exposes a **clean domain REST/JSON API** to the modern side and translates to the legacy SOAP/
flat-file on the other — the **anti-corruption boundary** (domain terms in, legacy codes out):

| Modern (REST/JSON) | → ACL translates to → | Legacy |
|---|---|---|
| `POST /policies` (domain Submission JSON) | maps domain→codes, builds the flat-file row / SOAP env | `issuePolicy` (batch-queued) |
| `GET /policies/{id}` | SOAP `getPolicy` → domain JSON | `getPolicy` (SOAP) |
| `GET /policies/{id}/status` | reads batch/issuance state | poll |
| events out: `policy.issued`, `policy.rejected` | from batch callback | nightly batch result |

The ACL owns the **mapping tables** (domain enum ⇄ legacy code), input validation, and shields every
consumer from the legacy weirdness. This is `ktayl-integration` (board #25).

### 3b. The Temporal bind workflow (sync-over-async avoidance — the key skill)
Binding is **not** synchronous (the core issues only at night). The workbench must not block:
```
Workbench POST /policies ─► ACL starts Temporal workflow "BindMarinePolicy"
   1. validate + map submission → legacy intake (flat-file row)
   2. drop the row for the nightly batch; emit policy.submitted (NATS)
   3. WAIT (durable timer) for the batch cycle
   4. on batch result: getPolicy → confirm official issuance
   5. emit policy.issued (or policy.rejected) → workbench notified (async)
```
Temporal gives durability + retries + the long wait; NATS carries the events; the workbench subscribes
and updates the file when "issued" arrives. **This is the two-speed choreography to master.**

### 3c. The strangler plan
- `ktayl-policy-service` (modern PAS, live) = the **strangler target**.
- `ktayl-legacy-marine` = the **legacy core being strangled**.
- **Routing at the ACL:** *new* Marine business → issue in the modern PAS; *legacy Marine renewals* →
  still route to the old core via the ACL. Over time, more flows move modern-side — the classic
  strangler-fig, demoable by flipping a route.

## 4. UC-2 — Document-AI ingestion → legacy intake

Broker sends a submission pack → **Docling/markitdown** (OCR/convert) → **LLM extraction** → structured
Submission JSON → **human-verify** in the workbench → pre-fill via the **ACL** into the legacy intake.

**AI-Act gate (exercised for real):** this is a **limited-risk** use case (assistive, human-verified,
PII present) → controls: transparency, **human sign-off**, **Presidio** PII masking, **Langfuse** logging,
+ an **AI System Card** (see `ai-ml/ai-act-gate`). The LLM provider also lands in the **DORA third-party
register** (Retrieva). Boards: DMS #24 + AI #4.

## 5. Minimal Marine data model
`Submission` (broker, insured entity, vessel/cargo basics, cover, dates) → `Quote` (terms, premium) →
`Policy` (the legacy record: `POLNO/INSDNM/EFFDT/EXPDT/PREMAMT/STATCD`). Keep it tiny — one peril set.

## 6. Epic map (across boards)

| Epic | Board / repo | Scope |
|---|---|---|
| **LGM-01** legacy Marine schema + code tables | Policy #6 · `ktayl-legacy-marine` | rigid schema, cryptic cols, code tables, seed data |
| **LGM-02** SOAP/WSDL + fixed-width flat-file I/O | #6 · `ktayl-legacy-marine` | `getPolicy`/`issuePolicy` SOAP + batch file import/export |
| **LGM-03** nightly batch rating/issuance | #6 · `ktayl-legacy-marine` | cron engine: reads intake rows → rates → issues → result file |
| **ACL-01** clean REST/JSON facade | **Integration #25** · `ktayl-integration` | domain API for the workbench |
| **ACL-02** anti-corruption translator | #25 · `ktayl-integration` | domain⇄legacy code mapping, SOAP/flat-file adapters |
| **ACL-03** Temporal BindMarinePolicy + NATS events | #25 · `ktayl-integration` | the async bind choreography |
| **UW-01** (existing) workbench thin slice | Underwriting #12 · `ktayl-underwriting` | bind one Marine policy via the ACL; subscribe to `policy.issued` |
| **ING-01** submission-pack intake | DMS #24 · (ingestion svc) | email/PDF/Excel → Docling/markitdown |
| **ING-02** LLM extraction (limited-risk) | AI #4 | extraction + Presidio + Langfuse + **AI System Card** |
| **ING-03** pre-fill legacy intake via ACL | #24/#25 | structured submission → ACL → legacy, human-verify |

## 7. Delivery path & governance
**Path C** (new services, cross-boundary integration, an AI use case). Named gates apply, *practice-sized*:
- **Architecture review** — the ACL contract + the Temporal workflow (this doc is the spine).
- **Security review** — the AI ingestion (PII/AI-Act) + the legacy auth boundary + egress.
- Per-repo BMAD: author epics/stories in each home repo → sync to its board.

## 8. Scope guardrails (keep it a practice, not a swamp)
- **One LOB (Marine), one flow** (submission → bind, + renewal). Defer claims, endorsements, other LOBs.
- **Fake constraints, not COBOL** — a modern service that *behaves* legacy (SOAP/flat-file/batch/codes).
- **DoD for the spine (UC-1):** an underwriter binds one Marine policy in the workbench → ACL → Temporal
  → nightly batch issues it in the legacy core → `policy.issued` notifies the workbench. Then add UC-2.
- **Non-goals:** real COBOL/AS/400; multi-LOB; production data; replacing `ktayl-policy-service`.

## 9. What you practice (the payoff)
Strangler-fig · anti-corruption layer · **SOAP/legacy + fixed-width/batch integration** · event-driven
wrapping (NATS) · **durable async orchestration / sync-over-async avoidance (Temporal)** · IDP/document-AI
· **AI-Act limited-risk controls by design**. This mirrors the real HDI hub-and-spoke and the apprenticeship
integration missions — high transfer value.

## References
- Underwriting FDE playbook (UW domain + §7a extraction + §12 regulatory): `ktayl-underwriting/docs/`
- AI-Act gate: `minicloud-platform-docs` → `ai-ml/ai-act-gate`
- Modernization/Strangler-Fig + ACL principle: `minicloud-gitops/.claude/rules` (modernization)
- GAP wrapper-chart deploy standard: `.claude/rules/gitops.md`

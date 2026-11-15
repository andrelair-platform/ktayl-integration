# ADR-0001 — API management: compose from the platform, gate the external gateway on need

- **Status:** Accepted (2026-11-15)
- **Owner:** AndreLiar (SA/TL)
- **Context gate:** `cloud-adoption.md` (need-first adopt-vs-build), `platform-vs-information-system.md`
  (provider↔consumer), `conventions.md` (`api/openapi.yaml` contract rule). This is the founding
  architecture decision for the `ktayl-integration` domain (#25) and frames epic **INT-01 — API gateway
  & broker APIs**.

## Context

The question raised: *"put in place API management — choose the best solution; does Backstage already
handle it?"*

"API management" is not one capability — it is routing + TLS, authN/Z, rate-limiting/quotas,
key/credential management, request validation/transformation, an API catalog/developer portal, and
per-consumer usage analytics. On this platform those capabilities are **already owned by existing
components**, and the need splits cleanly into two scopes that must not be conflated:

- **Internal east-west (service→service).** Already managed: **Authentik** (OIDC + M2M
  client-credentials + forward-auth), **Cilium/Hubble** (default-deny netpols + L7 observability),
  **Backstage** (service + API catalog), the **`api/openapi.yaml`** contract convention, and **L3
  contract tests** in CI. Nothing is missing here.
- **External north-south (broker / reinsurer / EDI / partner APIs).** This is the only real gap, and it
  is exactly what `ktayl-integration` INT-01 is chartered to build. Partners need API keys/quotas,
  rate-limits, request validation, and a partner-facing surface that nginx + Authentik do not provide
  cleanly today.

**Backstage verdict (checked, not assumed):** Backstage covers the **catalog / developer-portal half**
of API management and **none** of the runtime/gateway half. It *is* configured for it —
`helm-values/minicloud-1/backstage-values.yaml` allows `kind: API` entities and the custom image ships
the API Docs (Swagger/OpenAPI) plugin — but it is **underused**: existing OpenAPI specs
(`ktayl-core/api/openapi.yaml`, `ktayl-policy-service/api/openapi.yaml`) are **not registered as API
entities**. Backstage will never do routing, rate-limiting, key issuance, transformation, or traffic
policy — that is a dataplane's job, not a portal's.

**No gateway / APIM product exists in the cluster today** (no Kong/APISIX/Gravitee/Tyk, no Gateway API).

## Decision

**Do not adopt a monolithic API-management product. Assemble API management from the platform, and add a
single lightweight gateway at the external edge only — gated behind a real external consumer.**

Concrete sequence:

1. **Now (free, no new infra):** register the existing OpenAPI specs as Backstage `kind: API` entities
   and wire `providesApis` / `consumesApis` on the owning Components. Backstage becomes the real internal
   API catalog. (BC02 evidence — API contracts + provider/consumer graph.)
2. **At the INT-01 trigger** (a real or imminent broker/reinsurer/EDI consumer): deploy **Kong Gateway
   OSS in DB-less / decK mode, ArgoCD-managed**, at the **external edge only**. Expose one real internal
   API (e.g. `ktayl-policy-service`) behind **key-auth + rate-limit + Authentik OIDC validation**; prove
   the flow with a **synthetic broker client** (no fabricated live partner — honours the need-first
   gate). Capture the gateway product choice in a follow-up ADR (Kong vs APISIX/Gravitee/Tyk).
3. **Invariant:** **never** place a gateway in front of internal east-west traffic — Authentik M2M +
   Cilium already manage it and a gateway would only add a hop and duplicate identity.

Division of roles stays: **Authentik = IdP**, **Backstage = API catalog / internal portal**,
**Kong = external-only dataplane**.

## Why (vs the alternatives)

- **Need-first (`cloud-adoption.md`).** Internal API management already exists; buying an APIM for it
  solves nothing. The external need is real but has **no live consumer yet** — so the gateway is designed
  and trigger-gated, not stood up speculatively.
- **A monolithic enterprise APIM was rejected** (Gravitee full / WSO2 / Kong Enterprise): Elasticsearch +
  MongoDB footprints are disproportionate on a resource-constrained k3s cluster, and they **duplicate
  Authentik (identity) and Backstage (catalog)** — buying a second IdP and a second portal to get a
  gateway is the wrong trade.
- **Kong OSS DB-less over APISIX/Tyk/Gravitee** (provisional, to be confirmed in the INT-01 ADR):
  declarative YAML config is **GitOps-native** (ArgoCD-friendly, no external DB/etcd/Redis to operate),
  it has the needed plugins (key-auth, rate-limiting, OIDC token validation against Authentik), and it
  speaks Gateway API via KIC. Lightest fit for the existing GitOps + Cilium + Authentik stack.
- **Building against a fabricated partner was rejected** — it violates need-first and reads as a demo
  prop. Routing a **real** internal API to a **synthetic** consumer is the honest, defensible slice.
- **A standalone 4-way product bake-off doc was rejected** as a deliverable-for-its-own-sake at
  zero partners; the comparison folds into the INT-01 ADR when the gateway is actually built.

## Consequences

**Positive:** zero new infra to get the immediate win (step 1); no footprint regret on k3s; identity and
catalog stay single-sourced (Authentik, Backstage); the external gateway, when it lands, is scoped to the
edge and GitOps-managed like everything else; clean cert evidence (BC02 contracts now, BC02/BC03 gateway
+ security later).

**Negative / accepted:** external API management is **deferred** until INT-01's trigger — an external
partner integration cannot ship instantly, it needs the gateway built first (accepted; there is no
partner today). Rate-limiting for the handful of current nip.io/tunnel-exposed apps remains crude
(nginx annotations) until the gateway exists — acceptable, as none are partner-facing.

**Revisit trigger:** the first real broker/reinsurer/EDI consumer, or a partner contract requiring
metered/keyed API access → execute step 2 and write the gateway-product ADR.

## Current state — the interim API management in place (as-built, 2026-11-15)

Step 1 is **done and verified live**. What runs today (the "compose from the platform" model):

| Capability | Provided by | Status |
|---|---|---|
| **API catalog / developer portal** — discovery, ownership, OpenAPI/Swagger rendering, provider↔consumer graph | **Backstage** | ✅ live — API entities registered + **served** (verified in `backstage_plugin_catalog.final_entities`): custom = `ktayl-core-billing-api`, `minicloud-plane-api`, `litellm-api`, `erpnext-rest-api`, `platform-demo-api`, `markitdown-proxy-api`, `rag-ingest-api`; vendor = `authentik-api`, `harbor-api`, `nextcloud-api`, `n8n-api`. Each has a provider Component + `providesApi`/`apiProvidedBy` relation + rendered spec + a link to the authoritative docs. |
| **AuthN / AuthZ** — SSO, service-to-service | **Authentik** (OIDC + M2M client-credentials) | ✅ already there |
| **Network / L7 policy** | **Cilium + Hubble** (default-deny, per-service) | ✅ already there |
| **API contracts** | committed `api/openapi.yaml` + **L3 contract tests** | ✅ already there |

**Scope (be precise — do not overclaim):** this is the **catalog + identity + policy** half of API
management, for **internal (east-west)** APIs. It explicitly does **NOT** include the runtime gateway
half — **no per-consumer API keys, no rate-limiting, no quotas, no request transformation, no per-API
usage analytics.** Backstage is a *portal*, not a *dataplane*. The honest claim is: *"a working interim
API catalog + governance layer; the dedicated external gateway is designed and gated behind the first
external consumer."*

## When to switch to "real" API management (the gateway tier) — the trigger checklist

Stand up the Kong edge gateway (step 2) when **ANY** of these becomes true. The common thread:
**an API consumer outside the trust boundary, or money/metering riding on an API.**

1. **First external / third-party consumer** (THE primary trigger) — a broker, reinsurer, EDI/coverholder,
   or any client we don't control calls an API. Internal east-west never needs it; the edge does. (= INT-01.)
2. **Per-consumer API keys** — a partner expects an API key to issue/rotate/revoke, not an Authentik
   OIDC client-credentials flow.
3. **Rate-limiting / quotas to protect a backend** — abuse protection or plan/quota enforcement on a
   partner- or public-facing API (nginx annotations are too crude).
4. **Per-consumer / per-API usage analytics** — billing, SLA reporting, "who called what, how often".
5. **Monetization / productization** — a brick (e.g. the DORA vendor-risk SaaS) exposes a **metered** API
   to paying customers → keys + plans + metering = APIM (`productization.md`).
6. **Edge mediation / transformation** — protocol/format translation or a versioning facade for an
   external consumer (EDI, legacy partner formats).
7. **A regulatory / audit requirement** for the external API surface — DORA/ACPR requiring documented,
   controlled, per-consumer-audited external API access.

**NON-triggers — stay on the interim model, do NOT build a gateway for these:** more internal services
calling each other (Authentik M2M + Cilium handle it) · an internal UI/BFF consuming an internal API ·
wanting a nicer catalog (Backstage already does it) · adding more internal OpenAPI specs to the catalog.

**The one-line test:** *does an API now face a party I don't control, or does money/metering ride on it?*
→ **yes** = build the gateway; **no** = stay interim.

When a trigger fires: execute step 2 (**Kong OSS DB-less** at the edge — keys + rate-limit + Authentik
OIDC validation), keep Authentik as IdP + Backstage as catalog, and write the gateway-product ADR (the
Kong-vs-APISIX/Gravitee/Tyk comparison).

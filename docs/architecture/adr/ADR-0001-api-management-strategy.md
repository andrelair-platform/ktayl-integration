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

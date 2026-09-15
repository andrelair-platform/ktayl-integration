# AGENTS.md — ktayl-integration

Tiny repo-specific context. Org rules (`minicloud-gitops/.claude/rules/*`) still apply.

## Policy
- **ktayl-solution IS** product (IS Foundations, board **#25**) — Integration Platform. **NEVER involve Retrieva**
  (separate product / the RNCP certification).
- This repo is the **product home** (image + app config + docs); deploy wiring (chart/values/manifests/
  apps) lives in `minicloud-gitops`. Env-agnostic image, CODEOWNERS-gated prod, Kargo dev→prod.
- Custom apps ship as **GAP wrapper Helm charts** at `services/<svc>/helm/` in `minicloud-gitops`
  (not raw manifests, not a stock vendor chart) — see the gitops rules.
- Do **not** name the real reference insurer anywhere in docs/code.

## Status
Scaffold. BMAD per-product. Build not started — decompose the epics into stories first.

# Flex AI Platform: Architecture Summary

Covers four sibling repos: **token-service**, **fleet-manager**, **fcs**, **infra**.

> Source: parallel read-only analysis of each repo (docs, entry points, grep for cross-repo references). Not every doc was read in depth, and runtime behaviour was not verified. Treat dated migration plans and runbooks as decision records, not current state.

---

## 1. One-paragraph model

- **token-service** is the customer-facing inference API (`api.flex.ai/v1`, `tokens.flex.ai`) and the billing/identity back-office. It sells tokens but does not run GPUs.
- **fleet-manager** is the admin control plane for the GPU fleet and model serving. It decides which GPUs run which models and tells token-service what is serving and what may be sold.
- **fcs** ("fcsv1") is the training and dedicated-inference PaaS: console, CLI, a Go system of record and an in-cluster operator.
- **infra** is the GitOps/IaC monorepo that deploys the other three. The app repos publish charts and images to GAR; infra pins versions and Flux applies them.

The same physical GPU clusters (e.g. smc-001) serve both token-service inference and fcs training. fleet-manager's `tier-controller` arbitrates GPU ownership.

## 2. System diagram

```
                  ┌──────────── infra (GitOps / Pulumi / Ansible) ────────────┐
                  │ clusters.yaml + environments.yaml → generated k8s/<cluster>│
                  │ Flux HelmReleases pin charts from GAR; OpenBao; Skupper    │
                  └──────┬──────────────────┬──────────────────┬──────────────┘
                 deploys │                  │                  │
                         ▼                  ▼                  ▼
 customers ─► token-service ◄── feeds ── fleet-manager ──HTTP/Skupper──► cluster-agent (per GPU cluster)
 api.flex.ai  Envoy → LiteLLM/flexgate   /api/serving/live               tier-controller (GPU owner)
              portal-api (FastAPI)       /api/catalog/models                      ▲
              global/regional Postgres        ▲                                   │ /allocate /heartbeat /release
                   │ ▲                        │ reads prices from                 │
  identity/org SSOT│ │ managed proxy          │ flexaihq/artifacts           fcs flex-agent
  + gpu-pricing    ▼ │ + usage windows                                             ▲
                  fcs: platform BFF ─► experience (Go) ──Temporal / gRPC over Skupper┘
                  console, CLI, Lago billing
```

## 3. Repo summaries

### 3.1 token-service

**Role:** OpenAI-compatible inference gateway, per-user/org virtual keys, rate limits, budgets, metering, Stripe/wallet/credit-ledger billing, portal SPA and customer docs source.

**Components (namespace `token-service`)**
- LiteLLM proxy with custom hooks (`backend/litellm_hooks/*`): admission gate, usage guard, rpm gate, request validator, activation gate and others.
- `flexgate/`: a Go data plane being introduced beside LiteLLM (Redis Lua, Pub/Sub usage logs).
- Portal API (`backend/`, FastAPI). One image; `APP_ROLE` selects `control`, `media`, `worker` (all billing/metering loops) or `usage-ingest`.
- Portal SPA (`portal/`), `statuspage-sync/` CronJob, `runtimes/*` (TTS/OCR images), `flexserve-patch/`.

**Request path:** Envoy Gateway → `/v1/*` to LiteLLM (carve-outs to portal-api for `/v1/images|audio|videos|models|pricing`), `/api/*` to portal-api, `/*` to the SPA → LiteLLM → vLLM/SGLang/KServe pods on GPU clusters over Skupper listeners (`k8s/skupper-hub/`).

**Data:** Postgres via raw asyncpg (all SQL in `backend/database.py`), no ORM. A `global` schema (orgs, billing, ledger, memberships) is split from regional tables (keys, usage); no SQL may join across them (`make db-guard`). Redis for flexgate/rate limits. Numbered migrations in `backend/migrations/` and `backend/migrations-regional/`.

**Catalog:** `model_display`, `model_route`, `model_pricing`. `backend/fleet_publish.py` is the only writer, sweeping every ~5 minutes. Brakes: observe-only, per-sweep caps, "no signal means no write". `fleet_liveness.py` decides what `/v1/models` lists.

**Auth:** Ory OIDC with BFF cookie session. token-service is the SSOT for users and orgs.

**Deploy:** GKE + Flux. Dev floats on main; staging is pinned per RC; prod is promoted by a human-reviewed infra PR bumping `chartVersion` in `k8s/clusters.yaml`.

**Read first:** `docs/end-to-end-architecture.md` (current; `docs/architecture.md` is stale), `docs/fleet-publish-sweep.md`, `backend/fleet_publish.py`.

### 3.2 fleet-manager

**Role:** Teleport-gated admin control plane (Clusters, Models, Catalog tabs). Owns serving placement, perf scorecards and Catalog operator facts. `flexaihq/artifacts` owns prices, context, quant and legal facts; the contract is `ai-specs/contracts/artifacts-fleet-contract.toml` (locked, v1.1.0).

**Components**
- Hub: one FastAPI + React image (`backend/main.py`, `portal/`) with its own Postgres.
- Per GPU cluster: `cluster-agent` (inventory, scaling, stage jobs, weight purge) and `tier-controller` (admission webhook on KServe pods, 60s decision loop, defrag, external lease API).
- Also: `snapshot-plane` (GPU snapshot/restore), `tt-device-plugin` and `tt-exporter` (Tenstorrent), `serving-images/*`, `audex-serve`, `parakeet-serve`, `wan-serve`, `kv-cache-sim`.

**Hub ↔ cluster:** HTTP with a bearer over Skupper (`backend/cluster_agent_client.py`, `fleet_cache.py`; roster via `CLUSTER_AGENT_<NAME>_URL`). Mutations write an audit row in the same transaction.

**Publishing pipeline:** artifacts row → fleet-manager Catalog → cluster-agent creates InferenceService/Connector → hub Skupper Listener (infra GitOps PR) → token-service sweep writes catalog and LiteLLM routes.

**Tiering:** `desired_hot = clamp(demand, max(floor,1), ceiling)`. Only cold park remains (drain, delete, restore from snapshot or cold boot); warm/hibernate parking was retired 2026-09-05.

**Read first:** `docs/model-publishing.md`, `backend/main.py`, `tier-controller/src/tier_controller/webhook.py`.

### 3.3 fcs

**Role:** `flexai training run` / `flexai inference serve`, console at console.flex.ai, CLI.

**Layers**
- `platform/`: Python 3.12 FastAPI BFF (stateless), React 19 console, Go CLI, generated clients (do not hand-edit).
- `experience/`: Go system of record (Gin, GORM, Atlas) owning Postgres; commands `backend`, `worker` (Temporal), `dash`.
- `experience/flex-agent`: in-cluster operator. The backend pushes desired state over a clusterlink gRPC stream tunneled by Skupper; flex-agent creates `compute.flex.ai` Training/Inference CRs, and Flux renders the `flexai-training` chart.
- Billing: batch job (every 10 min) reads the experience DB and pushes GPU-hours × rate to Lago.

**GPU leasing:** flex-agent calls tier-controller `POST /allocate`, then pins workloads with `NVIDIA_VISIBLE_DEVICES=GPU-<uuid>` (no device-plugin fallback). Leases are in-memory in tier-controller, so a 404 on heartbeat triggers re-allocation.

**Read first:** `docs/architecture.md`, `docs/workload-runbook.md`, `experience/flex-agent/internal/controller/`.

### 3.4 infra

**Role:** desired state of every cluster (`k8s/`), cloud/secrets/DNS provisioning (`pulumi/`), host and cluster bring-up (`ansible/`, workflows).

**Key pieces**
- `k8s/clusters.yaml` (~16k lines) + `k8s/environments.yaml` → rendered by `make generate-resources` into `k8s/<cluster>/`. **Never hand-edit generated dirs.**
- Pulumi (Python), state in a GCS bucket with KMS. Projects: vault-v2, secrets-sync, teleport, gcp/*, cloudflare_*, grafana-*, byoc and more.
- `ansible/` for host config and k3s; `byoc-tool/` for customer BYOC clusters on AWS/Azure.
- OpenBao (Vault fork) for secrets, consumed through vault-secrets-operator. About 70 GitHub workflows for onboarding, Pulumi, e2e and chart publishing.
- Tiers: INFRA-DEV → INTERNAL-STAGING → INTERNAL-PROD → CLIENT-PROD.

**Topology**
- GKE control planes in project `fcs-production-cluster`: backoffice (OpenBao, Teleport, fleet-manager, staging token-service) and prod token-service (US/EU).
- GPU/edge clusters on k3s: smc-001, mi300x, jarvis, Tenstorrent, arm64, BYOC templates.
- fcs's control plane is being folded into the prod token-service cluster (plan 2026-07-28). Scaleway is being decommissioned.

**Read first:** `AGENTS.md`, `k8s/clusters.yaml`, `scripts/src/flexai/`, relevant `runbook-*.md`.

## 4. Cross-repo connections

| From → To | Mechanism | Key files |
|---|---|---|
| token-service → fleet-manager | polls `GET /api/serving/live` and `GET /api/catalog/models`, shared bearer over Skupper | token-service `backend/fleet_publish.py`, `fleet_liveness.py`; fleet-manager `backend/serving_registry_api.py`, `catalog_api.py` |
| fleet-manager → cluster-agent / tier-controller | HTTP over Skupper | `backend/cluster_agent_client.py` |
| fcs flex-agent → tier-controller (fleet-manager) | `POST /allocate`, `/heartbeat/{lease}`, `/release/{lease}`; sends `display_name`, `org_name` so the cluster page shows names | fcs `experience/flex-agent/internal/tierclient/`; fleet-manager `tier-controller/` |
| fcs → token-service | internal HTTP + bearer: user/org SSOT, `gpu-pricing/lookup`; billing-block verdict back via `PUT /admin/organization/{id}/billing-block` | fcs `experience/backend/internal/tokenservice/client.go` |
| token-service → fcs | `/api/managed/v1/*` proxy injecting `X-Ory-User-Id`; pulls `/admin/usage/windows` | token-service `backend/managed_proxy.py`, `managed_usage_client.py` |
| token-service portal ← fcs UI | fcs console frontend vendored into the portal (hash-checked) | token-service `portal/src/managed-console/` |
| fleet-manager → infra | dispatches infra workflows (cluster deploy, onboarding, Skupper listener PRs) | fleet-manager `backend/github_client.py` |
| infra → fleet-manager | cluster-enrol workflow dispatches fleet-manager to edit the `clusterAgents.listeners` roster | infra `.github/workflows/setup_k3s_training_cluster.yml` |
| infra ↔ token-service | pin workflows edit `clusters.yaml`; merged pins dispatch release events back | infra `scripts/set_token_service_pin.py`, `token-service-*.yml` |
| infra → all three | HelmRepository + HelmRelease per chart, pinned versions | infra `k8s/k8s-templates/helm-releases/`, `k8s/environments.yaml` |

Fleet-manager has no code dependency on fcs; the only link is the lease-metadata fields above.

## 5. Shared infrastructure

- **Skupper:** cross-cluster plumbing. Hub listeners live on the GKE token-service clusters (infra-owned via Flux); serving clusters hold connectors. Listener port must equal Connector port.
- **GAR:** all app charts and images publish to `us-west2-docker.pkg.dev/fcs-production-cluster/*`.
- **Ory:** auth.flex.ai points at token-service's Ory project.
- **Teleport:** fronts fleet-manager.
- **OpenBao:** secrets for all clusters, per-cluster JWT auth mounts.

## 6. Key flows

1. **Publish a model:** artifacts row → fleet-manager Catalog → cluster-agent IS/Connector → hub Listener (infra PR) → token-service sweep (~5 min) → `/v1/models` and LiteLLM route.
2. **Inference request:** client → Envoy → LiteLLM/flexgate (key, rate, budget, activation gates) → Skupper → vLLM/SGLang pod → usage metering → billing.
3. **Training job:** console/CLI → platform BFF → experience → Temporal → clusterlink gRPC → flex-agent → tier-controller lease → Training CR → Flux/chart → pod pinned to leased GPUs → usage → Lago.
4. **Deploy:** repo publishes chart to GAR → bot PR to infra bumps pin → merge → Flux applies (prod via human-reviewed PR).
5. **Identity:** token-service is SSOT; fcs resolves Ory sub through it (60s cache, fails open to local mirror, dormant unless URL and token env vars are set).

## 7. Gotchas

- Never hand-edit infra's generated `k8s/<cluster>/` dirs or `kubectl edit` live resources; Flux/VSO overwrites them.
- Adding a model is not a token-service change. Do it through artifacts and the fleet-manager Catalog; do not hand-UPDATE catalog columns the sweep re-asserts.
- Any `/v1` route carve-out needs a change in both token-service (chart) and infra (edge Gateway).
- Prod fleet feed flags (`SERVING_FEED_PER_ARM`, etc.) are ordered against token-service deploys; check before flipping.
- Teleport JWT audience for fleet-manager must be the internal Service URI, or it looks like a missing role.
- Publishing a fleet-manager chart rolls the whole fleet; `make deploy-agent` / `deploy-tier-controller` are break-glass only (suspend the HelmRelease first).
- tier-controller leases are in-memory; a restart drops them and flex-agent re-allocates.
- fcs: a 403 in Vault namespace `admin` means a missing mount; `enqueued` does not mean waiting for GPUs; never edit generated clients.
- Never run infra `secrets-sync` without the `VAULT_ADMIN_INFRA_DEV_*` credentials; it deletes fcsv1 training auth.
- Documentation drift: fcs CLAUDE.md claims `token-service/` is in its monorepo (it is not); token-service CLAUDE.md references a `cluster-agent/` dir that does not exist there (it lives in fleet-manager); token-service `docs/architecture.md` is a stale 2026-04 snapshot.

## 8. Where to start reading

| Goal | Start at |
|---|---|
| Whole-system picture | token-service `docs/end-to-end-architecture.md` |
| Model publishing | fleet-manager `docs/model-publishing.md` |
| GPU scheduling | fleet-manager `tier-controller/docs/autoscaling-state-machine.md`, `external-allocation-api.md` |
| Training workflow | fcs `docs/architecture.md`, `docs/workload-runbook.md` |
| Identity/org SSOT | fcs `docs/auth-ssot-consolidation-plan.md`, `docs/eng-1492-handoff/` |
| Deploy and release | infra `AGENTS.md`, token-service `docs/release-process.md`, fcs `docs/release-and-deploy.md` |
| Incidents and history | infra `runbook-*.md` and `migration-plan-*.md` |

# Flexgate Design Risks

This note summarizes the main risks in the flexgate proposal in plain language. It focuses on the areas most likely to affect customer traffic, billing correctness, operations, and rollout safety.

## Executive Summary

The performance case for flexgate is strong: the current LiteLLM path is expensive, slow to scale, and fragile under high streaming load. The main risk is not whether a Go data plane can be faster. The main risk is that the proposal changes too many critical systems at once:

- request routing
- billing event generation
- balance enforcement
- usage durability
- rollout fences
- dashboards and admin readers
- Skupper transport topology
- operational ownership

This makes the design harder to verify, debug, and roll back.

## Why the Flexgate Go Plane Helps

Flexgate is helpful because it moves the performance-sensitive request path out of LiteLLM's Python proxy and into a smaller Go data plane built for streaming inference traffic.

### 1. Lower CPU per request

LiteLLM does a lot of expensive work on the hot path: Python event-loop work, pydantic object rebuilds, per-chunk parsing, re-serialization, hooks, SDK overhead, and database-adjacent logic. The doc's measurements show flexgate using much less CPU per request than LiteLLM in the same-box tests.

Why this helps:

- More requests can be served with fewer cores.
- The system has more headroom during spikes.
- Rejections and overload handling can be cheaper.

### 2. Better fit for streaming

Inference responses are long-lived streaming requests. A thin Go relay can keep per-chunk work small and predictable.

Why this helps:

- Less work per SSE chunk.
- Fewer allocations and less object churn.
- Better behavior under many concurrent streams.
- Cleaner backpressure and admission control.

### 3. Faster startup and scaling

LiteLLM pods are slow to become ready because they carry a heavier Python application and startup path. Flexgate is designed to boot quickly and read control-plane state from snapshots.

Why this helps:

- Faster scale-out during traffic spikes.
- Lower need for a large always-on LiteLLM floor.
- Easier use of autoscaling once admission metrics are trusted.

### 4. Explicit overload behavior

The current path can turn saturation into 504s or hidden edge-level failures. Flexgate is designed to reject overload quickly with 429s and `Retry-After`, while preserving 5xx for actual failures.

Why this helps:

- Clients get clearer retry signals.
- Overload is cheaper for the system.
- Operators can distinguish capacity pressure from broken infrastructure.

### 5. Cleaner separation of data plane and control plane

The architecture moves fast request handling into flexgate and slower-changing policy/configuration into snapshots and changelogs.

Why this helps:

- The hot path avoids repeated database calls.
- Key/model/org/routing state can be refreshed outside the request path.
- Both LiteLLM and flexgate can eventually consume the same projected control-plane state.

### 6. More durable usage recording

Flexgate explicitly records usage before sending `[DONE]`, instead of relying on LiteLLM's delayed SpendLogs batch flush.

Why this helps:

- Fewer lost usage records when pods die.
- Better audit trail for finished, failed, and cancelled requests.
- Cleaner path to request-time frozen pricing.

### 7. Safer incremental migration

The proposed `off` / `ready` / `on` modes and per-org/model fences let flexgate be provisioned before it carries traffic.

Why this helps:

- Infrastructure can be deployed dark.
- Traffic can move gradually.
- Rollback can be a routing/fence change rather than a full redeploy, assuming billing and usage state are handled correctly.

### 8. Architectural focus

LiteLLM remains useful for compatibility, admin paths, and fallback. Flexgate focuses on the narrow high-volume path: accepting a request, enforcing lightweight controls, relaying the stream, and producing usage.

Why this helps:

- The busiest path becomes smaller and easier to reason about.
- The system can optimize the 99% path without carrying all LiteLLM complexity.
- Future scaling work can target a purpose-built component rather than a general Python proxy.

The core value is therefore sound: flexgate can make the inference gateway cheaper, faster to scale, and more predictable under streaming load. The risks below are mostly about whether the first release is scoped tightly enough and whether billing/control-plane correctness is proven before customer traffic moves.

## Highest-Risk Areas

### 1. Billing Correctness

Flexgate introduces a new usage-event path that is meant to replace or run beside LiteLLM SpendLogs.

Risks:

- Some traffic may be billed through new flexgate usage events while existing dashboards and admin tools still read LiteLLM SpendLogs.
- Cancel and disconnect billing semantics changed late in the design. Older code, tests, or docs may still assume "delivered tokens only."
- Different engine arms have different capabilities. A cancel may bill using engine abort counts on one arm and delivered estimates on another.
- Void/refund behavior is complex. A chargeable event may later need to become non-chargeable if the client did not actually receive the response.
- Aggregate billing is scalable, but it can make per-request disputes harder unless request-level audit records remain easy to query.

Why it matters:

Billing bugs become customer trust, finance, and audit problems. They are harder to fix after traffic has already moved.

Recommended gates:

- Two-human review of billing, finalize, usage ingest, void, and reconciliation code.
- End-to-end tests for success, engine fault, gateway fault, client disconnect, cancel, retry, duplicate event, and replay.
- Confirm all customer/admin usage readers handle flexgate traffic before any billed org moves.

## 2. Control Plane and Balance Model

The control plane supplies snapshots, changelogs, fences, capability flags, balance state, and rollout decisions.

Risks:

- The balance model is still an open decision: leases versus a simpler snapshot/counter model.
- The lease/re-anchor/reclaim/reaper design is effectively a distributed accounting system.
- Redis-gw becomes safety-critical for fences and balance. A redis-gw outage could block traffic or create inconsistent admits.
- Stale snapshots can cause wrong routing, wrong billing treatment, wrong access decisions, or wrong model availability.
- Revocation and re-enable flows are delicate. A stale revocation overlay could keep a re-enabled key dead for too long.

Why it matters:

The control plane decides who can send traffic, where traffic goes, and whether money can be spent. Bugs here can cause outages or incorrect charging.

Recommended gates:

- Decide and document the balance model before prod.
- Test stale snapshot, missed changelog, redis-gw outage, rollback, and key revocation/re-enable cases.
- Prefer a simpler phase 1 balance model if the first target is an exempt OpenRouter-style org.

## 3. Dual-Plane Rollout Risk

During migration, LiteLLM and flexgate may both exist on the request path.

Risks:

- The per-org/model fence must keep the two planes from admitting the same traffic incorrectly.
- LiteLLM grace periods, flexgate fence cache TTLs, and settle-on-release logic must all line up.
- Rollback is described as a fence flip or `mode: ready`, but in-flight usage, unsettled balance, and replayed events may make rollback messier.
- Enabling the fence on LiteLLM can introduce a new dependency on redis-gw before flexgate carries traffic.

Why it matters:

A migration should not make the incumbent LiteLLM path less reliable before the new path is proven.

Recommended gates:

- Run full dual-plane tests with traffic moving both directions.
- Test redis-gw failure while LiteLLM is still serving.
- Prove rollback with in-flight streams and unsettled usage.

## 4. Database and Schema Changes

The design mentions additive schema changes, especially migrations 414 and 415, plus new usage/billing schema from the usage pipeline.

Risks:

- "Off mode" is not truly byte-identical if schema changes still run.
- Migration 414 depends on the `flexgate_usage` Postgres role existing first.
- If the role/runbook step is missed, grants may not apply as expected.
- Existing SpendLogs readers may not see flexgate-served traffic.
- The billing ledger currently lacks a dedicated usage void/reversal type, so voids may appear as generic adjustments.

Why it matters:

Additive schema is safer than destructive schema, but it can still create broken readers, missing permissions, or audit ambiguity.

Recommended gates:

- List the exact new tables, roles, grants, indexes, and readers before staging.
- Confirm every environment has the required role before migration.
- Add explicit tests that dashboards, exports, admin views, request lookup, and billing reconciliation see flexgate traffic.

## 5. Usage Durability and Replay

Flexgate must durably record usage before sending `[DONE]`.

Risks:

- Pub/Sub, Postgres fallback, disk spool, replay jobs, format versions, and gap detection form a complex finalize chain.
- Pod death during finalize may produce duplicate, missing, or delayed usage events.
- Spool format changes require careful drain/upgrade behavior.
- Per-pod persistent disk spool adds operational complexity.

Why it matters:

This path sits directly between customer response completion and billing correctness.

Recommended gates:

- Chaos test pod death before publish, after publish, before `[DONE]`, and during replay.
- Prove idempotency for duplicate events.
- Prove safe replay across version changes.
- Consider simplifying the first release to Pub/Sub plus a smaller local fallback if acceptable.

## 6. Observability and Customer Support Gaps

Existing alerts and dashboards are built around LiteLLM metrics and SpendLogs.

Risks:

- Flexgate traffic may disappear from current customer usage pages.
- Admin request lookup may not find flexgate requests.
- Error-rate alerts based on LiteLLM metrics may go blind for moved traffic.
- Support may not know which request ID customers should quote.

Why it matters:

If a customer reports a billing or latency issue, support must be able to trace the request across edge, flexgate, engine, usage event, and ledger.

Recommended gates:

- One canonical customer-facing request ID.
- Request lookup works for both LiteLLM and flexgate.
- Alerts cover both planes before production traffic moves.
- Dashboards clearly distinguish old and new usage sources where needed.

## 7. Skupper and Transport Uncertainty

The design reduces LiteLLM cost, but Skupper may become the next bottleneck.

Risks:

- The single-router ceiling is still not fully proven.
- Real 10x/100x load through Skupper is not yet verified.
- Sharding Skupper before measuring could overbuild expensive infrastructure.
- Running heavy load tests through shared Skupper infrastructure could create an incident.

Why it matters:

If Skupper becomes the limiting factor, flexgate may be fast but the whole path still bottlenecks elsewhere.

Recommended gates:

- Measure Skupper versus an alternative tunnel path before major infra commitments.
- Use dedicated load-test infrastructure, not shared production routers.
- Derive shard count from measured router ceiling and real demand.

## 8. Capacity and Demand Assumptions

The proposal often discusses 100x current peak.

Risks:

- The 100x target may not match actual OpenRouter demand.
- Infrastructure may be sized for hypothetical demand rather than advertised/contracted demand.
- The design may reserve quota and capacity before the demand curve is agreed.

Why it matters:

Overbuilding increases cost and complexity. Under-measuring increases outage risk.

Recommended gates:

- Get a 30/60/90-day demand curve from OpenRouter owners.
- Size the first release to admitted engine capacity plus headroom, not hypothetical request volume.
- Treat excess demand as a controlled 429 budget.

## 9. Per-Tenant Fairness

Admission control is mostly per-pod and per-arm.

Risks:

- A high-volume aggregator or exempt org can consume all active stream capacity.
- Direct paying customers may receive 429s while aggregator traffic continues.
- OpenRouter exemption from prepaid balance could bypass one form of protection unless paired with explicit org caps.

Why it matters:

Protecting one large partner should not starve normal customers.

Recommended gates:

- Add per-org or per-tenant active request limits.
- Carry per-org caps in the control-plane snapshot.
- Set aggregator caps to advertised capacity plus agreed headroom.

## 10. Behavior Parity and Missing Features

Flexgate aims for parity with LiteLLM, with documented exceptions.

Risks:

- Exact parity increases implementation complexity.
- Some current behavior may not be ported, such as tool-choice enforcement or TPM limits.
- Some LiteLLM quirks may not be worth preserving.
- A byte-level parity bar may slow delivery and keep bad behavior alive.

Why it matters:

Customers care about stable behavior, but not every LiteLLM quirk is a product contract.

Recommended gates:

- Decide the parity bar explicitly.
- Separate "must preserve" API contracts from "LiteLLM implementation quirks."
- Add tests for product-visible behavior, not only byte-for-byte output.

## 11. Operational Ownership

Flexgate introduces a Go service into a path currently dominated by Python/LiteLLM.

Risks:

- The team needs Go production debugging skills.
- On-call must handle goroutine leaks, GC stalls, h2/h2c issues, replay bugs, and admission bugs.
- Runbooks may lag the new system.

Why it matters:

A faster system is not safer unless the on-call team can debug it at 3 a.m.

Recommended gates:

- Name Go on-call owners.
- Create runbooks for overload, replay backlog, Pub/Sub outage, redis-gw outage, stuck fence, and billing reconciliation.
- Add dashboards for Go runtime, queueing, admission, finalize, replay, and upstream transport.

## 12. Security and Secrets

The design introduces or depends on several sensitive credentials and tokens.

Risks:

- Abort tokens span multiple environments/clusters.
- redis-gw credentials are shared by multiple components.
- The doc mentions exposed OpenBao/dev secrets that still need rotation.
- Static DB logins for `flexgate_usage` were chosen over Vault-issued ones.

Why it matters:

Gateway credentials sit on a high-value request and billing path.

Recommended gates:

- Rotate exposed tokens before production.
- Document credential ownership and rotation procedures.
- Confirm least-privilege grants for `flexgate_usage`.
- Audit which services can read/write redis-gw state.

## Suggested Safer Phase 1

A lower-risk first release would be:

- OpenRouter-focused only.
- Metered, but exempt from prepaid balance.
- No lease-based balance subsystem in phase 1.
- RPM and per-org/arm caps enforced.
- Usage events durable and visible in dashboards/admin tools.
- Billing/finalize code human-reviewed.
- Skupper path measured before large infra commitments.
- Clear rollback tested with in-flight requests.

This still validates the core flexgate value: cheaper, faster request relay under real traffic, without coupling the first launch to every billing and balance mechanism.

## Go / No-Go Checklist

Before production `on`, require:

- [ ] Balance model decision recorded.
- [ ] Two-human review of billing/finalize/control-plane money paths.
- [ ] Dashboard, export, admin, and request lookup support flexgate traffic.
- [ ] Cancel/disconnect billing tests pass across old and new arms.
- [ ] Duplicate, replay, void, and pod-death usage tests pass.
- [ ] Redis-gw outage behavior is tested.
- [ ] Rollback is tested with in-flight traffic.
- [ ] Per-tenant fairness limits exist.
- [ ] Skupper ceiling is measured.
- [ ] Production runbooks and owners exist.
- [ ] Exposed secrets are rotated.

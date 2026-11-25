# Blueprint: Implementation Plan

> AI-assisted desired-state management for large fleets of edge Kubernetes clusters.

Status: draft v0.1 · 2026-11-25

---

## 1. What Blueprint does

Each edge cluster runs a **Blueprint Agent**. Every agent keeps an outbound connection to the **Blueprint Hub**. The hub:

1. Holds a live copy of each cluster's **observed state**.
2. Compares it with that cluster's **desired state** (a layered baseline).
3. When they differ, asks a model to produce a **remediation plan**. The hub checks the plan against policy and runs it through the agent's tools, then confirms the drift is gone.
4. Lets operators **change the desired state in natural language**. The model proposes a diff to the baseline, and a person reviews and merges it like a PR.

### Design principles

- **Deterministic first, model second.** Drift detection is a plain structural diff. The model only plans *how* to close a gap that code has already found and measured. It never decides *whether* drift exists.
- **The model proposes; policy decides.** Every model-produced action goes through a policy engine and, when needed, a human approval gate before anything runs. The agent applies its own local policy as well, so a compromised hub can't do anything it likes.
- **Plan once, apply many.** At fleet scale, the same drift shows up on hundreds of clusters. Fingerprint the drift, reuse the plan, and roll it out in waves.
- **Edge is unreliable.** Agents dial out, work through NAT, tolerate disconnects, and resync after reconnecting.

---

## 2. Architecture

```
                     ┌──────────────────────── Blueprint Hub ───────────────────────┐
  Operators ──UI/CLI─▶  API (gRPC + REST)                                            │
                     │    │                                                          │
                     │    ├─ Desired-State Store  (versioned, layered docs; Git mirror)
                     │    ├─ Observed-State Store (Postgres JSONB + history)        │
                     │    ├─ Drift Engine         (deterministic diff → DriftReport) │
                     │    ├─ Planner              (Claude + tool schemas → Plan)     │
                     │    ├─ Policy Engine        (OPA/Rego: allow / approve / deny) │
                     │    ├─ Rollout Orchestrator (waves, canaries, circuit breaker) │
                     │    ├─ Change Assistant     (NL → desired-state diff)          │
                     │    └─ Agent Gateway        (mTLS WebSocket tunnels, MCP client)
                     └───────────────▲───────────────────────────────────────────────┘
                                     │  agent-initiated, long-lived, mTLS
          ┌──────────────────────────┼──────────────────────────┐
   ┌──────┴──────┐            ┌──────┴──────┐            ┌──────┴──────┐
   │ Edge cluster│            │ Edge cluster│    …       │ Edge cluster│
   │  bp-agent   │            │  bp-agent   │            │  bp-agent   │
   │  (MCP srv)  │            │  (MCP srv)  │            │  (MCP srv)  │
   └─────────────┘            └─────────────┘            └─────────────┘
```

### 2.1 Blueprint Agent (`bp-agent`)

- **Form:** a single Go binary, deployed as a Deployment in a `blueprint-system` namespace through a Helm chart. It uses a scoped ServiceAccount, and its RBAC is configurable per profile: read-only, standard, or admin.
- **Role:** an **MCP server** whose transport runs *over the agent's own outbound tunnel*. Edge sites are usually behind NAT, so the hub can't dial in. Instead the agent opens a WebSocket to the hub, and the hub acts as the MCP client over it, sending JSON-RPC MCP messages through that socket. This keeps the MCP tool model while flipping who opens the connection.
- **State sync:** client-go informers watch the configured resource kinds. The agent sends a full snapshot when it connects, then compressed deltas tagged with `resourceVersion`. Heartbeats carry a state hash, and the hub requests a resync when hashes don't match.
- **MCP tools (v1):**

  | Tool | Kind | Notes |
  |---|---|---|
  | `get_resources(kind, ns, selector)` | read | |
  | `get_cluster_info()` | read | version, nodes, capacity, CNI/CSI |
  | `get_logs(pod, container, tail)` | read | output is redacted |
  | `get_events(ns, since)` | read | |
  | `diff_manifest(manifest)` | read | server-side dry-run diff |
  | `apply_manifest(manifest, dry_run)` | write | server-side apply, field manager `blueprint` |
  | `delete_resource(kind, ns, name)` | write | denied by default locally |
  | `helm_upgrade(release, chart, version, values)` | write | |
  | `rollout_restart(kind, ns, name)` | write | |
  | `scale(kind, ns, name, replicas)` | write | |
  | `exec_runbook(id, params)` | write | runs only signed, pre-registered runbooks; no arbitrary shell |

- **Local guardrails:** a local policy file covers protected namespaces, forbidden kinds, and a cap on how many resources one action can touch. Every write needs a **hub-signed action envelope** containing the plan ID, a hash of the action, and an expiry. The agent rejects unsigned or expired actions and keeps a local audit log.
- **Offline behaviour:** state deltas are buffered locally and capped. The agent keeps the last approved desired-state revision. In phase 6 it can optionally re-apply that revision deterministically when it detects drift while disconnected. No model is ever involved on the edge.

### 2.2 Blueprint Hub

- **Language:** Go, so the hub shares API types and client-go with the agent.
- **Storage:** Postgres. Observed state goes in JSONB per resource with append-only history. Desired-state revisions, plans, approvals, and audit records also live in Postgres. Plan fan-out at scale uses NATS JetStream, which can be added later.
- **Desired-state model:** layered documents, similar to Kustomize:
  `global baseline → fleet group (e.g. region, hardware SKU) → site → cluster overrides`.
  Each layer is a set of Kubernetes manifests, Helm releases, and **invariants** (assertions such as "no pod runs privileged outside `kube-system`" or "node OS ≥ SLE Micro 6.1"). Layers are stored as immutable revisions with author, message, and parent, and can be mirrored to a Git repo for GitOps users.
- **Drift Engine:** takes the rendered desired state for each cluster and the observed state, and produces a `DriftReport`: a list of `{resource, path, expected, actual, severity}` entries plus any failed invariants. Each report gets a normalised **drift fingerprint** that ignores cluster-specific values (names, IPs, UIDs), so identical drift on many clusters groups together.
- **Planner (model):** described in §3.
- **Policy Engine:** OPA/Rego evaluates every planned action and classifies it as `auto` (low risk, such as restoring a ConfigMap value), `approve` (needs a human), or `deny`. Inputs include action type, namespace, blast radius, cluster tier (prod or lab), and time window.
- **Rollout Orchestrator:** applies an approved plan to every cluster sharing the fingerprint, in waves: canary at 1%, then 10%, then 50%, then 100%. A verification gate between waves re-runs the diff and checks health. A circuit breaker stops the rollout when the failure rate crosses a threshold.
- **Change Assistant:** described in §4.
- **UI:** a fleet map and drift heatmap, a plan review screen (diff, model rationale, policy verdict), a desired-state history, and an audit trail. CLI: `bp`.

---

## 3. AI remediation loop

```
DriftReport ─▶ fingerprint ─▶ cached plan? ──yes──▶ policy ─▶ rollout
                    │ no
                    ▼
            Planner (Claude, read-only tools against a representative cluster)
                    │  structured Plan
                    ▼
            validate (schema, dry-run on target) ─▶ policy ─▶ approval? ─▶ rollout
                                                                       │
                                            verify (re-diff) ◀─────────┘
                                                    │ still drifted → re-plan (max N) → escalate to human
```

### 3.1 Planner design

- **Who runs the loop:** the hub runs it with the Messages API and tool use. Use the SDK Tool Runner or a manual loop. The hub hosts the loop because the tools only exist behind the hub's agent tunnels and approval gates sit inside the loop. Managed Agents isn't a good fit here.
- **Tools exposed to the model:** the hub re-exports the agent's MCP tools as **client tools**, with `strict: true` schemas. During planning the model gets only the **read** tools plus a `propose_plan` tool. It never gets write tools directly. Writes happen later and only from an approved `Plan`. This keeps model output separate from execution.
- **Why not the API's MCP connector:** that connector needs a URL the API can reach. Edge agents can't be reached that way, and we want the hub in the middle of every call regardless, for policy, audit, and redaction.
- **Output:** a structured `Plan` gathered through the `propose_plan` tool or structured outputs:
  `{ summary, rationale, risk, steps: [{tool, args, expected_effect, rollback}], verification: [checks] }`.
- **Model:** `claude-opus-5-5` with adaptive thinking. Set effort explicitly: the default is `medium`, but use `high` for novel drift and `low` for summarising or re-planning cached fingerprints. On this model, forced `tool_choice` returns a 400, so use `auto` and tell the model in the prompt to finish by calling `propose_plan`. Turn on server-side refusal fallbacks (`fallbacks: "default"`) and check `stop_reason` before parsing.
- **Prompt caching:** order the request as tools (static, sorted) → system prompt (static) → the fleet group's rendered baseline (changes per revision) → the DriftReport. Put a cache breakpoint after the baseline so every cluster in a group reuses the cached prefix.
- **Batch API:** use it for non-urgent fleet-wide work, such as nightly drift triage and summaries across thousands of clusters, at about half the cost. Interactive and incident remediation uses normal or streaming calls.
- **Prompt injection:** cluster data (annotations, logs, ConfigMap contents) is **untrusted**. Wrap it in clearly labelled data blocks, redact secrets before they reach the model, and rely on policy plus signed envelopes. A malicious annotation can at worst produce a bad *proposal*, which policy and review then catch.

### 3.2 Plan reuse at scale

The plan cache is keyed by `(drift fingerprint, desired-state revision, cluster profile)`. A cached plan is re-validated with `diff_manifest` (dry-run) on each target before it's applied. If 2,000 clusters show the same drift, the system makes one model call and runs 2,000 dry-runs.

---

## 4. Natural-language desired-state changes

1. The operator asks: *"Roll out Longhorn 1.12 to all retail-EU sites, 2 replicas, keep strict-local."*
2. The Change Assistant (Claude) gets read tools over the desired-state store: list layers, read a layer, render a cluster, and query the fleet inventory. It returns a **structured change set**: a patch to specific layers plus an explanation.
3. Validation: schema and Kubernetes OpenAPI checks, a render for every affected cluster, invariant checks, and a dry-run on a sample of real clusters.
4. The result appears as a **change request**: a PR-style view showing the diff, the clusters affected, the estimated drift each would get, and the risk.
5. Approval merges it as a new revision. The drift engine then sees the new gap, and the normal remediation loop with rollout waves applies it.

Every desired-state change goes through review, so the model never changes the baseline directly.

---

## 5. Security model

- **Identity:** each agent gets an X.509 identity (SPIFFE-style ID `spiffe://blueprint/<fleet>/<cluster>`) through one-time enrolment tokens. Certificates are short-lived and rotate automatically.
- **Authorisation:** hub RBAC (viewer, operator, approver, admin), scoped by fleet group.
- **Execution:** hub-signed action envelopes, an agent-side local policy, and no arbitrary command execution. Runbooks are signed and registered ahead of time.
- **Data:** secrets are redacted on the agent before they're sent. A per-fleet setting controls which resource kinds may ever reach the model.
- **Audit:** an append-only log of every model call (prompt hash, response, tokens), plan, approval, and action result.

---

## 6. Phased delivery

| Phase | Scope | Exit criteria |
|---|---|---|
| **0 · Foundations** (2 wks) | Repo, CI, Go module layout, protobuf/API types, dev fleet simulator (k3d/kind, about 20 clusters) | `make dev-fleet` starts hub + 20 agents locally |
| **1 · Connect & observe** (4 wks) | Agent tunnel, enrolment and mTLS, informer state sync, hub inventory, read-only MCP tools, basic UI fleet view | 500 simulated clusters stay synced with 1 hub replica; resync works after a forced disconnect |
| **2 · Desired state & drift** (4 wks) | Layered desired-state store with revisions, renderer, invariants, deterministic drift engine, fingerprinting, drift heatmap | Seeded drift scenarios found with 100% accuracy and no false positives on a clean fleet |
| **3 · AI planning (propose only)** (4 wks) | Planner with read-only tools, structured plans, plan cache, policy engine, review UI. **No execution.** | Planner eval suite (§7) ≥ 90% acceptable plans on golden scenarios |
| **4 · Execute & verify** (4 wks) | Signed envelopes, write tools, agent local policy, rollout waves, verification, circuit breaker, audit | End-to-end auto-remediation of low-risk drift on the dev fleet; canary failure stops the rollout |
| **5 · NL desired-state changes** (3 wks) | Change Assistant, change requests, sample dry-run | Operators make a typical change (version bump, new app, config tweak) from one prompt |
| **6 · Scale & hardening** (ongoing) | Hub HA (stateless replicas behind an L4 LB, sticky by agent), NATS fan-out, Batch API triage, offline local reconcile, 10k-cluster load test, pen test | 10k simulated agents; p99 drift detection < 60 s after a change |

---

## 7. Testing & evaluation

- **Unit and integration:** standard Go tests. Integration tests run against kind clusters in CI.
- **Fleet simulator:** a lightweight fake agent that replays recorded state, used to load-test the hub at 10k+ connections without running real clusters.
- **Planner evals:** a golden set of drift scenarios, such as deleted Deployment, wrong image tag, altered Longhorn replica count, failed invariant, drift combined with a crashlooping pod, and an injection attempt hidden in an annotation. Grade plans on correctness after applying to a sandbox cluster (re-diff is clean), minimality, safety (policy verdict), and rationale quality using an LLM judge. Run the evals in CI whenever a prompt, tool schema, or model changes.
- **Chaos:** drop agent connections, partition the hub, kill an agent halfway through a plan, and confirm the system recovers idempotently.

---

## 8. Proposed repo layout (new repo: `blueprint`)

This repo holds the single-file Longhorn sizing calculator, and its CLAUDE.md says not to add build tooling. Blueprint should live in **its own repository**. This plan sits here only as a starting point.

```
blueprint/
  cmd/bp-agent/   cmd/bp-hub/   cmd/bp/          # binaries
  api/proto/                                     # agent⇄hub protocol, public API
  internal/agent/{tunnel,mcp,state,policy,tools}
  internal/hub/{gateway,store,drift,planner,policy,rollout,change,audit}
  pkg/desired/                                   # layer model + renderer (shared)
  policies/                                      # default Rego
  deploy/helm/{agent,hub}
  web/                                           # UI
  sim/                                           # fleet simulator
  evals/                                         # planner golden scenarios
```

---

## 9. Open questions

1. **Scope of "state":** Kubernetes only, or also the host OS (SLE Micro packages, kernel params) and firmware? Host OS needs a node-level agent component.
2. **Relationship to existing tools:** should Blueprint replace Fleet/Rancher/Argo CD, or sit above them and drive them? Driving Fleet GitRepos through the Git mirror may be the quickest way to adoption.
3. **Autonomy defaults:** which drift classes may auto-remediate without a person, and is that set per fleet group?
4. **Model hosting:** Anthropic API directly, or Bedrock/Vertex for data residency? Air-gapped customers would need a different planner path.
5. **Fleet size target:** the plan assumes up to 10k clusters per hub. Is 100k in scope, which would mean hub sharding?
6. **Agent footprint budget:** for small edge nodes, a target such as < 50 MiB RSS and < 50 m CPU.

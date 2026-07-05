# 06 — Cross-Cluster Correlation & Log Continuity

## Context

The system currently investigates one cluster/context at a time
(Prompts 02–05), and only pulls current + previous container logs for
pods flagged at the moment an investigation runs.

Two real gaps fall out of that design:

1. **Cross-cluster blast radius.** Some failures aren't contained to one
   cluster — a shared secret rotated upstream can cause the same
   CrashLoopBackOff across every cluster that references it, or a
   frontend in one cluster can't reach a backend service that lives in
   another. Today, investigating each cluster separately means the
   system never notices the two failures are the same incident.

2. **Log rotation / pod churn.** If a failing pod has already been
   replaced by the time an investigation starts, its logs are gone.
   Kubernetes doesn't keep them around indefinitely, and today the system
   has no fallback beyond whatever the Events Analyzer already captured.

This prompt builds both fixes. Neither changes existing single-cluster
behavior — they extend it.

---

## Goal

```text
Cross-Cluster Investigation Orchestrator
Cross-Cluster Correlator (AI component)
Log Availability Tracking
Log Archiver (persistent snapshot layer)
Confidence Engine adjustment for degraded evidence
```

---

## Part A — Cross-Cluster Root Cause Correlation

### 1. Request contract (backward compatible)

Extend `POST /investigate`'s request body — do not replace it:

```json
// existing single-cluster call, unchanged behavior
{ "context": "docker-desktop" }
```

```json
// new multi-cluster call
{ "contexts": ["docker-desktop", "minikube"] }
```

Validation:
- Exactly one of `context` or `contexts` must be present — reject both or
  neither with `422`.
- `contexts` with fewer than 2 entries is a client error — that's just
  the existing single-cluster path, tell the caller to use `context`
  instead.

### 2. Cross-Cluster Investigation Orchestrator

- Runs the existing per-cluster Investigation Service (Prompt 02) against
  every requested context, **in parallel**, not sequentially — one slow
  or unreachable cluster shouldn't block the others.
- Each cluster's investigation and diagnosis is independent and complete
  on its own — a failure connecting to one cluster doesn't cancel the
  others; it's reported per-cluster in `errors`, same pattern as Prompt 02.

### 3. Cross-Cluster Correlator (new AI component)

Runs **after** every per-cluster diagnosis (Prompt 03) completes. Looks
for two patterns specifically:

```text
Shared root cause — the same underlying issue (e.g. same missing env
var, same expired credential, same misconfigured secret) appears
independently in multiple clusters.

Causal chain — a failure in one cluster is explained by something in
another (e.g. Cluster A's pods can't resolve/reach a service that
actually lives in Cluster B).
```

- Input to this component is the **set of per-cluster diagnoses**, not
  raw evidence — it reasons over conclusions, not logs, keeping the
  prompt small and the reasoning auditable.
- If no cross-cluster relationship is found, say so explicitly rather
  than forcing a connection — a null/independent result is a valid and
  common outcome, not a failure.
- Output schema:

```python
class CrossClusterFindings(BaseModel):
    correlated: bool
    relationship_type: Literal["shared_root_cause", "causal_chain", "none"]
    summary: str
    involved_clusters: list[str]
    confidence: int
```

### 4. Response shape

```json
{
  "status": "success",
  "mode": "multi_cluster",
  "per_cluster": [
    {
      "context": "docker-desktop",
      "investigation": {},
      "diagnosis": {}
    },
    {
      "context": "minikube",
      "investigation": {},
      "diagnosis": {}
    }
  ],
  "cross_cluster_findings": {
    "correlated": true,
    "relationship_type": "shared_root_cause",
    "summary": "Both clusters reference the same expired API credential.",
    "involved_clusters": ["docker-desktop", "minikube"],
    "confidence": 81
  }
}
```

Single-cluster calls keep returning exactly what Prompt 03 already
defined — no `mode`, no `per_cluster` wrapper, no `cross_cluster_findings`.
That shape is untouched.

### 5. Frontend — Cluster Selector, multi-select mode

Extend the Cluster Selector from Prompt 04 with a mode toggle:

```text
( ) Investigate one cluster
( ) Investigate across clusters

☐ docker-desktop  (current)
☐ minikube
☐ staging-cluster
```

- Single mode keeps the existing radio-button behavior exactly as built
  in Prompt 04.
- Multi mode switches to checkboxes; the CTA becomes
  `[ Investigate 2 clusters ]` and updates live as selections change.
- Stay within the black-and-white system: selected = filled black square,
  unselected = outlined square — same visual language as the existing
  selected/unselected state, just a different shape for "select many"
  vs. "select one."
- If a cross-cluster relationship is found, show it as a distinct card
  above the per-cluster Root Cause Cards, clearly labeled
  `Cross-Cluster Finding`, so it doesn't get lost among individual
  diagnoses.
- History rows for multi-cluster runs list all involved clusters
  (e.g. `docker-desktop + minikube`) rather than a single cluster name.

---

## Part B — Log Continuity

### 6. Log Availability Tracking (extends Prompt 02's Logs Collector)

For each flagged pod, attempt logs in this order and record what
actually worked:

```text
1. Current container logs
2. Previous container logs (--previous), if the pod restarted
3. Sibling pod under the same deployment/replicaset with a matching
   failure signature (same image, same status), if the original pod
   no longer exists
```

Add explicit tracking to the payload instead of silently returning
nothing:

```json
{
  "logs_available": false,
  "log_source": "sibling_pod",
  "log_gap_reason": "Original pod no longer exists; logs sourced from sibling pod payment-service-9f2k1 with matching CrashLoopBackOff state.",
  "excerpt": "..."
}
```

If sibling-pod logs are used, label them clearly in the payload and the
eventual prompt to the LLM — they're corroborating evidence, not a
direct capture of the original failure, and should never be presented as
if they were.

### 7. Log Archiver (new component)

A structural fix for the general problem, not just this one
investigation:

- On **every** investigation, persist a trimmed log excerpt (a few KB
  max, same size limit as Prompt 02) to InsForge, keyed by
  `context + namespace + deployment` — **not** by pod name, since pod
  names churn but the deployment identity is stable.
- On a future investigation of that same deployment, if live logs and
  sibling-pod logs both come up empty, fall back to the most recent
  archived snapshot, explicitly labeled with its original timestamp:

```text
"No live logs available. Showing archived logs from a prior
investigation (2026-07-03 14:02 UTC) — may not reflect the current
failure."
```

- Be upfront about the honest limitation this doesn't solve: the
  **first-ever** investigation of a given workload still has no archive
  to fall back on if logs have already rotated away. This component
  builds a safety net for every investigation after that one — it
  doesn't retroactively recover logs that were never captured.
- Archived entries older than a configurable retention window (e.g. 7
  days) should be pruned rather than accumulating indefinitely.

### 8. Confidence Engine adjustment (updates Prompt 03)

Update the Prompt Builder / Confidence Engine instructions so degraded
evidence is reflected honestly in the score, not papered over:

```text
If logs_available is false and the diagnosis relies on sibling-pod
logs, archived logs, or events alone, confidence must be capped lower
than a case with direct current-pod logs, and confidence_reasoning
must state which fallback was used and why that lowers certainty.
```

This is a direct extension of Prompt 03's existing principle that
confidence should track evidence strength — it just gives the model a
concrete new signal to factor in.

---

## Constraints

Do **NOT**:
- Change the existing single-cluster request/response contract from
  Prompts 02/03 — `{ "context": "..." }` must behave exactly as before.
- Turn the Log Archiver into a background poller or watcher — it only
  writes on investigations a user actually triggered, consistent with
  this being an on-demand tool, not a controller.
- Silently substitute sibling-pod or archived logs without labeling them
  as such — every fallback must be visible to both the LLM prompt and
  the frontend.
- Force a cross-cluster correlation when none genuinely exists —
  `correlated: false` is a normal, expected result for most multi-cluster
  runs.

Do **NOT** break Prompts 01–05.

---

## Expected Result

```http
POST /investigate
{ "contexts": ["docker-desktop", "minikube"] }
```

returns independent diagnoses for both clusters, plus an honest
assessment of whether they're related — and single-cluster calls keep
working exactly as they did before this prompt.

Separately, an investigation into a pod whose logs have already rotated
away no longer returns nothing — it falls back through sibling pods, then
the archive, clearly labels which one it used, and reflects that
uncertainty in the confidence score instead of hiding it.

## Definition of Done (checklist)

- [ ] `POST /investigate` accepts `context` (unchanged) or `contexts` (new); rejects both/neither with `422`
- [ ] Existing single-cluster response shape is byte-for-byte unchanged
- [ ] Multi-cluster investigations run in parallel, not sequentially
- [ ] Cross-Cluster Correlator reasons over diagnoses, not raw evidence
- [ ] `correlated: false` is treated as a normal result, not suppressed or forced positive
- [ ] Logs Collector records `logs_available`, `log_source`, and `log_gap_reason` for every flagged pod
- [ ] Sibling-pod and archived logs are visibly labeled as fallbacks, never presented as direct evidence
- [ ] Log Archiver keys entries by context + namespace + deployment, not pod name, and prunes on a retention window
- [ ] Confidence score is visibly lower, with a stated reason, whenever a log fallback was used
- [ ] Cluster Selector's multi-select mode stays within the black-and-white design system
- [ ] Prompts 01–05 functionality is fully intact
# 02 — Kubernetes Investigation Engine

## Context

Project setup (Prompt 01) is complete: backend, frontend, Docker, and the
black-and-white design system are in place.

We now build the **Kubernetes Investigation Layer**.

```text
Frontend
    ↓
FastAPI Backend (Orchestrator)
    ↓
Kubernetes Investigation Layer
        ├── Cluster Registry (kubeconfig contexts)
        ├── Check Pods
        ├── Read Logs
        ├── Analyze Events
        ├── Inspect Deployments
        └── Check Networking
    ↓
AI Kubernetes Agent
```

This layer behaves like a **junior DevOps engineer collecting evidence**
before AI reasoning starts. It only gathers facts — no interpretation, no
root cause, no fixes.

> ⚠️ Still **NOT** implementing AI reasoning. Evidence gathering only.

### Multi-cluster requirement (new)

A local kubeconfig can contain multiple clusters (contexts) — e.g.
`docker-desktop`, `minikube`, `staging-cluster`. This system must support
investigating **any** of them, not just whatever `kubectl` treats as the
current context. Every kubectl call in this layer must be able to target an
explicit context, and the backend must expose a way to list what's
available. This is what the frontend dashboard (Prompt 04) will use to let
the user click a cluster before investigating.

Use **kubectl commands internally** (subprocess) — not the Kubernetes
Python SDK. Keep it simple and inspectable.

---

## Goal

Build the Kubernetes Investigation Layer as modular, testable components:

```text
Cluster Registry
Kubectl Executor
Pod Inspector
Logs Collector
Events Analyzer
Deployment Inspector
Network Inspector
Investigation Service (orchestrator)
```

FastAPI exposes this layer through two endpoints: list clusters, run an
investigation against one of them.

---

## Requirements

### 1. Cluster Registry

Responsible for discovering and validating kubeconfig contexts.

- Read contexts via `kubectl config get-contexts -o name` (names) and
  `kubectl config view -o json` (to identify the current context and
  cluster server info).
- Expose:
  - `list_contexts() -> list[ClusterInfo]`
  - `context_exists(name: str) -> bool`
- `ClusterInfo` (Pydantic model):

```json
{
  "name": "docker-desktop",
  "is_current": true
}
```

- If `KUBECONFIG_PATH` is set (from Prompt 01's env var), read from that
  path explicitly rather than relying on the shell's default.
- If no contexts are found, return an empty list — do not raise. The API
  layer decides how to surface that to the user.

---

### 2. Kubectl Executor

A single, reusable utility that every other component calls through — no
component should shell out to `kubectl` directly.

Requirements:

- Built on `subprocess.run`, never `shell=True`.
- Accepts an optional `context: str | None`. When provided, every command
  is run with `--context <name>` appended.
- Captures stdout, stderr, and exit code separately.
- Enforces a timeout on every call (default from config, e.g. 15s) so a
  hung cluster can't hang the whole investigation.
- Returns a structured result, never a raw string blob:

```python
class KubectlResult(BaseModel):
    success: bool
    command: str
    stdout: str
    stderr: str
    exit_code: int
```

- Logs every invocation (command + context + duration) at INFO, failures
  at WARNING, with stderr included.
- Never raises on a failed kubectl call — callers inspect `success` and
  decide how to degrade.

Example supported commands (all context-aware):

```bash
kubectl --context <ctx> get pods -A -o json
kubectl --context <ctx> get events -A -o json
kubectl --context <ctx> logs <pod-name> -n <namespace> --tail=200
kubectl --context <ctx> describe deployment <deployment-name> -n <namespace>
kubectl --context <ctx> get svc -A -o json
```

Prefer `-o json` over parsing human-readable `kubectl` output wherever
possible — it's far more reliable than screen-scraping text.

---

### 3. Pod Inspector

Detects unhealthy pods and classifies the failure type:

```text
CrashLoopBackOff
ImagePullBackOff / ErrImagePull
Pending (unscheduled)
Error
OOMKilled
ContainerCreating (stuck beyond a threshold)
```

Structured output:

```json
{
  "healthy": false,
  "checked_namespaces": ["default", "kube-system"],
  "problematic_pods": [
    {
      "name": "payment-service-7d8f9",
      "namespace": "default",
      "status": "CrashLoopBackOff",
      "restart_count": 7,
      "container": "payment-service"
    }
  ]
}
```

---

### 4. Logs Collector

Fetches recent logs **only for pods the Pod Inspector flagged** — don't
pull logs for every pod in the cluster.

Focus extraction on:

```text
Exceptions / stack traces
Connection failures
Missing environment variables / config
Image or startup failures
```

Constraints:

- Tail logs (e.g. last 200 lines), never fetch full history.
- If a pod has restarted, prefer the **previous** container's logs
  (`kubectl logs --previous`) since that's usually where the failure is.
- Truncate any single log excerpt to a few KB max before returning it —
  this payload will later be sent to an LLM, so keep it lean.

---

### 5. Events Analyzer

Reads and summarizes cluster events, filtered to warning-level reasons:

```text
FailedScheduling
BackOff
FailedMount
FailedPull / ErrImagePull
Unhealthy
```

Group repeated events (same reason + object) into a single summarized
entry with a count, rather than returning every duplicate.

---

### 6. Deployment Inspector

Checks deployment health:

```text
Desired vs. available vs. unavailable replicas
Rollout status / stuck rollouts
Deployment conditions (Available, Progressing)
```

Flags a deployment as unhealthy if available replicas < desired for
longer than a configurable grace period, or if a condition reports
`False`.

---

### 7. Network Inspector

Checks service/networking health:

```text
Service exists for the workload
Selector matches at least one pod's labels
Endpoints object is non-empty
```

DNS-level checks (e.g. resolving a service name from inside the cluster)
are out of scope for this prompt — flag selector/endpoint mismatches only.

---

### 8. Investigation Service (Orchestrator)

Runs the full pipeline for a **specific context**, in order:

```text
Resolve & validate context
    ↓
Check Pods
    ↓
Collect Logs (for flagged pods only)
    ↓
Analyze Events
    ↓
Inspect Deployments
    ↓
Check Networking
```

- Each step's failure should not abort the whole investigation — if
  `kubectl get events` fails, still return what pod/log data was
  collected, with an explicit `errors` list noting what step failed.
- Returns one structured payload:

```json
{
  "context": "docker-desktop",
  "pods": {},
  "logs": {},
  "events": {},
  "deployments": {},
  "network": {},
  "errors": []
}
```

---

## FastAPI API

### `GET /clusters`

Lists kubeconfig contexts available to investigate.

```json
{
  "clusters": [
    { "name": "docker-desktop", "is_current": true },
    { "name": "minikube", "is_current": false }
  ]
}
```

### `POST /investigate`

Request body:

```json
{ "context": "minikube" }
```

- `context` is optional. If omitted, use the current kubeconfig context.
- If `context` is provided but not found via the Cluster Registry, return
  `422` with a clear error message rather than silently falling back.

Response:

```json
{
  "status": "success",
  "investigation": {
    "context": "minikube",
    "pods": {},
    "logs": {},
    "events": {},
    "deployments": {},
    "network": {},
    "errors": []
  }
}
```

No AI yet. No root cause analysis yet. This step is evidence gathering
only, scoped to one explicit cluster per call.

---

## Constraints

Do **NOT** implement:
- OpenRouter / any LLM reasoning
- Root cause analysis or fix recommendation
- InsForge integration
- Authentication
- Realtime updates

Do **NOT** use the Kubernetes Python SDK — kubectl subprocess calls only.

Do **NOT** break the project-setup foundation from Prompt 01 (health
endpoint, CORS, logging, config, black-and-white frontend shell all keep
working).

Keep every component small, typed, and independently testable.

---

## Expected Result

```http
GET /clusters
```
returns every cluster found in the local kubeconfig, correctly flagging
the current one.

```http
POST /investigate
{ "context": "docker-desktop" }
```
returns structured, real Kubernetes troubleshooting evidence for that
specific cluster — pods, logs, events, deployments, and networking findings
— with zero AI interpretation applied.

## Definition of Done (checklist)

- [ ] `GET /clusters` lists all kubeconfig contexts with `is_current` set correctly
- [ ] `POST /investigate` accepts an optional `context` and targets it end-to-end
- [ ] Invalid/unknown context returns `422`, not a silent fallback
- [ ] Kubectl Executor never uses `shell=True`, always has a timeout, and returns structured `KubectlResult`
- [ ] Logs are pulled only for flagged pods, tailed and size-limited
- [ ] Events are deduplicated/summarized, not raw event dumps
- [ ] A single component failure (e.g. events call fails) doesn't abort the whole investigation — it's captured in `errors`
- [ ] No Kubernetes SDK usage anywhere — kubectl subprocess only
- [ ] No AI, auth, or realtime logic present yet
- [ ] Prompt 01's `/health` endpoint and frontend still work unmodified
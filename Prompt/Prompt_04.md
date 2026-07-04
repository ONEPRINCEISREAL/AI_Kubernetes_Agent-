# 04 — Dashboard & Frontend API Integration

## Context

The backend can now, for any specific cluster/context:

```text
Investigate Kubernetes
        ↓
Collect Evidence
        ↓
AI Reasoning
        ↓
Root Cause Analysis
        ↓
Suggested Fix
```

```text
Frontend
    ↓
FastAPI Backend (Orchestrator)
    ↓
Kubernetes Investigation Layer
    ↓
AI Kubernetes Agent
    ↓
LLM Reasoning
(OpenRouter via InsForge Key)
    ↓
Root Cause + Suggested Fix
```

Goal: turn this into a real application experience. The user should be
able to:

```text
See every cluster in their local kubeconfig
        ↓
Pick one
        ↓
Click Investigate
        ↓
Watch live investigation progress
        ↓
Receive a diagnosis
        ↓
View investigation history (per cluster)
```

Use InsForge for **authentication, investigation history, and realtime
updates**. FastAPI remains the orchestrator — InsForge does not replace
it.

---

## Goal

Build the **Investigation Dashboard**:

```text
Cluster Selector      (new — reads GET /clusters from Prompt 02)
Authentication (InsForge)
Realtime Investigation Progress
Root Cause Card
Investigation History (per cluster)
Frontend → Backend Integration
```

Keep the UI minimal, black-and-white (per Prompt 01's design system), and
easy to explain in a short walkthrough.

---

## Requirements

### 1. Authentication (InsForge)

- Login support, protected dashboard, basic session handling.
- Only authenticated users can select a cluster, trigger an investigation,
  or view history.
- Keep it minimal — no roles/permissions system, just "logged in or not."

---

### 2. Cluster Selector (new)

Before anything else on the dashboard, show the clusters available from
the user's local kubeconfig, fetched from `GET /clusters` (Prompt 02).

```text
Select a Cluster

●  docker-desktop   (current)
○  minikube
○  staging-cluster
```

Behavior:
- Render as a simple clickable list, one item per context returned by the
  API — not a raw dropdown buried in a form; the user should be able to
  see all clusters at a glance.
- Selected state uses the black-and-white system from Prompt 01: selected
  = solid black background / white text; unselected = white background,
  black border. No color is used to indicate selection.
- The context flagged `is_current: true` by the API is preselected by
  default, but the user can click any other one.
- The "Investigate Cluster" button is disabled until a cluster is selected
  and stays labeled with which cluster it will target, e.g.
  `[ Investigate "minikube" ]`.
- Empty state — if `GET /clusters` returns zero contexts:

```text
No clusters found in your kubeconfig.
Check KUBECONFIG_PATH and cluster access, then refresh.
```

The selected cluster name is passed as `context` in the `POST /investigate`
request body (per the Prompt 02/03 contract) and is carried through the
rest of this screen — progress, diagnosis, and history should all clearly
show which cluster they belong to.

---

### 3. Investigation Dashboard Layout

**Header**
```text
AI Kubernetes Agent
```

**Cluster Selector** (above) →
**Main CTA**
```text
[ Investigate "docker-desktop" ]
```

**Investigation Progress** — realtime, via InsForge:

```text
Investigating "docker-desktop"

✓ Checking Pods
✓ Reading Logs
✓ Analyzing Events
✓ Inspecting Deployments
✓ Checking Networking
✓ AI Reasoning
✓ Root Cause Found
```

Represent each step's state (pending / in-progress / done / failed) with
shape and weight, not color — e.g. a hollow circle for pending, a spinner
outline for in-progress, a filled circle for done, an "×" mark for failed
— consistent with the black-and-white status pattern from Prompt 01.

---

### 4. Root Cause Card

Displays the diagnosis returned by `POST /investigate` (Prompt 03):

```text
Cluster: docker-desktop

Root Cause
DATABASE_URL missing

Explanation
Application failed during startup.

Suggested Fix
Add the missing environment variable.

kubectl Command
kubectl edit deployment payment-service -n default

Confidence
92%
```

Keep it a single clean card — no charts, no colored severity badges.
Confidence can be shown as a plain percentage plus its one-line reasoning
from Prompt 03's `confidence_reasoning`, not a colored gauge.

---

### 5. Investigation History (per cluster)

Persist investigations via InsForge. Store per record:

```text
Timestamp
Cluster / context
Root Cause
Confidence
Status (success / partial / failed)
```

Display as a simple table, most recent first:

```text
Recent Investigations

Time       Cluster          Root Cause          Confidence
14:02      docker-desktop   CrashLoopBackOff     92%
13:47      minikube         ImagePullBackOff     88%
13:10      docker-desktop   OOMKilled            95%
```

No filters, no pagination beyond a reasonable default limit (e.g. last 20)
for this prompt.

---

### 6. Frontend API Integration

```text
User selects a cluster
        ↓
User clicks "Investigate <cluster>"
        ↓
Frontend calls POST /investigate { context }
        ↓
Backend runs investigation + AI reasoning
        ↓
Realtime progress updates (InsForge)
        ↓
Diagnosis returned
        ↓
UI updates: Root Cause Card + History table
```

Handle explicitly:
```text
Loading state (button disabled + progress shown while in flight)
API failures (see error copy below)
Empty diagnosis (cluster healthy — no issues found)
Timeouts
```

---

## Constraints

Do **NOT**:
- Change the working backend investigation or AI reasoning logic from
  Prompts 02/03 — this prompt is frontend + InsForge integration only.
- Introduce color into the UI — stay within the black-and-white system.
- Add charts or complex data visualizations.
- Add complex state management libraries — React Query + local component
  state is sufficient.

Use InsForge only for: authentication, realtime updates, investigation
history. FastAPI remains the single orchestrator for investigation logic.

Do **NOT** break Prompts 01–03.

---

## Expected Result

A logged-in user can:

```text
Open the dashboard
        ↓
See every cluster from their local kubeconfig
        ↓
Pick one and click Investigate
        ↓
Watch realtime progress for that specific cluster
        ↓
Receive a diagnosis
        ↓
See it added to a per-cluster investigation history
```

The system now feels like a real AI-powered Kubernetes troubleshooting
product — one that works across every cluster the user has access to, not
just whichever one `kubectl` happens to point at.

## Definition of Done (checklist)

- [ ] Dashboard lists every cluster from `GET /clusters`, with the current context preselected
- [ ] Selecting a cluster updates the Investigate button label and is passed as `context` to `POST /investigate`
- [ ] Empty-kubeconfig state is handled with clear copy, not a blank screen
- [ ] Login gates cluster selection, investigation, and history
- [ ] Investigation progress updates in realtime via InsForge, scoped to the selected cluster
- [ ] Root Cause Card matches Prompt 03's `Diagnosis` shape exactly (no missing/renamed fields)
- [ ] History table shows cluster name per row and persists across page reloads
- [ ] No color is used anywhere in the UI — status/selection communicated via shape/weight/text only
- [ ] Prompts 01–03 functionality is unchanged
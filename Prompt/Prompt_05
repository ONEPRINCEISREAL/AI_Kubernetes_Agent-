# 05 — Integration, Testing & Deployment

## Context

The application is now feature-complete end-to-end:

```text
Login
        ↓
See all clusters from local kubeconfig
        ↓
Select a cluster
        ↓
Click "Investigate <cluster>"
        ↓
Backend investigates that specific cluster
        ↓
AI reasons about failures
        ↓
Root cause generated
        ↓
Suggested fix shown
        ↓
Investigation history saved (per cluster)
```

```text
Frontend
    ↓
FastAPI Backend (Orchestrator)
    ↓
Kubernetes Investigation Layer   (multi-cluster aware)
    ↓
AI Kubernetes Agent
    ↓
LLM Reasoning
(OpenRouter via InsForge Key)
    ↓
Root Cause + Suggested Fix
    ↓
InsForge
(Auth + History + Realtime)
    ↓
Frontend Diagnosis
```

Goal: prove this actually works against real Kubernetes failures, harden
error handling, and make it feel like a finished product — not a demo that
only works on the happy path.

---

## Goal

```text
End-to-End Integration (including multi-cluster switching)
Error Handling
Loading & Empty States
Real Kubernetes Failure Testing
```

Keep it beginner-friendly. No new features — this prompt is about making
the existing features reliable.

---

## Requirements

### 1. End-to-End Integration

Validate the full workflow exactly as a real user would experience it:

```text
User logs in
        ↓
Dashboard loads clusters from GET /clusters
        ↓
User selects a cluster
        ↓
User clicks Investigate
        ↓
Frontend sends POST /investigate { context }
        ↓
FastAPI orchestrates investigation for that context
        ↓
Kubernetes evidence collected
        ↓
AI reasoning triggered
        ↓
Root cause + fix generated
        ↓
History saved, tagged with the correct cluster
        ↓
Realtime UI updates
        ↓
User sees diagnosis
```

Explicitly verify **cluster isolation**: investigating `docker-desktop` and
then `minikube` must never mix evidence, diagnoses, or history rows between
the two. Run the flow twice in a row against two different contexts and
confirm the second run's results are entirely its own.

Fix any integration gaps found between Prompts 02–04 before moving on.

---

### 2. Improve Reliability

Add proper, user-facing error handling for every failure point introduced
across Prompts 02–04:

```text
kubectl failures
Cluster unreachable
Missing or invalid kubeconfig
Requested context no longer exists (e.g. removed since page load)
Zero contexts found in kubeconfig
OpenRouter / LLM failures
API timeout
No unhealthy resources found (healthy cluster)
Authentication issues
```

Show plain-language errors, never a raw stack trace or raw HTTP error body
in the UI. Examples:

```text
Unable to connect to Kubernetes cluster "minikube".
Please verify:
- kubeconfig path
- cluster access
- kubectl permissions
```

```text
The cluster "staging-cluster" is no longer available in your kubeconfig.
Refresh the cluster list and try again.
```

```text
The AI reasoning service is temporarily unavailable.
Your Kubernetes evidence was still collected — you can retry
diagnosis without re-running the full investigation.
```

That last case is worth calling out: if investigation evidence (Prompt 02)
succeeds but AI reasoning (Prompt 03) fails, don't discard the evidence —
surface a retry path for just the reasoning step rather than forcing a
full re-investigation.

---

### 3. Loading & Empty States

**On cluster list load:**
```text
Loading clusters from kubeconfig...
```

**On investigation start:**
```text
Investigating "docker-desktop"...
```

**During investigation** (realtime, per Prompt 04):
```text
✓ Checking Pods
✓ Reading Logs
✓ Analyzing Events
✓ Inspecting Deployments
✓ Checking Networking
✓ AI Reasoning
```

**If the cluster is healthy:**
```text
No critical Kubernetes issues detected in "docker-desktop".
Cluster appears healthy.
```

**If kubeconfig has zero contexts:**
```text
No clusters found. Check KUBECONFIG_PATH and try again.
```

---

### 4. Test Real Kubernetes Failures

Create intentional failure scenarios and confirm the system diagnoses them
correctly, end-to-end, through the actual dashboard (not just via curl).

#### Scenario 0 — Multi-cluster selection & switching (new)

Set up at least two local clusters/contexts (e.g. two `kind` or `minikube`
clusters, or `docker-desktop` plus one other).

```text
Setup: Cluster A healthy, Cluster B has a CrashLoopBackOff pod
Steps:
  1. Confirm both appear in GET /clusters
  2. Select Cluster A → investigate → expect "healthy" result
  3. Select Cluster B → investigate → expect CrashLoopBackOff diagnosis
  4. Confirm history shows two rows, each tagged to the correct cluster
Expected: no evidence or diagnosis leakage between clusters
```

#### Scenario 1 — CrashLoopBackOff

```text
Setup: Deployment missing a required environment variable
Expected Root Cause: Missing env variable
Expected Fix: Add missing secret/configmap value
```

#### Scenario 2 — ImagePullBackOff

```text
Setup: Deployment references a non-existent image tag
Expected Root Cause: Invalid image tag
Expected Fix: Update deployment image
```

#### Scenario 3 — OOMKilled

```text
Setup: Container memory limit set below actual usage
Expected Root Cause: Container exceeded memory limit
Expected Fix: Increase memory requests/limits
```

#### Scenario 4 — Service Selector Mismatch

```text
Setup: Service selector labels don't match pod labels
Expected Root Cause: Service selector does not match pod labels
Expected Fix: Update service selector
```

For each scenario, confirm: correct root cause, a fix that references the
real resource name, a confidence score consistent with the evidence
strength (per Prompt 03), and a correctly tagged history entry.

---

## Constraints

Do **NOT** add new product features in this prompt — this is integration,
reliability, and testing only, on top of Prompts 01–04.

Do **NOT** introduce color into the UI for error/warning states — continue
using the black-and-white system (e.g. a bordered box with an "!" glyph
for errors, not a red banner).

Do **NOT** break any previously working functionality — every fix here
should be additive (better error handling, better tests) not a rewrite of
working flows.

---

## Expected Result

The system reliably supports:

```text
Multiple local clusters, correctly isolated from each other
        ↓
Clear, plain-language errors at every failure point
        ↓
Honest loading/empty states instead of blank screens
        ↓
Verified-correct diagnoses across 4 real Kubernetes failure types
```

This is the point where the project stops being a demo and becomes a
system you'd trust to run against a real cluster.

## Definition of Done (checklist)

- [ ] Full E2E flow (login → select cluster → investigate → diagnosis → history) works for at least two different clusters
- [ ] No evidence/diagnosis/history leakage between clusters when switching
- [ ] Every failure mode listed above produces a plain-language message, never a raw stack trace
- [ ] A failed AI-reasoning step offers a retry that reuses already-collected evidence instead of forcing a full re-investigation
- [ ] Healthy-cluster case shows an explicit "no issues found" state, not an empty/broken UI
- [ ] Zero-context kubeconfig case is handled gracefully on the dashboard
- [ ] All 4 failure scenarios (CrashLoopBackOff, ImagePullBackOff, OOMKilled, selector mismatch) produce correct root cause + fix through the real dashboard
- [ ] No color introduced anywhere, including error states
- [ ] Prompts 01–04 functionality is fully intact
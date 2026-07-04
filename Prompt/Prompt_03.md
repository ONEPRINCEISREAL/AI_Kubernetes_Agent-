# 03 — AI Reasoning Engine

## Context

The Kubernetes Investigation Layer (Prompt 02) is complete and
**context-aware** — it can gather structured evidence (pods, logs, events,
deployments, networking) for any specific cluster found in the local
kubeconfig.

```text
Frontend
    ↓
FastAPI Backend
    ↓
Kubernetes Investigation Layer  (scoped to one cluster/context)
    ↓
AI Kubernetes Agent
        ↓
LLM Reasoning
(OpenRouter via InsForge Key)
        ↓
Root Cause Analysis
        ↓
Suggested Fix
```

Goal: make the system intelligent. The AI agent should behave like a
**Senior Kubernetes SRE** who:

1. Understands Kubernetes failure modes
2. Correlates pod status + logs + events + deployment state — not just
   summarizes them individually
3. Identifies a root cause
4. Suggests a concrete, actionable fix
5. States a confidence score with reasoning

Use the **OpenRouter API key provided via InsForge**. Never hardcode
secrets; read from environment variables only.

---

## Goal

Build the **AI Kubernetes Agent** as a distinct layer that consumes the
Prompt 02 investigation payload and produces a diagnosis:

```text
Prompt Builder
LLM Client
Root Cause Analyzer
Fix Recommendation Engine
Confidence Engine
```

---

## Requirements

### 1. Prompt Builder

Builds a single, structured troubleshooting prompt from the investigation
payload — not a dump of raw JSON.

System prompt persona: **Senior Kubernetes SRE**, precise and evidence-based,
who never guesses without stating the evidence behind a claim.

The user-turn content must include, clearly labeled:

```text
Cluster/context under investigation
Pod status (name, namespace, failure type, restart count)
Relevant log excerpts (already trimmed by Prompt 02)
Summarized events
Deployment health
Networking findings
Any collection errors reported by the investigation layer
```

Instruct the model to return exactly these sections, in this order, and
nothing else:

```text
1. Root Cause
2. Explanation
3. Suggested Fix
4. kubectl Commands
5. Prevention Recommendation
6. Confidence Score (0-100) with a one-line justification
```

Ask explicitly for **structured, deterministic** output (e.g. request JSON
matching a schema — see Root Cause Analyzer below) rather than free-form
prose, so the backend can parse it reliably instead of regex-scraping a
paragraph.

If the investigation payload reports `errors` (a component failed to
collect data), instruct the model to factor that into its confidence score
and say so explicitly rather than ignoring the gap.

---

### 2. LLM Client

Calls **OpenRouter** using the key sourced from InsForge.

Config (from `.env`, already scaffolded in Prompt 01):

```env
OPENROUTER_API_KEY=
OPENROUTER_MODEL=
```

Requirements:

- Use `HTTPX` (async client).
- Explicit timeout (e.g. 30s) — LLM calls are slower than kubectl calls.
- Retry transient failures (e.g. 429, 5xx) with capped exponential backoff
  (e.g. up to 3 attempts) — do not retry on 4xx client errors like bad
  auth.
- On final failure, raise a typed exception the API layer can translate
  into a clean error response — never let a raw HTTPX exception or stack
  trace reach the client.
- Log request duration, model used, and token usage (if the response
  includes it) — never log the API key or full prompt/response bodies at
  INFO level; use DEBUG for verbose payloads only, and ensure the key
  itself is never logged at any level.

---

### 3. Root Cause Analyzer

Parses the LLM's structured response into a typed model — this is the
contract between the AI layer and the API layer:

```python
class Diagnosis(BaseModel):
    context: str
    root_cause: str
    explanation: str
    suggested_fix: str
    kubectl_commands: list[str]
    prevention: str
    confidence: int  # 0-100
    confidence_reasoning: str
```

Example correlation the model should be capable of:

```json
// investigation evidence (input)
{
  "pods": { "status": "CrashLoopBackOff" },
  "logs": { "error": "DATABASE_URL missing" }
}
```

```text
// expected reasoning (output)
Root Cause: Application failed to start because the DATABASE_URL
environment variable is missing from the deployment spec.
```

The point of this component is **correlation, not transcription** — it
should never just echo the log line back as the root cause without
connecting it to the pod state and restart pattern.

If the LLM response doesn't parse into the `Diagnosis` schema, treat it as
a failure (log it, return a clean error) rather than silently passing
through malformed data.

---

### 4. Fix Recommendation Engine

Ensures `suggested_fix` and `kubectl_commands` are:

- **Practical** — a real command the user could actually run, not
  pseudocode
- **Beginner-friendly** — plain language explanation alongside the command
- **Kubernetes-specific** — tied to the actual resource names found during
  investigation (e.g. the real deployment name), not a generic template

Example:

```text
Suggested Fix:
Add the missing DATABASE_URL environment variable to the deployment spec.

kubectl Commands:
kubectl edit deployment payment-service -n default
```

---

### 5. Confidence Engine

Confidence is not decoration — it should reflect actual evidence strength.
Encourage (via the prompt) reasoning like:

```text
Confidence: 92%
Reasoning:
- Pod state = CrashLoopBackOff (strong signal)
- Logs explicitly show missing env var (direct evidence)
- Restart count confirms repeated startup failure (corroborating)
```

Lower confidence should correlate with thinner evidence — e.g. if the
investigation payload had `errors` (a component didn't collect data), the
model should reflect that uncertainty in a lower score, not report 90%+ on
partial evidence.

---

## FastAPI Integration

Update `POST /investigate` (from Prompt 02) so the full flow now runs:

```text
Collect Kubernetes Evidence (scoped to `context`)
    ↓
Send to AI Agent
    ↓
LLM Reasoning
    ↓
Root Cause Analysis
    ↓
Suggested Fix
    ↓
Return Diagnosis
```

Request is unchanged from Prompt 02:

```json
{ "context": "docker-desktop" }
```

Response now includes the diagnosis:

```json
{
  "status": "success",
  "context": "docker-desktop",
  "investigation": {
    "pods": {},
    "logs": {},
    "events": {},
    "deployments": {},
    "network": {}
  },
  "diagnosis": {
    "root_cause": "DATABASE_URL missing",
    "explanation": "Application cannot connect to the database on startup.",
    "suggested_fix": "Add the missing DATABASE_URL environment variable.",
    "kubectl_commands": ["kubectl edit deployment payment-service -n default"],
    "prevention": "Validate required env vars in CI before deploying.",
    "confidence": 92,
    "confidence_reasoning": "Pod state, logs, and restart count all agree."
  }
}
```

Keep both `investigation` (raw evidence) and `diagnosis` (AI output) in the
response — the frontend will want to show both.

---

## Constraints

Do **NOT** implement in this prompt:
- Authentication
- Investigation history / persistence
- Realtime updates
- Frontend changes
- Deployment/infra changes

Only build the AI reasoning layer on top of the existing investigation
layer. Keep it explainable — no hidden multi-agent chains, one clear
prompt → one clear structured response.

Do **NOT** break Prompt 01 or Prompt 02 functionality — `GET /clusters`
and the evidence-only shape of `POST /investigate` must continue to work;
this prompt extends the response, it doesn't replace the contract.

---

## Expected Result

```http
POST /investigate
{ "context": "docker-desktop" }
```

now returns real Kubernetes evidence **and** an AI-generated diagnosis in
one response — root cause, explanation, fix, exact kubectl commands,
prevention advice, and a confidence score grounded in the evidence
collected.

The backend now behaves like a **Senior Kubernetes SRE** helping
troubleshoot a specific cluster on request.

## Definition of Done (checklist)

- [ ] `POST /investigate` returns both `investigation` and `diagnosis` for the requested context
- [ ] LLM output is parsed into a typed `Diagnosis` model, not regex-scraped text
- [ ] Malformed/unparseable LLM responses fail cleanly instead of returning garbage
- [ ] OpenRouter key is read from env only, never logged, never hardcoded
- [ ] HTTPX client has a timeout and capped retries on transient failures only
- [ ] Confidence score visibly correlates with evidence strength (spot-check a few scenarios)
- [ ] `kubectl_commands` reference real resource names from the investigation, not placeholders
- [ ] Prompt 01 and Prompt 02 functionality still works unchanged
# 01 — Project Setup

## Context

We are building an **AI Kubernetes Troubleshooting Agent**.

### Architecture

```text
Frontend
    ↓
FastAPI Backend (Orchestrator)
    ↓
Kubernetes Investigation Layer
    ↓
AI Kubernetes Agent
    ↓
LLM Reasoning (OpenRouter via InsForge)
    ↓
Root Cause + Suggested Fix
    ↓
Frontend Diagnosis
```

### System type

This is an **on-demand troubleshooting system**, triggered manually by a user action — not a background service.

```text
User clicks "Investigate Cluster"
        ↓
API call
        ↓
Kubernetes investigation
        ↓
AI reasoning
        ↓
Diagnosis shown to user
```

> ⚠️ We are **NOT** building a Kubernetes controller/operator, admission webhook, or any component that watches/reconciles cluster state continuously. Everything runs synchronously, on request.

---

## Goal

Set up the project foundation **only**. This prompt establishes structure, tooling, and a working "hello world" loop end-to-end (frontend → backend → health check). No Kubernetes logic and no AI logic yet — those come in later prompts.

Deliverables:
- Working monorepo with backend + frontend
- Docker Compose that boots both services together
- One real, working endpoint (`/health`) wired frontend → backend
- Env var scaffolding (values empty, keys present)
- Clean placeholder modules for future logic, matching the target architecture

---

## Tech Stack

**Backend**
- Python 3.12+
- FastAPI
- Uvicorn (ASGI server)
- Pydantic v2 (schemas/config)
- Loguru (structured logging)
- HTTPX (outbound HTTP client, for later OpenRouter calls)

**Frontend**
- Next.js 14+ (App Router)
- TypeScript (strict mode)
- Tailwind CSS
- Axios (HTTP client)
- React Query (server state/caching)

**Infrastructure**
- Docker
- Docker Compose (v2 syntax, `docker compose`, not `docker-compose`)

---

## Project Structure

Monorepo layout:

```text
ai-kubernetes-agent/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   ├── core/
│   │   ├── kubernetes/
│   │   ├── ai/
│   │   ├── services/
│   │   └── models/
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .env.example
├── frontend/
│   ├── src/
│   │   ├── app/
│   │   ├── components/
│   │   ├── services/
│   │   ├── hooks/
│   │   └── types/
│   ├── package.json
│   ├── Dockerfile
│   └── .env.example
├── docs/
├── prompts/
├── docker-compose.yml
├── .gitignore
└── README.md
```

### Backend module responsibilities (placeholders only)

| Module | Future responsibility | This prompt |
|---|---|---|
| `api/` | Route definitions | Add only `health` route |
| `core/` | Config, logging, settings | Implement (needed now) |
| `kubernetes/` | Cluster inspection (pods, events, logs) | Stub functions, no logic |
| `ai/` | LLM reasoning / prompt orchestration | Stub functions, no logic |
| `services/` | Business logic orchestration | Empty, with `__init__.py` |
| `models/` | Pydantic schemas | Only `HealthResponse` model |

Stub function example (do not implement logic):

```python
def inspect_pods() -> None:
    """Placeholder — implemented in a later prompt."""
    pass
```

### Frontend module responsibilities (placeholders only)

| Module | Future responsibility | This prompt |
|---|---|---|
| `components/` | UI building blocks | Only the homepage + button |
| `services/` | API client wrappers | Only `healthCheck()` call |
| `hooks/` | React Query hooks | Only `useHealthCheck()` |
| `types/` | Shared TS types | Only `HealthResponse` type |

---

## Backend Requirements

### Endpoint

```text
GET /health
```

Response (200):

```json
{
  "status": "healthy",
  "service": "ai-kubernetes-agent"
}
```

Model it with a Pydantic schema, not a raw dict:

```python
class HealthResponse(BaseModel):
    status: str
    service: str
```

### Cross-cutting concerns

- **CORS**: allow the frontend origin (`http://localhost:3000`) via env-configurable allowed origins — don't hardcode `*` in a way that survives into later prompts.
- **Logging**: configure Loguru once in `core/logging.py`, imported by `main.py`. Log startup, shutdown, and each request at INFO level.
- **Config**: centralize env loading in `core/config.py` using `pydantic-settings`, not scattered `os.getenv()` calls.
- **Structure**: keep `main.py` thin — it should only create the app, mount routers, and register middleware.

---

## Frontend Requirements

### Homepage content

```text
AI Kubernetes Agent
Troubleshoot Kubernetes with AI

[ Investigate Cluster ]

System Status: Ready
```

The "Investigate Cluster" button should not yet trigger real investigation logic — for this prompt, wire it (or a visible status indicator on the page) to actually call `GET /health` on the backend via React Query, so we prove the full frontend → backend loop works. If the call succeeds, "System Status" reflects the live response instead of a hardcoded string.

### Design system — Black & White

Establish this now so every later screen (investigation results, diagnosis view, logs) inherits it automatically. No color other than black, white, and greys — no blue links, no colored buttons, no colored status badges.

**Palette (Tailwind config, `tailwind.config.ts`):**

```text
background:    #FFFFFF   (white)
foreground:    #0A0A0A   (near-black text)
surface:       #F5F5F5   (card / panel background)
border:        #E0E0E0   (dividers, outlines)
muted:         #6B6B6B   (secondary text)
inverse-bg:    #0A0A0A   (dark sections / primary button)
inverse-fg:    #FFFFFF   (text on dark sections)
```

**Component conventions:**
- Primary button: black background, white text, no border-radius greater than `rounded-md`, subtle scale/opacity change on hover — no color shift.
- Secondary/ghost button: white background, black border, black text.
- Status indicator ("Ready" / "Investigating" / "Error"): communicate state via **text + icon/shape + border weight**, never via color alone (e.g. a filled circle for "Ready", a dashed outline for "Pending"), since we have no color channel to lean on.
- Typography: one sans-serif font (e.g. Inter or system font stack), strong weight contrast (bold headings, regular body) to create hierarchy without color.
- Dark mode is not required for this prompt — the site is black-and-white by design, not by theme toggle.

Keep the homepage minimal, centered, generous whitespace — this is a diagnostic tool, not a marketing page.

---

## Environment Variables

**Backend** — `backend/.env.example`

```env
OPENROUTER_API_KEY=
OPENROUTER_MODEL=
KUBECONFIG_PATH=
ALLOWED_ORIGINS=http://localhost:3000
LOG_LEVEL=INFO
```

**Frontend** — `frontend/.env.example`

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

Commit `.env.example` files; do not commit real `.env` files (add both to `.gitignore`).

---

## Docker Requirements

- `backend/Dockerfile` — Python 3.12-slim base, installs `requirements.txt`, runs via `uvicorn app.main:app --host 0.0.0.0 --port 8000`.
- `frontend/Dockerfile` — Node LTS base, multi-stage build (install → build → run), serves on port 3000.
- `docker-compose.yml` at repo root:

```text
services:
  backend:
    build: ./backend
    ports: ["8000:8000"]
    env_file: ./backend/.env

  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    env_file: ./frontend/.env
    depends_on: [backend]
```

---

## Constraints

Do **NOT** implement in this prompt:
- `kubectl` / Kubernetes API logic
- AI reasoning or prompt construction
- OpenRouter or InsForge integration
- Authentication or authorization
- Realtime updates (WebSockets/SSE/polling loops)
- Any color in the UI beyond black/white/grey

Additional rules:
- **Do not break existing code** in future prompts — this foundation must remain stable as later prompts build on it.
- Keep code **beginner-friendly but production-style**: clear naming, typed functions, no magic strings, no dead code left behind "for later."
- Prefer explicit, small modules over clever abstractions at this stage.

---

## Expected Result

Running:

```bash
docker compose up --build
```

should give:

```text
http://localhost:3000   → Homepage renders, "System Status: Ready" reflects a live /health call
http://localhost:8000/health → {"status": "healthy", "service": "ai-kubernetes-agent"}
```

## Definition of Done (checklist)

- [ ] Repo structure matches the layout above
- [ ] `docker compose up --build` runs both services without errors
- [ ] Frontend homepage loads and displays live backend status (not hardcoded)
- [ ] `/health` returns the exact JSON shape specified, backed by a Pydantic model
- [ ] CORS, logging, and config are centralized, not ad-hoc
- [ ] Black-and-white design tokens are in `tailwind.config.ts` and used on the homepage
- [ ] `.env.example` files exist for both services; real `.env` files are gitignored
- [ ] No Kubernetes, AI, or auth logic present anywhere in the codebase yet
- [ ] `README.md` explains how to run the project locally and via Docker
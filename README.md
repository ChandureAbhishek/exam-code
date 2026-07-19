# Online Exam Platform (NEET / JEE / UPSC-style)

A horizontally-scalable, server-authoritative online examination system.
MCQ-first, extensible to MSQ / numerical / comprehension formats.

**Read [`ARCHITECTURE.md`](./ARCHITECTURE.md) first** — it explains how this scales
to millions of candidates and, honestly, what a demo can and cannot show.

Stack: **FastAPI (async) + Redis + Postgres** backend · **React 18 + Vite** frontend.

---

## Quick start (Docker — full stack)

```bash
# 1. Build and start everything (Postgres, Redis, 2 API pods, Nginx frontend)
docker compose up --build

# 2. In another terminal, seed demo users + a NEET mock exam
docker compose exec api python seed.py
```

Open **http://localhost:8080**

**Demo logins** (from `seed.py`):
- Candidate: `NEET0001` … `NEET0050`, password `test123`
- Admin: `admin` / `admin123`

---

## Quick start (no Docker — local dev)

You need Python 3.12+, Node 20+, and running Postgres + Redis (or use SQLite +
fakeredis via the test harness).

### Backend
```bash
cd backend
pip install -r requirements.txt
export DATABASE_URL="postgresql+asyncpg://exam:exam@localhost:5432/examdb"
export REDIS_URL="redis://localhost:6379/0"
python seed.py
uvicorn app.main:app --reload --port 8000
```
API docs (Swagger): http://localhost:8000/docs

### Frontend
```bash
cd frontend
npm install
VITE_API_BASE=http://localhost:8000 npm run dev
```
Open http://localhost:5173

---

## Run the test suite (no services needed)

Proves the whole flow — version guard, resume, scoring, idempotent submit —
against in-memory SQLite + fakeredis:

```bash
cd backend
pip install fakeredis lupa httpx aiosqlite
python test_flow.py    # expects: ALL TESTS PASSED
```

---

## Load testing

```bash
cd loadtest
pip install locust
locust -f locustfile.py --host http://localhost:8000
# open http://localhost:8089, set users + spawn rate (spawn rate = login-storm knob)
```

Seed more users first (edit the range in `seed.py`) for a meaningful test.

---

## Project layout

```
backend/
  app/
    main.py            FastAPI app + startup (launches flusher + sweeper)
    config.py          all tunables via env vars
    db.py              async SQLAlchemy engine/session
    redis_client.py    Redis pool + cluster-safe key naming
    models.py          users, exams, questions, attempts, answers, results
    schemas.py         request/response models (version field lives here)
    security.py        JWT + password hashing + admin guard
    routers/           auth, exam (candidate), admin
    services/
      attempts.py      THE HOT PATH — Redis-backed session, Lua answer save
      scoring.py       +4/-1 scoring from the Redis answer hash
      background.py    async flush to Postgres + auto-submit sweeper
  seed.py              demo data
  test_flow.py         end-to-end smoke test
frontend/
  src/
    api.js             API client + resilient AnswerSync (batch/retry/version)
    App.jsx            screen router
    pages/             Login, Dashboard, Exam, Result
    components/        Timer (server-synced), Palette (status grid)
docker-compose.yml     Postgres + Redis + 2 API pods + Nginx
ARCHITECTURE.md        scaling design + honest "what's not built" section
loadtest/locustfile.py candidate-behavior load test
```

---

## What's proven vs. what's production-simplified

See §10 of `ARCHITECTURE.md`. Short version: every core distributed-systems
pattern (buffered writes, idempotency, server-authoritative time, crash-resume,
async persistence, stateless pods) is **implemented and tested**. Proctoring,
encrypted paper distribution, biometric entry, and the virtual waiting room are
**deliberately not built** and are documented as such — they're specialist/
operational concerns, and pretending they're done would be misleading.

---

## Roles & portals

Four roles, each landing on its own portal after login. **Permissions are enforced
server-side** — the client routing is convenience, never security.

| Role | Reach | Power | Portal |
|---|---|---|---|
| **Candidate** | own attempt | take exam | exam interface |
| **Invigilator** | **one centre only** | grant time, force-submit, verify ID, log incidents | live hall console |
| **Support** | **national** | **no write access to exam state** — tickets and diagnostics only | helpdesk console |
| **Admin** | national | author exams, manage staff/centres, reopen attempts, read audit | control room |

### The separation-of-duties model

The real threat to a high-stakes exam is a **trusted insider**, not an outside
attacker. So power and reach are deliberately inverted:

- **Support has the widest reach and the least power.** An agent can look up any
  candidate nationally to diagnose a problem, but cannot change a deadline,
  status, or answer. They can only *request* extra time, which an invigilator or
  admin must approve. If a support account is compromised, the blast radius is
  "they read some metadata."
- **Invigilators are powerful but contained.** A Bengaluru invigilator gets a 403
  on any candidate seated in Delhi. Tested.
- **Nobody can read the answer key.** No endpoint returns `correct_option`.
  Support diagnostics return counts and timestamps only, so an agent can say
  "your last save was 3 seconds ago, 112 answers are recorded" without ever
  learning what was answered.
- **Every privileged action is audited**, and time grants / force-submits /
  reopens are rejected without a written reason of 10+ characters. The audit row
  is written in the *same transaction* as the change — if the audit fails, the
  change rolls back.
- **Suspension is immediate.** JWTs stay cryptographically valid until expiry, so
  staff requests re-check `active` against the DB. A suspended invigilator is
  locked out in seconds, not whenever their token happens to lapse.

### Demo logins (after `python seed.py`)

| Role | Login | Password | Scope |
|---|---|---|---|
| Admin | `admin` | `admin123` | national |
| Invigilator | `inv_blr` | `invig123` | BLR01 — candidates 1–30 |
| Invigilator | `inv_del` | `invig123` | DEL01 — candidates 31–50 |
| Support | `support1` | `support123` | national, read-only |
| Candidate | `NEET0001`…`NEET0050` | `test123` | every 10th has +20 min compensatory time |

**Try the security model yourself:** log in as `inv_del`, and try to grant time to
a candidate numbered 1–30. You'll get a 403.

### Run the RBAC test suite

```bash
cd backend && python test_rbac.py    # 15 boundary assertions, expects ALL RBAC TESTS PASSED
```

Every assertion is an attack that must fail: cross-centre grants, support
privilege escalation, answer-key leakage, reason-less time grants, and acting
with a valid token after suspension.

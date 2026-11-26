# Simplething: Project Specification (v1.0)

> Status: **approved** (2026-11-26). Changes after this point go through a Pull Request.

## 1. Why this project exists

Simplething is a learning project with three goals:

1. **Learn Claude Code**: plan, write, test and review code with an AI assistant.
2. **Learn security**: build security in from day one (secure coding, automated scans, secrets management).
3. **Learn microservices**: split an app into small independent services that talk to each other.

The product itself is kept small on purpose, so the focus stays on *how* it is built.

## 2. The product: a small task manager

Users create an account, log in, and manage a personal to-do list.

### User stories (what a user can do)

| ID | As a user, I can… | Priority |
|----|-------------------|----------|
| US-1 | create an account with an email and a password | Must |
| US-2 | log in and get access to my own data | Must |
| US-3 | create a task (title, optional description, optional due date) | Must |
| US-4 | list my tasks | Must |
| US-5 | edit a task / mark it as done | Must |
| US-6 | delete a task | Must |
| US-7 | **never** see or change another user's tasks | Must (security) |
| US-8 | get a notification (logged/emailed) when a task is due | Could (phase 2) |

### Out of scope (v1)
Sharing tasks, teams, web or mobile interface (API only), payments, password reset by email.

## 3. Architecture (microservices)

```
                 ┌─────────────────┐
  user / tests ─▶│   API Gateway   │  single entry point, HTTPS, rate limiting
                 └───────┬─────────┘
             ┌───────────┴───────────┐
             ▼                       ▼
     ┌───────────────┐       ┌───────────────┐
     │ auth-service  │       │ tasks-service │
     │ accounts,     │       │ CRUD on tasks │
     │ login, tokens │       │ checks token  │
     └──────┬────────┘       └──────┬────────┘
            ▼                       ▼
      ┌──────────┐            ┌──────────┐
      │ users DB │            │ tasks DB │      one database per service
      └──────────┘            └──────────┘
                                    │ (phase 2) "task due" event
                                    ▼
                          ┌──────────────────────┐
                          │ notification-service │
                          └──────────────────────┘
```

| Service | Responsibility | Owns data |
|---------|----------------|-----------|
| gateway | Routes `/auth/*` → auth-service, `/tasks/*` → tasks-service; rate limiting | none |
| auth-service | Register, login, issue signed tokens (JWT) | users |
| tasks-service | Create/read/update/delete tasks for the logged-in user | tasks |
| notification-service (phase 2) | Listen for events and send notifications | none / logs |

Key microservice rules we will follow:
- Each service has **its own code folder, its own tests, its own database**.
- Services talk only through **HTTP APIs** (and later a message queue), never by reading each other's database.
- Every service runs in its own **container** (Docker); the whole system starts with one command.

## 4. API (first version)

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| POST | /auth/register | anyone | create account |
| POST | /auth/login | anyone | returns an access token |
| GET | /tasks | logged in | list my tasks |
| POST | /tasks | logged in | create a task |
| GET | /tasks/{id} | owner | get one task |
| PATCH | /tasks/{id} | owner | update / mark done |
| DELETE | /tasks/{id} | owner | delete |
| GET | /health | anyone | each service reports it is alive |

## 5. Technology choices (approved)

| Topic | Recommendation | Why (for a beginner) |
|-------|----------------|----------------------|
| Language | **Python 3.13** | Most readable language, huge community |
| Web framework | **FastAPI** | Little code, automatic API docs page, built-in input validation |
| Database | **PostgreSQL** (SQLite for local tests) | Industry standard, free |
| Containers | **Docker + Docker Compose** | Run all services with `docker compose up` |
| Gateway | **Traefik** or **Nginx** | Simple config, no code needed |
| Messaging (phase 2) | **Redis** or **RabbitMQ** | Easy introduction to async events |
| CI | **GitHub Actions** | Repo is public on GitHub: Actions and CodeQL are free |
| Dev environment | **GitHub Codespaces** | Docker and Python preinstalled, nothing to set up locally |

Alternatives if you prefer: Node.js + Express/NestJS (TypeScript), or Go. Mixing languages between services is possible (a microservice benefit) but harder to learn, so not recommended for v1.

## 6. Security requirements

| ID | Requirement |
|----|-------------|
| SEC-1 | Passwords stored only as strong hashes (argon2 / bcrypt), never in clear |
| SEC-2 | Access tokens are signed, expire (e.g. 30 min), are checked on every task request |
| SEC-3 | Every task query is filtered by owner: no access to other users' data (tested!) |
| SEC-4 | All input validated (types, lengths); errors don't leak internal details |
| SEC-5 | No secrets in the code or in Git: use environment variables / GitHub Secrets |
| SEC-6 | Dependencies scanned for known vulnerabilities on every change |
| SEC-7 | Container images scanned; services run as non-root users |
| SEC-8 | Rate limiting on login to slow down password guessing |
| SEC-9 | Security headers and HTTPS at the gateway |
| SEC-10 | `main` branch protected: changes only via Pull Request with green CI |

Reference: OWASP Top 10 and OWASP API Security Top 10.

## 7. Quality & testing strategy

| Test type | What it checks | Tool (Python) |
|-----------|----------------|---------------|
| Lint / format | Code style, simple mistakes | ruff |
| Type check | Wrong data types | mypy |
| Unit tests | Small functions in isolation | pytest |
| Functional / API tests | Each endpoint behaves as the user stories say | pytest + httpx |
| Integration tests | Services work together (via docker compose) | pytest against running stack |
| Non-regression | The whole suite runs on every change; a bug fix always adds a test | CI + branch protection |
| Coverage | % of code exercised by tests (target ≥ 80%) | pytest-cov |
| SAST (code security) | Dangerous code patterns | Bandit, CodeQL / Semgrep |
| Dependency scan (SCA) | Known-vulnerable libraries | pip-audit, Dependabot |
| Secret scan | Passwords/keys committed by mistake | gitleaks |
| Container scan | Vulnerable OS packages in images | Trivy |
| DAST (live attack test) | Scans the running API for common flaws | OWASP ZAP baseline |

## 8. CI pipeline (runs on every Pull Request)

```
1. Lint + type check  ─┐
2. Secret scan         ├─ fast checks (parallel)
3. SAST + dependency   ─┘
4. Unit + functional tests per service (+ coverage)
5. Build Docker images → Trivy scan
6. Start full stack with docker compose → integration tests → ZAP scan
7. All green → PR can be merged into main
```

## 9. Roadmap

| Phase | Goal | Deliverable |
|-------|------|-------------|
| 0 | Validate this spec & tech choices | SPEC v1.0 approved |
| 1 | Repo skeleton + CI with all tools (on a "hello world" service) | Green pipeline |
| 2 | auth-service (US-1, US-2) + tests | Register/login working |
| 3 | tasks-service (US-3 → US-7) + tests | Task CRUD with ownership checks |
| 4 | Gateway + docker compose + integration & DAST tests | Full system runs with one command |
| 5 | (optional) notification-service + message queue | Async events |

Each phase = one or more small Pull Requests, reviewed before merge.

## 10. Decisions log

| Date | Decision |
|------|----------|
| 2026-11-26 | Product: task manager, scope as above |
| 2026-11-26 | Stack: Python + FastAPI, PostgreSQL, Docker Compose |
| 2026-11-26 | Repo: public on GitHub, developed in Codespaces, CI on GitHub Actions |
| 2026-11-26 | v1 is API only (no web page) |

## Glossary

- **API**: a set of URLs a program calls to ask another program to do something.
- **Microservice**: a small, independent program doing one job, with its own data.
- **Container (Docker)**: a package holding a program and everything it needs to run, identical everywhere.
- **CI (Continuous Integration)**: robots that test every change automatically before it's accepted.
- **JWT / token**: a signed "badge" proving who you are, sent with each request.
- **Pull Request (PR)**: a proposed change, reviewed and tested before joining `main`.
- **SAST / DAST / SCA**: security scans of source code / of the running app / of third-party libraries.

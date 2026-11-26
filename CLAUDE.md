# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Simplething is a **learning project**: a small task-manager REST API (API only, no UI) built as microservices, to learn Claude Code, application security, and microservice architecture. `docs/SPEC.md` is the source of truth (user stories `US-*`, security requirements `SEC-*`, roadmap phases); read it before starting any work and reference those IDs in code, tests and PRs.

Current state: phase 0 done (spec approved). No code yet. Next is phase 1 (CI + security tooling on a hello-world service).

## Working with the user

The user is a beginner at coding and wants to understand each step, not just get results.
- Work one roadmap step at a time; explain what changed and why in plain language, then wait for their OK before the next step.
- Changes go through a branch + Pull Request; never commit directly to `main`.

## Stack

Python 3.13, FastAPI, PostgreSQL (SQLite allowed in unit tests), Docker + Docker Compose, gateway via Traefik or Nginx, GitHub Actions CI. Development happens in GitHub Codespaces (Docker preinstalled). Repo is public on GitHub.

## Architecture rules

- Services: `gateway` (routing, rate limiting, HTTPS/security headers), `auth-service` (register/login, issues expiring signed JWTs), `tasks-service` (task CRUD, validates JWT), and later `notification-service` (consumes events from a queue).
- Each service is independent: its own folder, dependencies, tests, Dockerfile and **its own database**. Services never read another service's database; they communicate only over HTTP APIs (later: message queue).
- Every service exposes `GET /health`.
- Every tasks query must be filtered by the token's user (SEC-3); cross-user access must have tests proving it is denied.
- Secrets come from environment variables / GitHub Secrets only. Containers run as non-root.

## Quality gates (CI on every PR)

ruff (lint/format), mypy, pytest + pytest-cov (≥ 80% coverage), Bandit, CodeQL, pip-audit, Dependabot, gitleaks, Trivy (images), and from phase 4 docker-compose integration tests + OWASP ZAP baseline. Every bug fix adds a regression test.

## Commands

Not defined yet: add build, lint, test (including running a single test) and `docker compose` commands here as phase 1 creates them.

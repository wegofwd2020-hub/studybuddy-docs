# Code Quality & Security Tooling — Impact Summary

**Date added:** 2026-03-28
**Branch:** feat/round-2-dev

---

## Before This Work

The project had no automated guardrails. Code could be committed with:
- Security vulnerabilities nobody would catch until production
- Known CVEs sitting in dependencies with no alert
- Bugs introduced silently (no lint, no type enforcement in CI)
- Any developer could push broken, unformatted, or insecure code and it would merge cleanly

---

## The Four Layers of Protection Now in Place

### 1. Before You Even Commit — Pre-commit Hooks

When a developer runs `pre-commit install`, every `git commit` now automatically:
- Checks for **accidentally committed secrets** (API keys, passwords, tokens) and blocks the commit
- Runs **ruff** to enforce consistent code style and catch obvious bugs
- Runs **bandit** to flag security anti-patterns in Python
- Runs **eslint + prettier** to enforce frontend code quality

**Impact:** Bad code never reaches the repo in the first place. No code review needed to catch formatting or obvious security issues.

---

### 2. On Every Pull Request — CI Gates

Four jobs now run automatically on every PR to `main`:

```
backend-lint  ──────────► backend-test
                          (only runs if lint passes)

frontend-lint ──────────► frontend-test
                          (only runs if lint passes)
```

| Job | Tools | What it catches |
|---|---|---|
| `backend-lint` | Ruff, Bandit, pip-audit | Style violations, undefined vars, security patterns, CVEs in Python deps |
| `backend-test` | pytest + coverage | Regressions; fails if coverage drops below 70% |
| `frontend-lint` | ESLint, Prettier, tsc, npm audit | Type errors, style violations, CVEs in JS deps |
| `frontend-test` | Vitest | Frontend unit test regressions |

**Impact:** A PR with a security vulnerability, broken dependency, or type error **cannot merge**. No human reviewer needs to catch these — the pipeline does it automatically.

---

### 3. Every Week — Dependabot

Every Monday morning, GitHub automatically opens PRs to update outdated dependencies across:
- Backend Python packages (`/backend`)
- Pipeline Python packages (`/pipeline`)
- Frontend JavaScript packages (`/web`)
- GitHub Actions versions

Grouped to reduce noise (e.g. all `@types/*` in one PR, all testing tools in one PR).

**Impact:** You find out about security patches and upgrades through a controlled PR — not by discovering you've been running a vulnerable library for 6 months.

---

### 4. The Bugs It Found Immediately

Running ruff and bandit against the existing codebase **immediately caught real issues**:

| Bug | File | Risk if Undetected |
|---|---|---|
| `log` variable used in 15+ functions but never defined at module level | `src/auth/tasks.py` | Celery worker would crash on any auth task that hit an error path, silently swallowing exceptions |
| `subject_breakdown` always returned `[]` | `src/analytics/router.py` | Student dashboard would never show subject breakdown data — a silent data bug |
| `subj_rows` fetched from DB on every request but immediately discarded | `src/analytics/router.py` | Wasted DB query on every student stats call |
| Unused `pool`, `first_pass_count`, `first_att` variables | `src/content/router.py`, `src/reports/service.py` | Dead code from a prior refactor — indicates incomplete work |

These bugs existed in the codebase before today and would only have been discovered during runtime testing or user reports. Ruff caught them in seconds.

---

## Files Added / Modified

| File | Purpose |
|---|---|
| `backend/pyproject.toml` | Ruff and Bandit configuration |
| `backend/requirements.txt` | Added `ruff`, `bandit`, `pip-audit`, `pytest-cov` |
| `.github/workflows/test.yml` | CI rewritten with 4 jobs and quality/security gates |
| `.github/dependabot.yml` | Weekly automated dependency update PRs |
| `.pre-commit-config.yaml` | Pre-commit hooks for all quality and security checks |
| `.secrets.baseline` | detect-secrets baseline (no committed secrets) |
| `web/package.json` | All `^` semver ranges replaced with exact pinned versions |

---

## The Bottom Line

| | Before | After |
|---|---|---|
| Security vuln in code | Found in production | Blocked at commit |
| CVE in a dependency | Unknown until breach | Weekly automated PR |
| Bad code merges to main | Possible | Blocked by CI gate |
| Coverage drops silently | Yes | Fails CI at < 70% |
| New developer commits | No standards enforced | Pre-commit enforces automatically |

The cost of fixing a bug **before commit** is minutes. The cost of fixing it **after it reaches production** — especially a security vulnerability in an app handling student data — is orders of magnitude higher.

---

## Developer Setup

To activate pre-commit hooks locally:

```bash
pip install pre-commit
pre-commit install
```

To run all hooks manually against every file:

```bash
pre-commit run --all-files
```

To run ruff and bandit directly:

```bash
cd backend
python -m ruff check src/
python -m bandit -c pyproject.toml -r src/ -ll
```

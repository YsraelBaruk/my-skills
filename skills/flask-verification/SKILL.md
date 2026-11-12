---
name: flask-verification
description: Run the full Flask verification loop — environment and app-factory check, ruff/black/mypy linting, config and route sanity via the flask CLI, pytest with the test client and CLI runner, pip-audit and bandit security scans, logging and error-handler review, deployment readiness, and diff review — producing a phased pass/fail report before release or PR. Use when preparing a Flask pull request, validating coverage or app configuration, or running pre-deploy readiness checks for a Flask service.
metadata:
  origin: ECC
---

# Flask Verification Loop

Run before PRs, after major changes, and pre-deploy to ensure Flask application quality,
configuration correctness, and security. Targets Flask 3.x (3.1+ for `TRUSTED_HOSTS`,
`SECRET_KEY_FALLBACKS`, `SESSION_COOKIE_PARTITIONED`, per-request `max_content_length`).

## When to Activate

- Before opening a pull request for a Flask project
- After changing the application factory, config, blueprints, CLI commands, or error handlers
- After dependency upgrades or Python version bumps
- Pre-deployment verification for staging or production
- Running the full environment → lint → test → security → readiness pipeline
- Validating test coverage or `TESTING` configuration

## Phase 1: Environment Check

```bash
# Verify Python version (Flask 3.1 requires 3.9+)
python --version

# Confirm virtualenv and dependency source of truth
which python
python -m pip list --outdated
python -m pip check          # dependency conflicts

# App discovery: factory or module, and entrypoint env files
flask --app 'myapp:create_app' --help
cat .flaskenv .env 2>/dev/null   # should not contain production secrets
```

If `flask --app ... --help` cannot import the app, stop and fix the factory/imports first —
every later phase depends on it.

## Phase 2: Code Quality & Formatting

```bash
ruff check . --fix
black . --check
isort . --check-only
mypy . --config-file pyproject.toml   # or: pyright
```

Report: type errors, formatting drift, unused imports, `print()` left in library code
(use `current_app.logger` instead).

## Phase 3: App, Config, and Route Sanity

Routes and blueprint wiring:

```bash
flask --app 'myapp:create_app' routes            # duplicate/conflicting rules
flask --app 'myapp:create_app' shell <<'EOF'
from myapp import create_app
app = create_app()
print(len(list(app.url_map.iter_rules())), "routes")
for rule in app.url_map.iter_rules():
    print(rule, sorted(rule.methods - {"HEAD", "OPTIONS"}))
EOF
```

Config review (run inside `app.app_context()` — Flask config is not global):

```python
from myapp import create_app

app = create_app()
c = app.config

checks = {
    "DEBUG is False": c["DEBUG"] is False,
    "TESTING is False": c["TESTING"] is False,
    "SECRET_KEY set and >= 32 chars": bool(c.get("SECRET_KEY")) and len(c["SECRET_KEY"]) >= 32,
    "SECRET_KEY not a dev default": c.get("SECRET_KEY") not in {"dev", "changeme", "secret"},
    "SESSION_COOKIE_SECURE": c.get("SESSION_COOKIE_SECURE") is True,
    "SESSION_COOKIE_HTTPONLY": c.get("SESSION_COOKIE_HTTPONLY") is True,
    "SESSION_COOKIE_SAMESITE": c.get("SESSION_COOKIE_SAMESITE") in {"Lax", "Strict"},
    "TRUSTED_HOSTS configured": bool(c.get("TRUSTED_HOSTS")),
    "MAX_CONTENT_LENGTH set": c.get("MAX_CONTENT_LENGTH") is not None,
    "PREFERRED_URL_SCHEME https": c.get("PREFERRED_URL_SCHEME") == "https",
}
for name, ok in checks.items():
    print(("✓" if ok else "✗"), name)
```

Notes:
- Enable debug with `flask --app ... run --debug`, not by setting `DEBUG` in config.
- Load secrets with `app.config.from_prefixed_env()` (e.g. `FLASK_SECRET_KEY`) or a
  `from_envvar`/`from_file` path; never commit `.env` with real values.
- Behind a proxy, `ProxyFix` must be applied (Phase 9) or `url_for(_external=True)` and
  redirects will use the wrong scheme/host.

## Phase 4: Tests + Coverage

Run the suite with the test client and CLI runner — both are Flask-native and need no
live server:

```bash
pytest -q --cov=myapp --cov-report=term-missing --cov-report=xml \
       --cov-fail-under=80
```

Minimum fixtures (application factory):

```python
# tests/conftest.py
import pytest
from myapp import create_app

@pytest.fixture()
def app():
    app = create_app({"TESTING": True, "SECRET_KEY": "test"})
    yield app

@pytest.fixture()
def client(app):
    return app.test_client()

@pytest.fixture()
def runner(app):
    return app.test_cli_runner()
```

Required coverage of Flask-specific surfaces:

```python
def test_create_user(client):
    resp = client.post("/users", json={"email": "a@example.com"})
    assert resp.status_code == 201
    assert resp.get_json()["email"] == "a@example.com"

def test_missing_user_returns_json_error(client):
    resp = client.get("/users/999")
    assert resp.status_code == 404
    assert resp.is_json

def test_cli_seed(runner):
    result = runner.invoke(args=["seed"])
    assert result.exit_code == 0

def test_app_context_resource(app):
    with app.app_context():
        assert get_db() is not None   # teardown must release it
```

Report: total passed/failed/skipped, overall and per-package coverage.

| Component | Target |
|-----------|--------|
| Views / blueprints | 85%+ |
| Services | 90%+ |
| Error handlers | 100% of registered codes |
| CLI commands | 80%+ |
| Overall | 80%+ |

Traps to check:
- Tests that only pass with a global `app` import instead of the factory
- Requests made outside an app context, or `current_app` accessed in module scope
- `TESTING=True` missing, so exceptions propagate as 500 instead of surfacing in tests
- Untested `errorhandler` branches and `before_request`/`teardown_appcontext` paths

## Phase 5: Security Scan

```bash
pip-audit
bandit -r . -f json -o bandit-report.json
gitleaks detect --source . --verbose   # if installed

# Secret/config greps
grep -rnE "SECRET_KEY\s*=\s*['\"][^'\"]+" --include="*.py" .
grep -rnE "app\.run\(.*debug\s*=\s*True" --include="*.py" .
grep -rnE "password|api_key|secret" .env* 2>/dev/null
```

Check in code:
- `str(e)` or tracebacks returned in responses (leaks internals; log instead)
- `send_file`/`send_from_directory` with user-controlled paths (path traversal)
- `render_template_string` with user input (SSTI) and `|safe` on request data (XSS)
- CORS wildcard combined with credentialed sessions
- `TRUSTED_HOSTS` unset (Host-header attacks)
- Session cookies without `Secure`/`HttpOnly`/`SameSite`
- Unbounded uploads (`MAX_CONTENT_LENGTH`, `MAX_FORM_MEMORY_SIZE`, `MAX_FORM_PARTS`)

## Phase 6: Logging & Error Handling Review

```python
# Configure logging before creating the app
from logging.config import dictConfig

dictConfig({
    "version": 1,
    "formatters": {"default": {
        "format": "[%(asctime)s] %(levelname)s in %(module)s: %(message)s",
    }},
    "handlers": {"wsgi": {
        "class": "logging.StreamHandler",
        "stream": "ext://flask.logging.wsgi_errors_stream",
        "formatter": "default",
    }},
    "root": {"level": "INFO", "handlers": ["wsgi"]},
})
```

Verify:
- Errors raised in views are logged with `app.logger.exception(...)` where they are caught
- Registered handlers return meaningful status codes and content types:

```bash
flask --app 'myapp:create_app' shell <<'EOF'
from myapp import create_app
c = create_app({"TESTING": True}).test_client()
for path in ("/does-not-exist", "/boom"):
    r = c.get(path)
    print(path, r.status_code, r.content_type)
EOF
```

- 404/405 return your JSON (API) or template (HTML), not Flask defaults, if the service
  has a unified error contract
- No `print()` debugging or `app.logger.debug` without a level gate in hot paths

## Phase 7: Performance Smoke Check

```bash
# Route count and slow endpoints; run against a local dev server
flask --app 'myapp:create_app' run --port 5001 &
python - <<'EOF'
import time, urllib.request
for path in ("/", "/health"):
    start = time.perf_counter()
    with urllib.request.urlopen(f"http://127.0.0.1:5001{path}") as r:
        body = r.read()
    print(f"{path}: {r.status} {time.perf_counter()-start:.3f}s {len(body)}B")
EOF
kill %1
```

Check:
- Query count per request (`SQLAlchemy` event listener or extension toolbar) — N+1 patterns
- Missing indexes on filter/sort columns
- Caching used for expensive read paths, with invalidation defined
- Blocking I/O in request handlers where a task queue is expected

## Phase 8: Deployment Readiness

```bash
# Never deploy with the dev server
grep -rn "app.run(" --include="*.py" .          # should be dev-only / absent

# Production server smoke run (pick one installed)
gunicorn -w 4 -b 127.0.0.1:8000 'myapp:create_app()' --timeout 30
waitress-serve --listen=127.0.0.1:8000 --call myapp:create_app

# Health endpoint
curl -fsS http://127.0.0.1:8000/health
```

Checklist:
- [ ] `DEBUG=False`, `TESTING=False`, no reloader in production
- [ ] `SECRET_KEY` from the environment; `SECRET_KEY_FALLBACKS` set during rotation
- [ ] `ProxyFix` configured for the exact trusted proxy hop count, or proxy headers verified
- [ ] `TRUSTED_HOSTS` set to real hostnames
- [ ] `SESSION_COOKIE_SECURE/HTTPONLY/SAMESITE` set; `PREFERRED_URL_SCHEME=https`
- [ ] Upload and form limits configured
- [ ] Database migrations applied (`flask db upgrade` with Flask-Migrate/Alembic)
- [ ] Static files served by the proxy/CDN, not by Flask in production
- [ ] Health/readiness endpoint exists and checks dependencies
- [ ] Workers/timeouts sized; graceful shutdown verified
- [ ] Logs shipped and error monitoring (Sentry, etc.) initialized in the factory

## Phase 9: Diff Review

```bash
git diff --stat
git diff
git diff | grep -nE "print\(|breakpoint\(|pdb|debug=True|TODO|FIXME"
git status --short
```

Checklist:
- No debug statements, `app.run()`, or `debug=True` left behind
- Config/env changes documented; no secrets in the diff
- Migrations included for model changes
- New routes registered on a blueprint and covered by a test
- Error handlers cover new failure modes
- README/`.env.example` updated for new variables

## Output Template

```
FLASK VERIFICATION REPORT
=========================
Phase 1 Environment   [PASS/FAIL]  Python X.Y, deps consistent
Phase 2 Quality       [PASS/FAIL]  ruff/black/isort/mypy
Phase 3 App & Config  [PASS/FAIL]  N routes, config checks x/y
Phase 4 Tests         [PASS/FAIL]  X passed, Y failed, Z% coverage
Phase 5 Security      [PASS/FAIL]  CVEs: N, bandit: M, secrets: K
Phase 6 Logging/Errors[PASS/FAIL]  handlers verified
Phase 7 Performance   [PASS/FAIL]  p95 < target, no N+1
Phase 8 Deployment    [PASS/FAIL]  prod server + checklist
Phase 9 Diff          [PASS/FAIL]  X files changed, +A/-D

Overall: [READY / NOT READY]

Issues to Fix:
1. ...
2. ...

Unresolved Risks:
- ...
```

## CI Example (GitHub Actions)

```yaml
name: Flask Verification
on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready --health-interval 10s
          --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r requirements.txt
      - run: pip install ruff black isort mypy pytest pytest-cov bandit pip-audit
      - run: ruff check . && black . --check && isort . --check-only && mypy .
      - run: pip-audit && bandit -r . -q
      - env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          FLASK_SECRET_KEY: test-secret-key-for-ci-only
        run: pytest --cov=myapp --cov-report=xml --cov-fail-under=80
```

## Quick Reference

| Check | Command |
|-------|---------|
| App import | `flask --app 'myapp:create_app' --help` |
| Routes | `flask --app 'myapp:create_app' routes` |
| Shell / config | `flask --app 'myapp:create_app' shell` |
| Lint | `ruff check .` |
| Format | `black . --check` |
| Types | `mypy .` |
| Tests | `pytest --cov=myapp` |
| Security | `pip-audit && bandit -r .` |
| Prod server | `gunicorn -w 4 'myapp:create_app()'` |
| Migrations | `flask db upgrade` |
| Diff | `git diff --stat` |

Remember: automated checks catch the mechanical failures. Verify at least one real request
path end to end against a production-style server before declaring READY.

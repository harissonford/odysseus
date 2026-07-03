# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Odysseus is a self-hosted AI workspace (FastAPI backend + vanilla-JS frontend) covering chat/agents, deep research, local-model serving ("Cookbook"), documents, email, notes/calendar, and more. Single-process by convention. Python 3.11+; the Docker image runs on `python:3.14-slim`.

## Branch model (important for PRs)

- **`dev`** is the default branch — all PRs land here. The current working branch may be a feature branch off `dev`.
- **`main`** is the curated/stable branch users run; it is fast-forwarded from a stable `dev` commit at release. Do **not** target `main` with PRs.
- Commits use [Conventional Commits](https://www.conventionalcommits.org): `type(scope): summary` (`fix`, `feat`, `refactor`, `docs`, `test`, `chore`, `ci`).

## Commands

If a project venv exists, prefer it (`./venv/bin/python`) so commands don't fall back to system Python. There is no committed venv — create one (`python -m venv venv && ./venv/bin/pip install -r requirements.txt`) before running locally, or use Docker (below) which needs no local venv.

```bash
# Run the app (manual dev)
python -m uvicorn app:app --host 127.0.0.1 --port 7000   # http://localhost:7000

# Run the app (Docker, recommended for full-feature testing)
cp .env.example .env && docker compose up -d --build
docker compose logs --tail=120 odysseus     # first admin password is printed here on first boot

# Tests
python -m pytest                              # full suite
python -m pytest tests/test_<name>.py         # single file
python -m pytest tests/test_<name>.py::test_x # single test
python -m pytest -m area_security             # taxonomy slice (areas: security, routes, services, cli, js, helpers, unit, uncategorized)
python -m pytest -m "area_services and sub_cookbook"
./venv/bin/python tests/run_focus.py --area services --sub-area cookbook   # focused runner (validates names, --last-failed, --fast, --durations)
python -m pytest -m "not slow"                # fast lane — skips tests tagged @pytest.mark.slow (run_focus --fast does the same)

# Syntax checks (mirror CI — fast, no deps needed)
python -m compileall -q app.py core routes src services scripts tests
node --check static/app.js                    # or static/js/<file>.js
```

Test environment: `tests/conftest.py` defaults `DATABASE_URL` to `sqlite:///:memory:` (so the suite never writes `data/app.db`) and stubs heavy optional deps with `MagicMock` when they aren't installed — tests must not assume a real DB file or all deps present. The `area_*`/`sub_*` marker taxonomy is documented in `tests/README.md` and `tests/TESTING_STANDARD.md`; mark a test `slow` only with duration evidence (`run_focus.py --durations`).

CI (`.github/workflows/ci.yml`) runs three jobs: `compileall` (Python syntax), `node --check` over `static/app.js` + `static/js/**/*.js`, and pytest (skipped on docs-only PRs; CI creates `./data` first). The pytest job is **`continue-on-error` (informational)** — the suite has known flaky/environment-dependent failures, so a red pytest check does not block merge, but don't add to the flakiness.

Separately, several **security workflows run on every PR and are blocking** (unlike pytest): `secret-scan`, `dependency-review`, `workflow-security`, and `container-trivy` (skips doc-only changes). GitHub CodeQL also scans pushes — much of the recent commit history is clearing its path-traversal/parser alerts, so treat sanitizing user-controlled paths and parser inputs as a first-class review concern, not an afterthought.

## Architecture

### Backend request flow
`app.py` is a slim orchestrator: it configures logging/MIME/middleware, builds the FastAPI app, then **imports each `routes/*_routes.py` module and calls its `setup_*_routes(...)` factory**, passing in shared manager/handler instances and `include_router`-ing the result. New HTTP surface = a new `routes/<area>_routes.py` exposing a `setup_<area>_routes(deps) -> APIRouter`, wired into `app.py`.

`src/app_initializer.py::initialize_managers()` constructs the shared singletons (memory, sessions, RAG, chat/research handlers, model discovery, presets, etc.) and returns them in a dict that `app.py` distributes to route factories. This is the dependency-injection seam — add new long-lived managers here rather than constructing them in route modules.

### Layering
- `routes/` — HTTP/API endpoints (thin). `*_helpers.py` siblings hold per-area request logic.
- `services/` — service-layer subsystems, each a package (`research/`, `search/`, `shell/`, `tts/`, `stt/`, `memory/`, `hwfit/`, `docs/`, `faces/`, `youtube/`).
- `src/` — core business logic and managers (agent loop, LLM core, RAG, embeddings, email, calendar sync, cookbook serving, etc.).
- `core/` — framework plumbing: `database.py` (SQLAlchemy models + `SessionLocal`), `auth.py`, `middleware.py`, `session_manager.py`, `exceptions.py`. `core/constants.py` only **re-exports** `src/constants.py` for backward compatibility.
- `mcp_servers/` — built-in MCP servers exposed to the agent (`email_server.py`, `image_gen_server.py`, `memory_server.py`, `rag_server.py`); `integrations/` (`claude/`, `codex/`) and `companion/` add external coding-agent and device-pairing surfaces.
- `scripts/` — standalone maintenance/migration/diagnostic scripts (run directly, not imported); not part of the request path.
- `docs/` — canonical setup/reference docs (`setup.md` for native/GPU/Windows/macOS/HTTPS installs, `security-ci.md` for the blocking-workflow rationale); `specs/architecture-runtime-inventory.md` is the detailed architecture inventory.

### Agent loop & tools
`src/agent_loop.py` wraps `src/llm_core.py::stream_llm()` with a multi-round tool-execution loop. The LLM invokes tools by emitting **fenced code blocks** that are parsed/executed via `src/agent_tools/` (`filesystem_tools`, `subprocess_tools`, `web_tools`, `document_tools`, `session_tools`, `model_interaction_tools`, `bg_job_tools`). Tool gating is enforced per-owner (`src/tool_security.py`, `src/tool_policy.py`). Built-in and external **MCP** servers (`mcp_servers/`, `src/mcp_manager.py`) extend the tool set.

### Persistence
Two stores: JSON files (sessions, memory, presets, settings, auth, …) and a SQLite DB at `DATA_DIR/app.db` (`core/database.py` declares all SQLAlchemy models — `Session`, `ChatMessage`, `Document`, `Memory`, `EmailAccount`, `ScheduledTask`, `CalendarEvent`, etc.). Some columns are transparently encrypted via the `EncryptedText` type. There are also dedicated SQLite DBs for email cache and scheduled emails.

### Frontend
Vanilla ES modules, no build step. `static/index.html` + `static/app.js` + `static/js/<feature>.js` (some features are subdirectories). `static/lib/` is vendored and excluded from `node --check`. Dark theme is default; styling goes through CSS variables in `static/style.css`. `package.json` defines no scripts — it exists only for a test devDependency; there is nothing to `npm run`.

## Project conventions (enforced in review)

### Paths & config — `src/constants.py` is the single source of truth
- Every persisted file/dir has a named constant in `src/constants.py` (e.g. `AUTH_FILE`, `SETTINGS_FILE`, `CHROMA_DIR`, `UPLOAD_DIR`, `APP_DB`). **Import the constant** — never re-derive with `os.path.join(DATA_DIR, "x.json")`, `DATA_DIR / "x.json"`, `Path(__file__)...`, hardcoded `/app/...`, or relative `"data/..."` strings. `DATA_DIR` is the only place that reads `ODYSSEUS_DATA_DIR`; use it directly only for dynamic per-owner paths with no fixed name. If a path has no constant yet, **add one**.
- The source tree is read-only in Docker and `/app/...` doesn't exist on native runs — guard directory creation so an unwritable path degrades gracefully instead of crashing at import.
- Internal/loopback URLs: use `internal_api_base()` from `src.constants`, not hardcoded `http://localhost:7000`.
- Reuse existing constants for ports, limits, and model lists rather than copying literals.

### Visual style (UI changes are scrutinized; PRs ignoring this are closed)
- Reuse existing CSS variables (`--red`, `--fg`, `--bg`, `--card`, `--border`, …), button/input/card classes, and components — don't introduce new color/size/spacing values or parallel widgets.
- **No Unicode emoji in UI or code** — use inline monochrome SVG (matching `static/index.html`) or plain text.
- Monospaced `Fira Code` is the primary UI font; dark theme is default (light mode goes through the theme system).
- Run the app and attach a screenshot (plus mobile if relevant) for any change to buttons, icons, fonts, colors, spacing, layout, CSS/HTML/SVG, or a DOM-drawing `static/js/` module.

## Self-hosting notes that affect code

- **Cross-platform**: Windows-specific guards exist (UTF-8-BOM `.env` parsing in `app.py`; `HF_HUB_DISABLE_SYMLINKS` for HuggingFace on UNC shares). Don't remove them.
- **Docker**: `docker/entrypoint.sh` drops privileges via `gosu` using `PUID`/`PGID` and chowns bind-mounts (the #1 self-host footgun). The Docker socket is bind-mounted so Cookbook can `docker exec` sibling containers. GPU variants live in `docker-compose.gpu-{amd,nvidia}.yml`.
- **Optional deps** (`requirements-optional.txt`, gated by `INSTALL_OPTIONAL`) keep the default image MIT-core; AGPL/heavy extras like PyMuPDF are opt-in. Code that uses them should degrade gracefully (`src/optional_deps.py`).

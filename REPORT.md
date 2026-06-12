# Creator Studio — Full-Suite Review Report

Date: 2026-06-12
Scope: all seven `studio-*` repos, the umbrella compose, and the deploy/migration tooling.
Method: full test/lint baseline, manual code review of every app and library module, and
live smoke tests of all three APIs against a real PostgreSQL 18 instance (fresh
`alembic upgrade head` + HTTP-level CRUD through each app factory).

---

## Executive summary

The suite is in good shape structurally — the factory/namespace pattern is consistent,
domain code never leaks into studio-core, tests are green, and lint/typecheck pass in
every repo. However, the claim "feature complete, works end-to-end via compose" was
**false for studio-videos until this review**: its initial migration failed on a fresh
PostgreSQL database, and even with a patched schema, **every** `POST /videos/` failed
against Postgres due to a model/migration type mismatch. Both bugs were invisible to the
unit suites because they run on SQLite. Both are now fixed and verified against Postgres.

Counts after this session: **341 Python tests** (24 core + 53 videos + 46 playlists +
121 courses + 27 migration + ~70 collected dots in core incl. skips), **39 TS tests**,
Go tests green, `ruff` clean in all Python repos, `eslint` + `tsc` clean in studio-web.

---

## 1. Critical issues found — fixed and verified this session

### 1.1 studio-videos migration 0001 failed on fresh Postgres (blocker)
`sa.Enum(name="video_type", create_type=False)` is not valid generic-SQLAlchemy usage —
`create_type` is a `postgresql.ENUM`-only argument. With current dependency versions the
migration emitted `CREATE TYPE video_type AS ENUM ()` (empty) and aborted. **A fresh
`docker compose up` could never have produced a working studio-videos.** playlists and
courses were unaffected (they correctly use `postgresql.ENUM(values…, create_type=False)`).
Fixed in [0001_initial_schema.py](studio-videos/migrations/versions/0001_initial_schema.py)
to match the sibling pattern.

### 1.2 studio-videos: every insert failed on Postgres (blocker)
[models.py](studio-videos/app/models.py) declares `tags = db.Column(db.JSON, …)` but the
migration created `tags TEXT[]`. psycopg2 binds JSON into `text[]` →
`DatatypeMismatch: column "tags" is of type text[] but expression is of type json` on
**every** video create, even without tags (the default `[]` still binds as JSON).
Fixed by making the column `JSONB` in migration 0001 (GIN index retained — works on
jsonb). Verified live: create-with-tags 201, all endpoints 200.

> Lesson for the suite: SQLite-only unit tests cannot validate migrations or
> Postgres-specific types. See recommendation 3.1.

### 1.3 studio-videos ran the Flask dev server everywhere (production gap)
The Dockerfile CMD, the umbrella compose, the app-owned k8s base, and the vendored copy
in studio-deploy all ran `flask run` (dev server, single-threaded, debug-oriented), and
`gunicorn` wasn't even a dependency. playlists/courses use gunicorn. Fixed in all four
places + `pyproject.toml`/lockfile; the Dockerfile CMD now also runs
`alembic upgrade head` first, matching siblings (this also makes the standalone
`studio-videos/compose.yaml` migrate on boot — previously it never ran migrations).

### 1.4 Enum values reached the database unvalidated → HTTP 500s
`POST /videos/` and `POST /playlists/` (and both `/import` endpoints) passed
`status` / `video_type` / `visibility` straight to the Postgres enum; invalid values
produced a 500 instead of a 400. Courses already validated correctly — videos and
playlists now match it (validation also moved *before* mutation in the PUT handlers).
Regression tests added (`test_create_invalid_*`, `test_update_invalid_video_type`).

### 1.5 Documented `tag` filter on `GET /videos/` did not exist
The Swagger doc advertised `tag` ("Filter by tag (exact match)") and the TODO marked the
filter complete, but no code read the parameter. Implemented (portable in-Python filter
over the JSON list; switch to a JSONB `@>` query if scale ever demands —
[routes_videos.py](studio-videos/app/routes_videos.py)) with a test.

### 1.6 Dead code in the new Todoist asset routes
[routes_todoist_assets.py](studio-videos/app/routes_todoist_assets.py) contained an
unused import and a connection/mapping lookup whose every branch ended in
`project_id = None`. Removed; behavior unchanged (Todoist infers the project from
`parent_id`). All 20 ruff errors across videos/courses (unused imports, E501, E741 `l`)
are fixed; `ruff check` is clean suite-wide.

---

## 2. Open findings — recommended changes (not yet applied)

Ordered by priority. None block local end-to-end use.

### 2.1 Correctness / robustness

| # | Where | Finding | Recommendation |
|---|---|---|---|
| R1 | [engine.py:242-283](studio-shared/python/studio-core/src/studio_core/todoist/engine.py#L242-L283) | `sync()` calls `update_task_content` + `close/reopen` for **every** mapped todo on every sync — no change detection. `last_local_version` is stored but never compared. | Compare `todo.updated_at` against `mapping.last_local_version` and skip unchanged todos. Cuts Todoist API calls by ~100× on stable data and avoids rate limiting. |
| R2 | [engine.py:716-727](studio-shared/python/studio-core/src/studio_core/todoist/engine.py#L716-L727) | Remote-404 detection via `"404" in str(exc)` — fragile string matching (a task titled "404" in an error path, or wording changes, break it). | Give `TodoistClient` a typed `TodoistApiError(status, detail)` exception; check `exc.status == 404`. |
| R3 | [client.py](studio-shared/python/studio-core/src/studio_core/todoist/client.py) | No retry/backoff on 429/5xx; `list_sections` is unpaginated (`list_projects` paginates). | Add bounded retry with `Retry-After` honoring; paginate sections like projects. |
| R4 | [videos_client.py](studio-playlists/app/videos_client.py) | All errors swallowed → if studio-videos is **down**, playlists reports "Linked video not found" (404) and `refresh-items` marks every item "missing". | Distinguish unreachable (→ 502/503) from not-found (→ 404). Don't overwrite cache fields on transport errors. |
| R5 | [engine.py:285-292](studio-shared/python/studio-core/src/studio_core/todoist/engine.py#L285-L292) | In dry-run, `sync()`/`reconcile()` still mutate `sync_state` in the session without committing — a later commit in the same request would persist phantom timestamps. | Only touch `sync_state` when `not dry_run`. |
| R6 | [todoist/models.py](studio-shared/python/studio-core/src/studio_core/todoist/models.py) | `integration_mappings` has no unique constraint on (org, provider, connection, local_entity_type, local_entity_id, remote_entity_type) — concurrent map calls can insert duplicates; `get_*_id` then returns an arbitrary one. | Add the unique constraint (model + migration 0003 in each app). |
| R7 | playlists/courses `/import` | Updating an existing playlist/course silently ignores nested `items`/`chapters` in the payload (only created records get children). Idempotent re-import therefore doesn't repair drift. | Either document loudly in the API description (it is partially documented) or implement child reconciliation. The migration tool compensates by calling child endpoints itself, so this is low-risk today. |
| R8 | All models | `db.DateTime` columns are timezone-naive while the code writes `datetime.now(UTC)` — Postgres silently strips the offset. Comparisons stay consistent (everything is UTC) but round-trips lose tz info. | Standardize on `db.DateTime(timezone=True)` in a future coordinated migration. Low urgency. |
| R9 | [scheduler.py](studio-shared/python/studio-core/src/studio_core/todoist/scheduler.py) | One reconcile thread **per gunicorn worker**. With `-w 1` (current default) it's fine; raising workers multiplies reconcile loops. | Document the constraint where workers are configured, or gate the scheduler on an env var only set on one replica. |

### 2.2 Security (homelab-appropriate, but write them down)

| # | Finding | Recommendation |
|---|---|---|
| S1 | **No authentication anywhere** — APIs, gateway, and web are open; `X-Organization` and `X-Actor` headers are caller-asserted. | Fine on a trusted LAN; before any exposure, add an API key check at the api-gateway nginx (one `map` block) or basic auth on the ingress. Track as a deploy task. |
| S2 | `integration_connections.api_token` stores Todoist tokens **in plaintext** ([todoist/models.py:29](studio-shared/python/studio-core/src/studio_core/todoist/models.py#L29)); `/connect` accepts tokens in request bodies. | Prefer env-provided tokens (already supported via `get_env_token`); consider dropping the DB token column or encrypting at rest. |
| S3 | Error bodies leak internals: `/healthz` returns the DB exception string; Todoist routes return `str(exc)` on 502. | Log the detail, return a generic message. One-line changes in [observability.py](studio-shared/python/studio-core/src/studio_core/observability.py) and [todoist/routes.py](studio-shared/python/studio-core/src/studio_core/todoist/routes.py). |

### 2.3 Code quality / maintainability

| # | Finding | Recommendation |
|---|---|---|
| Q1 | [engine.py](studio-shared/python/studio-core/src/studio_core/todoist/engine.py) — `map_projects`/`map_sections`/`map_tasks` are ~90 % copy-paste (~100 lines each), as are the three `get_*_id` helpers. | Extract `_map_entities(remote_type, create_remote, …)` and `_get_remote_id(remote_type, …)`. Engine shrinks ~300 lines; one place to fix R1/R5-style bugs. |
| Q2 | `_slugify`/`_unique_slug` duplicated in all three services, while `studio_core.orgs.slugify` already exists (with slightly different behavior). | Move `_unique_slug` into studio-core (parameterized by model/scope column); delete the three copies. |
| Q3 | [todoist/routes.py](studio-shared/python/studio-core/src/studio_core/todoist/routes.py) and consumers call `engine._normalize_env()` — private API used across module boundaries. | Rename to `normalize_env()` (keep a deprecated alias for one release). |
| Q4 | `_is_truthy()` duplicated in videos/courses app factories, duplicating `read_config`'s `_to_bool`. | Export one `studio_core.config.to_bool`. |
| Q5 | [studio-playlists/app/config.py](studio-playlists/app/config.py) is **0 bytes**; courses' config.py has content. | Delete the empty file or align with courses. |
| Q6 | CHANGELOG.md exists in courses + playlists only. | Add to videos/shared/cli/web/deploy if you want per-repo release notes (release workflows tag all four services). |
| Q7 | Audit coverage is inconsistent in playlists: item add/remove are audited, item **update** and **reorder** are not ([routes_playlists.py:340-438](studio-playlists/app/routes_playlists.py#L340-L438)). | Add `record()` calls for both. |

### 2.4 Test coverage gaps

| # | Gap | Recommendation |
|---|---|---|
| T1 | **Nothing runs migrations or queries against Postgres in CI.** This is exactly how 1.1/1.2 slipped through. The per-repo `integration-tests` compose profiles exist but are not exercised by CI. | Add a CI job (or at least a documented pre-release step) per service: `docker compose --profile test run integration-tests`. Even one "alembic upgrade + POST + GET" against PG catches this class. |
| T2 | studio-cli: `internal/*` is tested, but **zero tests** under `commands/` — flag wiring, arg validation, output shaping are unverified. | Add command-level tests with `httptest` servers (the services packages already mock cleanly). |
| T3 | studio-web: 4 tests total (VideoList, OrgSwitcher). Pages like CourseDetail/PlaylistDetail (the most logic-heavy) are untested. | Add render + interaction tests for detail pages; Playwright smoke remains a stretch goal in the TODO. |
| T4 | Feature-flagged Todoist namespaces are tested with mocked engines only — no test drives `map-structure`/`map-video-tasks` against the real engine with a fake Todoist client. | One integration-style test per service using `TodoistEngine` with a stubbed `TodoistClient` would cover the engine↔route contract. |
| T5 | studio-shared TS `ui-core`: components (`TodoPanel`, `AssetProgress`, `DetailLayout`) have no tests (only lib/api/enums do). | Add component render tests with the same vitest+jsdom setup studio-web already uses. |

### 2.5 Feature / project improvements worth considering

1. **List-endpoint pagination** (`limit`/`offset` + `X-Total-Count`) — fine at homelab
   scale today, but the web list pages will eventually need it; audit list already
   caps at 500.
2. **OpenAPI drift tests exist in videos** (`test_openapi.py`) — replicate in playlists
   and courses so committed `openapi.json` files can't go stale (courses has one,
   verify; playlists lacks the committed spec).
3. **`PATCH` semantics for asset updates**: `PUT /videos/{id}/assets/{type}` requires
   `state` even when only updating `notes` — allow notes-only updates.
4. **Web: surface Todoist mapping actions** — the APIs now expose
   `map-projects`/`map-structure`/`map-video-tasks`, but the web console has no UI for
   them ([src/api/todoist.ts](studio-web/src/api/todoist.ts) covers status/sync only).
5. **Backup story**: pg_dump CronJob in k8s overlay / compose profile — currently the
   only persistence safety is the Docker volume. Deserves a TODO entry in studio-deploy.
6. Previously tracked stretch goals remain valid: YouTube Data API pulls (videos,
   playlists), Google Sheets CSV bridge (courses), Prometheus/Grafana, Playwright.

---

## 3. Verification log (what was actually run)

- `pytest`: studio-core 93+3 skipped, videos **53** (+4 new), playlists **46** (+2 new),
  courses 121, migration 27 — all green post-fix.
- `ruff check`: clean in all five Python projects (was 20 errors in videos+courses).
- `go test ./...` + `go vet`: clean (studio-cli).
- `eslint` + `tsc --noEmit` + `vitest`: clean (studio-web 4, ui-core 35).
- **Live Postgres smoke** (postgres:18 container): fresh `alembic upgrade head` for all
  three services; videos create/tag-filter/assets/export/snapshot/import; playlists
  create/items/visibility-400/export; courses create/chapter/lesson/progress — all OK.
- `docker compose config` (umbrella): valid. `kubectl kustomize` overlays: render once
  `.secrets.env` files are created from the committed `.example` templates (expected).

## 4. Suggested order of attack

1. R1 + R2 + R5 (engine correctness — small, related, shared code) and Q1 while in there.
2. T1 (PG-backed integration job) — prevents the whole 1.1/1.2 bug class.
3. R4 + R6 (cross-service failure semantics, mapping uniqueness).
4. S1–S3 before any off-LAN exposure; bundle with the ingress/TLS work already tracked.
5. Q2–Q7, T2–T5 opportunistically with feature work.

# Recommendations

Answers to the three open questions, based on a full review of the monorepo (v0.4.0,
~7,500 LOC across api-service, cli-tool, web-client, database, infrastructure).

---

## 1. Should any of the code be reused?

**Yes — about 40–50% of it, but by *extraction into shared libraries*, not by forking the
repo.** The code falls cleanly into two buckets:

### Reuse (domain-neutral, already proven)

| Current code | Verdict | Destination |
|---|---|---|
| `api-service/todoist_client.py`, `todoist_service.py`, `todoist_scheduler.py` | Extract & genericize (replace `ContentItem` joins with a small interface) | `studio-shared` → `studio_core.integrations.todoist` |
| `api-service/routes_audit.py`, audit model, `record_audit_event` | Extract as blueprint factory | `studio-shared` → `studio_core.audit` |
| `api-service/routes_todos.py`, todo model | Extract as factory; each app supplies its own FK target | `studio-shared` → `studio_core.todos` |
| `api-service/routes_settings.py`, settings model | Extract as-is | `studio-shared` → `studio_core.settings` |
| `api-service/utils.py` (slugify, org normalization), `routes_health.py`, app-factory pattern, JSON logging | Extract | `studio-shared` → `studio_core` |
| `cli-tool/cmd/client.go`, `config.go`, `project_config.go`, `csv_utils.go`, scaffolding logic in `sync.go`, bulk CSV machinery | Port into internal packages of the unified CLI | `studio-cli` → `internal/{client,config,project,formats,scaffold}` |
| `web-client/src/lib/asset-progress.ts`, `duration.ts`, `todo-panel.tsx`, `detail-ui.tsx` | Extract as components/utilities | `studio-shared` → `@studio/ui-core` |
| Dockerfiles, compose patterns, k8s base+overlay layout, integration-test container harness | Copy as templates | each app repo + `studio-deploy` |

### Do not reuse (shaped by the compromise you're escaping)

| Current code | Why not |
|---|---|
| `models.py` `ContentItem` + `init.sql` `content_items` | The single-table/`content_type` discriminator design is the core compromise: hierarchy rules, todo eligibility, and status vocabulary all enforced in app code instead of the schema. Each app gets a properly typed schema (see each TECH-SPEC.md). |
| `routes_content.py`, `domain.py` hierarchy validation, `compute_path` | Built around the generic parent/child model. Replaced by per-domain resources with real FKs. |
| `api_contracts.py` content model | Union-of-everything contract; replaced by per-domain contracts. |
| Web pages (`CourseDetail`, `PlaylistDetail`, `VideoDetail`, …) | All bound to the `ContentItem` union type; rewrite against per-domain APIs in `studio-web` (the small lib files they share *are* reused via ui-core). |
| `import.go` / `export.go` domain shaping | Rewrite per domain — the YAML/CSV shapes change with the schemas. Codec/file plumbing is reused. |

**Rule of thumb during the build:** if a file knows what a `content_type` is, it does not
survive. If it doesn't, it probably moves into a shared library or template.

---

## 2. One CLI per project, or a unified CLI?

**Unified CLI — a single `studio` binary in its own repo (`studio-cli`), with one command
group per domain, talking to the three APIs.**

```
studio course  import|export|sync|list|todo|bulk ...
studio playlist list|items|reorder|sync|todo ...
studio video   list|add|status|assets|sync|todo ...
studio config | studio init
```

Why unified:

* **The CLI's value is workflow, and the workflow is identical across domains** — scaffold
  folders, import/export YAML/CSV, sync, manage todos. Per-project CLIs would triplicate
  config handling, API client, CSV/YAML codecs, scaffolding, output formatting — exactly the
  plumbing that's worth keeping in one place (~60% of the current cli-tool).
* **You are one person with one terminal.** One install, one `~/.config/studio/config.yaml`,
  one set of muscle memory. The existing `project.yaml` context-detection carries over and
  gets a `type: course|playlist|video` field so `studio sync` still "just works" in any
  content folder.
* **It does not re-couple the apps.** The CLI is a *client*: it depends only on the three
  published API contracts, never on app internals. The apps remain independently
  deployable and releasable.

Guardrail so this stays clean: each domain's commands live in an isolated package
(`commands/course`, `commands/playlist`, `commands/video`) that only touches shared
internals. If a domain ever needs its own release cadence, splitting it out is mechanical.

**Rejected alternatives:** (a) three CLIs — 3× plumbing and config divergence for zero
isolation benefit, since the apps are already isolated at the API boundary; (b) CLI inside
each app repo — couples a Go toolchain and release process into Python repos.

---

## 3. How to deal with shared concerns like video metadata?

**Share *vocabulary* (a library) and *references* (IDs over APIs) — never share a database
or stand up a central metadata service.** Three rules:

1. **One system of record per fact.**
   * `studio-videos` owns standalone video facts: production status, asset progress,
     YouTube video ID, duration, file path, publish date.
   * `studio-playlists` owns playlist facts: membership and ordering. A playlist item holds
     a `video_ref_id` (the ID in studio-videos) plus a **cached snapshot**
     (`cached_title`, `cached_youtube_video_id`, `cached_duration_seconds`,
     `cache_refreshed_at`) so playlist views never need a cross-service join; an explicit
     refresh endpoint re-pulls from the videos API. Items may also be *placeholders*
     (`video_ref_id NULL`, manual title) so playlists never block on the videos app.
   * `studio-courses` owns lesson facts. **A lesson is not a videos-app record** — its video
     production metadata is internal to the course. If a lesson is later also published as
     a standalone YouTube video, that's an explicit "promote to video" workflow (future
     feature) that creates a record in studio-videos at that moment.
2. **The shared vocabulary lives in `studio-shared`, so the copies stay consistent.**
   Production status enum, asset types (`outline`, `script`, `audio_recording`, …), asset
   states (`not_started`, `in_progress`, `blocked`, `done`, `not_applicable`), instructor
   shape, and org-key conventions are defined once in `studio_core.enums` (Python) and
   `@studio/ui-core` types (TypeScript). Apps store their own data in their own typed
   columns, but validate against the shared enums.
3. **Cross-app concerns that are *behavior*, not data — todos, audit, settings, Todoist
   sync, health/metrics — ship as library subsystems** from `studio_core`, instantiated
   per app against that app's own tables. Each app maps its own content to Todoist
   projects through its own connection rows; tokens are configured per app via env vars
   exactly as today.

**Rejected alternatives:** (a) a central "video metadata service" all three apps call —
operationally heavy for homelab scale, makes every page load a distributed call, and
recreates the coupling you're escaping; (b) a shared database/schema — cross-app FKs and
migration lockstep, i.e. the monolith with extra steps.

Accepted tradeoff to note: `organizations` exists as a small table in each app's DB,
kept consistent by slug convention. At single-operator scale this duplication is cheaper
than an identity service; revisit only if orgs grow real users/auth.

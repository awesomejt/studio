# Migration Plan — monorepo → studio-* repos

Two migrations happen here: **code salvage** (covered in
[RECOMMENDATIONS.md](RECOMMENDATIONS.md) §1 — extract, don't fork) and **data migration**
(this document). Data migration is the *last* step, after the three apps reach feature
parity and the CLI covers daily workflows.

## Source inventory (current schema)

Single `content_items` table (discriminator `content_type`) plus `todos`, `settings`,
`audit_events`, `integration_connections`, `integration_mappings`, `integration_sync_state`.

## Mapping: content_items → new homes

| Source rows | Destination | Notes |
|---|---|---|
| `content_type='course'` | `courses_db.courses` | `metadata` JSONB keys promoted to typed columns: `abbreviation`, `category`, `target_platform`, `instructors`. Unmapped keys → `extra` JSONB (kept for manual review). |
| `content_type='chapter'` | `courses_db.chapters` | `parent_id` → `course_id`; `ordering` → `position`. Orphaned chapters (parent missing/wrong type) are reported, not migrated. |
| `content_type='lesson'` | `courses_db.lessons` | `parent_id` → `chapter_id`; `asset_progress` JSONB → `lesson_assets` rows (key → `asset_type`, value → `asset_state`; unknown keys reported). |
| `content_type='playlist'` | `playlists_db.playlists` | |
| `content_type='video'`, `parent_id IS NULL` | `videos_db.videos` | Standalone videos. |
| `content_type='video'`, parent is a playlist | `videos_db.videos` **+** `playlists_db.playlist_items` | Video row created first; the playlist item stores `video_ref_id` + cached snapshot; `ordering` → `position`. |
| `path` values | recomputed | New apps compute `relative_path` from slugs; old paths are not trusted (old `compute_path` had org/rename edge cases). |

### Status / vocabulary mapping

| Old | New |
|---|---|
| `completed` | `published` (videos/playlists) / `completed` (lessons/courses) — review per row |
| asset_progress values (`draft`, `pending`, `done`, `in-progress`, free text) | normalize → `not_started` / `in_progress` / `done`; anything else → `in_progress` + report line |

## Mapping: supporting tables

| Source | Destination | Notes |
|---|---|---|
| `todos` | split by parent content type → `courses_db.todos` / `playlists_db.todos` / `videos_db.todos` | FK rewritten to the new entity ID via the migration's ID map. |
| `settings` | copied to each app where the key is relevant (`default_instructor`, `default_lesson_type` → courses; `preferred_platform` → all) | |
| `audit_events` | **not migrated** | History stays in a final SQL dump of the old DB (archive). New apps start their audit trail at migration time, with one synthetic `migrated` event per entity recording the old ID. |
| `integration_connections` / `mappings` / `sync_state` | **not migrated** | Re-connect Todoist per app after cutover; cheaper and safer than translating mapping rows. Old remote tasks remain in Todoist; first sync re-maps by creating fresh tasks (archive old Todoist projects first to avoid duplicates). |

## Mechanics

1. One-off Python script in `studio-deploy/migration/` (`uv` project): reads the old DB
   directly (read-only), writes through the **new apps' APIs** (not direct SQL) so all
   validation, slug/path computation, and audit hooks run.
2. Order: organizations (derived from distinct `organization` values) → videos →
   playlists (+items) → courses → chapters → lessons (+assets) → todos.
3. The script maintains an `old_id → (app, new_id)` map, emitted as `migration-report.json`
   together with all skip/normalize warnings.
4. Idempotent: safe to wipe the new DBs and re-run until the report is clean.

## Cutover checklist

- [ ] Feature parity confirmed: daily workflows (import, sync/scaffold, todo, export) work via `studio` CLI against all three apps
- [ ] Final `pg_dump` of old database stored as archive artifact
- [ ] Migration script run; `migration-report.json` reviewed, zero unexplained skips
- [ ] Spot-check: one course tree, one playlist with items, standalone videos, todos
- [ ] Old Todoist projects archived; per-app Todoist connections established
- [ ] CLI config switched (`~/.cman.yaml` retired → `~/.config/studio/config.yaml`); `project.yaml` files in working folders updated with `type:` + new IDs (script emits a rewrite helper)
- [ ] Monorepo: README banner pointing at new repos, repo archived on GitHub

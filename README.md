# Creator Studio — Refactor Workspace

Generated 2026-06-11 from a full review of the `course-management` monorepo (v0.4.0).

This folder contains the blueprint for splitting the monolith into focused projects. Each
subfolder maps 1:1 to a future GitHub repository and contains everything needed to start
work: requirements, technical spec, agent workflow instructions, TODO backlog, project
metadata, and seeded memory.

> **Naming:** `studio-*` and the `studio` CLI binary are working names for the product
> family ("Creator Studio"). They are used consistently across all documents so a
> rename is a mechanical find/replace. `<owner>` is a placeholder for your GitHub user/org.

## Repository map

| Folder / future repo | Purpose | Stack | Depends on |
|---|---|---|---|
| [studio-videos](studio-videos/) | API + DB for standalone video management. **System of record for videos.** | Python 3.14, Flask, Postgres 18 | studio-shared |
| [studio-playlists](studio-playlists/) | API + DB for YouTube playlist management. References videos by ID. | Python 3.14, Flask, Postgres 18 | studio-shared, studio-videos (API only) |
| [studio-courses](studio-courses/) | API + DB for online training courses (course → chapter → lesson). | Python 3.14, Flask, Postgres 18 | studio-shared |
| [studio-shared](studio-shared/) | Shared libraries: `studio-core` (Python) and `@studio/ui-core` (TypeScript). | Python + TypeScript packages | — |
| [studio-cli](studio-cli/) | **Unified** `studio` CLI: scaffolding, import/export, sync, todos for all three domains. | Go, Cobra | App APIs (contracts only) |
| [studio-web](studio-web/) | Unified web console covering all three domains. | React, TypeScript, Vite | App APIs, studio-shared (ui-core) |
| [studio-deploy](studio-deploy/) | Homelab deployment umbrella: compose + k3s kustomize for the full stack. | Docker Compose, Kustomize | All of the above (manifests only) |

Key root documents:

* [RECOMMENDATIONS.md](RECOMMENDATIONS.md) — answers to the three open questions (code reuse, CLI shape, shared video metadata), with rationale and rejected alternatives.
* [ARCHITECTURE.md](ARCHITECTURE.md) — target architecture: data ownership, API conventions, tenancy, deployment topology.
* [MIGRATION-PLAN.md](MIGRATION-PLAN.md) — old → new data mapping, code salvage map, decommission checklist.

## How to turn a folder into a repo

For each project folder, in dependency order:

```bash
gh repo create <owner>/studio-videos --private
mkdir studio-videos && cp -r refactor/studio-videos/* studio-videos/
cd studio-videos
git init -b main && git add -A && git commit -m "Bootstrap from refactor blueprint"
git remote add origin git@github.com:<owner>/studio-videos.git && git push -u origin main
```

The docs live at the repo root; code is scaffolded next to them per each repo's TECH-SPEC.md.

## Recommended build order

1. **studio-shared** — extract the proven, domain-neutral code from the monorepo first
   (Todoist engine, audit/todo/settings subsystems, observability). Everything else builds on it.
2. **studio-videos** — smallest domain; proves the per-app template (typed schema, Alembic,
   org scoping, audit, Todoist) end to end.
3. **studio-cli** — bootstrap with `studio video ...` commands against studio-videos.
4. **studio-playlists** — adds the cross-app pattern (video references + cached snapshots).
   Add `studio playlist ...` commands.
5. **studio-courses** — largest domain, done last when the template is settled.
   Add `studio course ...` commands.
6. **studio-web** — unified console once the three APIs are stable.
7. **studio-deploy** — umbrella manifests once two or more services run in the homelab.
8. **Data migration** — per [MIGRATION-PLAN.md](MIGRATION-PLAN.md), then archive the monorepo.

## What stays behind

The `course-management` monorepo remains the running system until migration completes.
Freeze new features there; bug fixes only. Archive (don't delete) the repo afterwards —
the audit history and git history stay retrievable.

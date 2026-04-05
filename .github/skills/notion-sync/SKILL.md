---
name: notion-sync
description: Bidirectional sync between tasks.md and a Notion database. Use when asked to push tasks to Notion, pull Notion statuses into tasks.md, sync task progress, assign sprints, or check sync status between local markdown task files and Notion. Supports push, pull, bidirectional sync, status diff, sprint assignment, and dry-run previews.
---

# Notion Sync

Keeps `specs/*/tasks.md` and a Notion database in sync via a Python script that uses the Notion REST API (no extra dependencies — calls `curl` under the hood).

## When to Use This Skill

- User asks to **push tasks to Notion** after implementing or updating `tasks.md`
- User asks to **pull Notion statuses** back into `tasks.md` checkboxes
- User asks to **sync tasks** or keep Notion up to date
- User asks to **check what's out of sync** between `tasks.md` and Notion
- User asks to **assign sprints** to Notion task pages
- User wants a **dry-run preview** of any sync operation
- Agent finishes a task and needs to mark it done in both places

## Prerequisites

1. **Python 3.9+** available in the environment (`python3 --version`)
2. **`curl`** available (`curl --version`)
3. **`.env` at repo root** (or exported environment variables) containing:
   ```
   NOTION_TOKEN=secret_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   NOTION_DB_ID=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
4. The Notion integration must be **connected to the target database** (share the DB with the integration in Notion settings)
5. `tasks.md` must exist at `specs/001-investment-intel-poc/tasks.md` with the standard format:
   ```markdown
   ## Phase 1
   - [ ] T001  Task description [US1]
   ```

## Quick Reference

| Command | What It Does |
|---|---|
| `push` | `tasks.md` → Notion (upsert content, preserve Notion status) |
| `push --push-status` | `tasks.md` → Notion (upsert content **and** status) |
| `pull` | Notion → `tasks.md` (update checkboxes from Notion Status) |
| `sync` | Pull status first, then push content + status (full bidirectional) |
| `status` | Read-only diff report (no writes) |
| `sprint <N\|all>` | Assign `Sprint` field in Notion based on phase mapping |
| `--dry-run` | Preview any command without writing anything |

## Step-by-Step Workflows

### 1. Check Current Sync State

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py status
```

Always run this first to understand what's out of sync before making changes.

### 2. Push Local Changes to Notion

After editing `tasks.md` (new tasks, updated descriptions, or marking tasks done):

```bash
# Push content only (preserves Notion status)
python3 .github/skills/notion-sync/scripts/sync_tasks.py push

# Push content AND status (when you've marked tasks done in tasks.md)
python3 .github/skills/notion-sync/scripts/sync_tasks.py push --push-status
```

### 3. Pull Notion Status into tasks.md

After a PM or teammate updates statuses in Notion:

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py pull
```

### 4. Full Bidirectional Sync (Recommended After Sprints)

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py sync
```

Runs pull (Notion → tasks.md), re-parses, then push with status (tasks.md → Notion).

### 5. Assign Sprints

```bash
# Assign a single sprint
python3 .github/skills/notion-sync/scripts/sync_tasks.py sprint 1

# Assign all sprints at once
python3 .github/skills/notion-sync/scripts/sync_tasks.py sprint all
```

### 6. Dry-Run Any Command

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py sync --dry-run
python3 .github/skills/notion-sync/scripts/sync_tasks.py sprint all --dry-run
```

## Typical Agent Flow

When an agent finishes implementing a task:

1. Edit `tasks.md` to mark the task done: `- [x] T042  ...`
2. Run: `python3 .github/skills/notion-sync/scripts/sync_tasks.py push --push-status`
3. Verify with: `python3 .github/skills/notion-sync/scripts/sync_tasks.py status`

## References

- [Full Workflow & Troubleshooting Guide](./references/workflow.md) — Detailed command docs, environment setup, tasks.md format, troubleshooting table
- [Sync Script](./scripts/sync_tasks.py) — The Python script (no third-party deps; uses `curl` for Notion API calls)

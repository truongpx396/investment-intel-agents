# Notion Sync – Detailed Workflow Reference

## Environment Setup

### Required Variables

| Variable | Description | Where to Set |
|---|---|---|
| `NOTION_TOKEN` | Notion integration secret (Internal Integration Token) | `.env` or shell environment |
| `NOTION_DB_ID` | The Notion database ID to sync against | `.env` or shell environment |

Place both in the repo-root `.env` file (auto-loaded by the script):

```
NOTION_TOKEN=secret_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
NOTION_DB_ID=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Prerequisites Check

```bash
# Verify .env is present
ls -la .env

# Dry-run status (no writes, no auth required beyond read)
python3 .github/skills/notion-sync/scripts/sync_tasks.py status
```

---

## Command Reference

### `push` — tasks.md → Notion

Upserts every task from `tasks.md` into the Notion database. Creates missing pages, updates changed content. **Preserves** Notion `Status` by default.

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py push
```

With `--push-status`: also writes checkbox states (`[ ]`, `[-]`, `[x]`) to Notion Status:

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py push --push-status
```

**Fields pushed:** Task ID, Phase, Story, Parallel flag, Description, (optionally) Status.

---

### `pull` — Notion → tasks.md

Reads each task's `Status` from Notion and rewrites the corresponding checkbox marker in `tasks.md`.

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py pull
```

**Status mapping:**

| Notion Status | tasks.md marker |
|---|---|
| `Not Started` | `- [ ]` |
| `In Progress` | `- [-]` |
| `Done` | `- [x]` |

---

### `sync` — Bidirectional

Runs pull then push in sequence. Pulls Notion statuses first (re-parses the file), then pushes all content **plus** status back to Notion.

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py sync
```

Equivalent to:
```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py pull && \
python3 .github/skills/notion-sync/scripts/sync_tasks.py push --push-status
```

---

### `status` — Read-only Diff

Prints a report of differences between `tasks.md` and Notion. Makes **no changes**.

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py status
```

Output includes:
- Task counts (local vs Notion)
- Notion status breakdown with bar chart
- Tasks with content or status differences
- Tasks only in `tasks.md` (not yet pushed)
- Tasks only in Notion (deleted locally)

---

### `sprint <N|all>` — Assign Sprint Tags

Assigns the Notion `Sprint` select property for all tasks whose `Phase` belongs to the given sprint. Uses the mapping below.

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py sprint 1
python3 .github/skills/notion-sync/scripts/sync_tasks.py sprint all
```

**Sprint → Phase Mapping:**

| Sprint | Phases |
|---|---|
| Sprint 1 | Phase 1 – Setup, Phase 2 – Foundation |
| Sprint 2 | Phase 3 – Strategy Config, Phase 4 – Telegram Linking |
| Sprint 3 | Phase 5 – Notifications, Phase 6 – Alert History |
| Sprint 4 | Phase 7 – Watchlist & Digest, Phase 8 – Billing |
| Sprint 5 | Phase 9 – Admin, Phase 10 – Quality Gates, Phase 11 – Deploy |

---

### `--dry-run` — Preview Without Writing

Append `--dry-run` to any command to preview what would happen without making any writes:

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py push --dry-run
python3 .github/skills/notion-sync/scripts/sync_tasks.py pull --dry-run
python3 .github/skills/notion-sync/scripts/sync_tasks.py sync --dry-run
python3 .github/skills/notion-sync/scripts/sync_tasks.py sprint all --dry-run
```

---

## Typical Agent Workflow

**Scenario: Agent finishes implementing a task and marks it done in `tasks.md`.**

```bash
# 1. Mark done in tasks.md (agent edits the file)
#    - [x] T042  Implement price alert webhook [US2]

# 2. Push status and content to Notion
python3 .github/skills/notion-sync/scripts/sync_tasks.py push --push-status

# Or, do a full bidirectional sync (pulls any Notion updates first)
python3 .github/skills/notion-sync/scripts/sync_tasks.py sync
```

**Scenario: PM updated statuses in Notion, agent needs latest state in `tasks.md`.**

```bash
python3 .github/skills/notion-sync/scripts/sync_tasks.py pull
```

---

## tasks.md Task Format

The script parses tasks under `## Phase N` headings:

```markdown
## Phase 1

- [ ] T001  Setup repository and CI pipeline
- [-] T002  Configure environment variables [US0] [P]
- [x] T003  Initialize database schema [US1]
```

**Markers:**
- `[P]` — Parallel task (Notion `Parallel` checkbox = true)
- `[USN]` — Story tag (e.g. `[US1]` → `US1 – Signal Strategy Config`)

Tasks not under a `## Phase N` heading are assigned `Phase 1 – Setup` by default.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `ERROR: NOTION_TOKEN not set` | Missing `.env` or env var | Add `NOTION_TOKEN=secret_...` to `.env` |
| `ERROR: NOTION_DB_ID not set` | Missing `.env` or env var | Add `NOTION_DB_ID=...` to `.env` |
| `ERROR: tasks.md not found` | Wrong working directory | Run from repo root, or check `TASKS_FILE` path in script |
| `401 Unauthorized` from Notion | Token is invalid or expired | Regenerate integration token in Notion settings |
| `404` on DB query | Wrong `NOTION_DB_ID` or integration not connected to DB | Share DB with the integration in Notion |
| Tasks parsed as 0 | Malformed `tasks.md` (missing `## Phase N` or wrong checkbox format) | Ensure tasks follow `- [x] TYYY description` format |
| Rate limit errors | Too many rapid requests | Script enforces 0.35s delay; if still hitting limits, run again |

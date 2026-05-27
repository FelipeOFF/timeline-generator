# timeline-generator

Living drag-and-drop Gantt timeline as a single self-contained HTML.

**Source-agnostic.** GSD `.planning/`, Jira epic, Linear project, manual task list, or a JSON schema. Adapters normalize everything to one shape.

**Daily granularity.** Bars snap by day; tooltip shows exact date, weekday, duration. Today is highlighted with a vertical marker.

**Living document.** A `--hook` flag installs Claude Code hooks that re-run the renderer when work progresses (phase complete, file edited). Status updates re-paint bars (`pending → in-progress → done`) without losing user drag overrides.

## Quick start

```bash
# 0. (one-time, if using Jira or Linear) — interactive prompts for credentials
python3 ~/.claude/skills/timeline-generator/scripts/config.py init

# 1. From a GSD project (cwd has .planning/)
python3 ~/.claude/skills/timeline-generator/scripts/generate.py --source gsd

# 2. From a Jira epic URL (needs JIRA_BASE_URL+JIRA_EMAIL+JIRA_API_TOKEN via env or config)
python3 .../scripts/generate.py --source jira --input https://x.atlassian.net/browse/PROJ-100

# 3. From a Linear project/issue URL (needs LINEAR_API_KEY via env or config)
python3 .../scripts/generate.py --source linear --input https://linear.app/team/project/abc

# 4. From a text list
python3 .../scripts/generate.py --source manual --input tasks.txt --days 30 --title "Sprint 12"

# 5. From a JSON schema file
python3 .../scripts/generate.py --source json --input my-data.json
```

## Config

Credentials resolve in this order: **env var > `config.json` > prompt user**.

```bash
python3 .../scripts/config.py show              # print resolved (tokens masked)
python3 .../scripts/config.py init              # interactive prompts
python3 .../scripts/config.py set JIRA_BASE_URL=https://acme.atlassian.net
python3 .../scripts/config.py unset JIRA_API_TOKEN
python3 .../scripts/config.py get JIRA_BASE_URL
```

`config.json` lives inside the skill folder (`~/.claude/skills/timeline-generator/config.json`) with mode `0600`. Gitignored.

## Manual list format

One task per line. Tags work anywhere in a line:

```
- [x] Setup repo @dev #backend                # checklist + assignee + lane tag
- [ ] Build API #backend PROJ-101                # Jira key auto-linked
- [ ] Deploy staging *in-progress #devops        # explicit status tag
- [ ] Smoke tests !blocked #qa
- Docs #docs https://example.com                 # URL becomes a link
```

Status tags: `~done`, `*in-progress`, `?in-review`, `!blocked`. Markdown checkboxes `[x]`/`[/]` also work.

## Schema (canonical)

See `examples/schema.json`. Minimum required:

```json
{
  "meta": {"title":"T", "base_date":"YYYY-MM-DD", "total_days": 30},
  "lanes": [{"id":"l1","label":"Lane","color":"#38bdf8","bars":[
    {"id":"t1","label":"Task","span":[1,8],"status":"pending"}
  ]}]
}
```

`span: [start, end_exclusive]` in day units (1 ≤ start < end ≤ total_days + 1).

## Updating the timeline

```bash
# After "PROJ-N done" or "Phase N complete" — patches the inline JSON in place
python3 .../scripts/update.py --note "PROJ-101 done" --out docs/timeline.html
python3 .../scripts/update.py --note "Phase 23 complete"
python3 .../scripts/update.py --note "PR #42 merged"

# Re-run the original generator (reads <out>.tlconfig.json sidecar)
python3 .../scripts/update.py --auto

# Only refresh if file mtime changed (used by hook)
python3 .../scripts/update.py --watch .planning/STATE.md --auto
```

## Hooks (--hook)

```bash
# Install (project-scope: .claude/settings.json in cwd)
bash ~/.claude/skills/timeline-generator/scripts/install-hook.sh --out docs/timeline.html

# Global (~/.claude/settings.json)
bash .../scripts/install-hook.sh --scope global --out docs/timeline.html

# Uninstall
bash .../scripts/install-hook.sh --uninstall --out docs/timeline.html
```

Installed hooks:
- `Stop` (after assistant turn) → `update.py --auto` (idempotent)
- `PostToolUse` (Edit/Write) → `update.py --watch .planning/STATE.md`

Requires `jq` (`brew install jq`).

## Drag overrides

Drag-to-move and drag-to-resize on the Gantt bars are persisted in `localStorage` keyed by bar `id`. Buttons in the top-right of the page:

- **⬇ Export** — download overrides as JSON
- **⬆ Import** — load overrides JSON (pick file)
- **↺ Reset** — clear all overrides and re-render

Regenerating the HTML does NOT lose your overrides (they live in browser storage).

## Today marker + bottom panel

Page detects `new Date()` and:
- Renders a vertical golden line at today's day column
- Bottom 3 cards split tasks into:
  - 🔥 *Em execução agora* — `start ≤ today < end`
  - 📋 *Próximas* — sorted by start ascending
  - ✅ *Concluídas* — status=done or `end ≤ today`

## --redesign (UI iteration via ui-ux-pro-max)

Pass `--redesign` to print a marker the skill workflow detects and invokes the `ui-ux-pro-max` subagent with the schema + current template. Subagent can rewrite the visual.

## File layout

```
timeline-generator/
├── SKILL.md
├── README.md
├── template/
│   ├── timeline.html.template
│   └── timeline.js
├── scripts/
│   ├── generate.py
│   ├── update.py
│   └── install-hook.sh
├── adapters/
│   ├── gsd.py
│   ├── jira.py
│   ├── linear.py
│   ├── manual.py
│   └── json.py
└── examples/
    └── schema.json
```

## Limitations / known caveats

- Pin positions (interventions) only render at integer days (no fractional).
- Manual adapter `--input -` (stdin) requires explicit path (`tasks.txt`); use a file for now.
- Jira/Linear adapters group lanes by assignee. To split differently, build the schema yourself and use `--source json`.
- `--redesign` is a hand-off marker; the actual visual iteration is performed by the calling agent (Skill workflow), not by this CLI.

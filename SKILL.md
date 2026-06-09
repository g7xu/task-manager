---
name: jason-task-manager
description: Jason's personal task-management skill for his Notion "Task Command Center". Use when he says "good morning", "daily update", "plan my day", "end of day"/"wrap up", asks to add, import, reschedule, or review tasks, or asks to "check/verify/audit my task command center" (drift check + repair). Enforces his well-formed-task framework (outcome-first, atomic, verb-led title, measurable Done-when) and runs the morning-plan and end-of-day-review rituals.
---

# Jason's Task Manager

> A Claude Code skill Jason built to run his own Notion task system.

This skill manages the user's tasks in their Notion **Task Command Center** via the
`claude.ai Notion` MCP tools (`notion-fetch`, `notion-create-pages`,
`notion-update-page`, `notion-update-view`, `notion-update-data-source`, `notion-search`).

## What the Task Command Center looks like

The **Task Command Center** page is a *dashboard*: the Tasks database lives on its **own page**,
and the dashboard shows **linked-view embeds** of it.

```
📋 Task Command Center  (page = dashboard)
├── Intro + daily-ritual + well-formed-task guidance
├── 📅 Today        (linked-view embed → Tasks)
├── 🗓️ This Week    (linked-view embed → Tasks)
├── 📥 Inbox        (linked-view embed → Tasks)
├── 📋 Tasks        (database, on its OWN page — NOT inline)
│        view tabs: 📥 Inbox · 📅 Today · 🗓️ This Week · 🌫️ Unscheduled · 🗂️ All Active · ✅ Done · Board · Calendar
└── 🌙 Daily Log    (database)  → one entry per day
```

Full schema, view configs, the template row, and rebuild steps:
`reference/command-center-structure.md`. Use it to verify, repair, or rebuild the structure —
then write any new IDs into `config.local.json`.

## 0. Load config first

All Notion IDs live in **`config.local.json`** (git-ignored, machine-specific).

1. Read `config.local.json` in this skill's directory.
2. If it's missing, copy `config.example.json` → `config.local.json` and ask the user to
   fill in their IDs — or offer to discover them by `notion-search` for
   "Task Command Center" and reading the page's child databases.

Never commit `config.local.json`. The committed `config.example.json` holds placeholders only.

## 1. The well-formed-task framework (apply to EVERY task you create)

A good task is: **outcome-first · atomic · action-oriented · context-complete · measurable.**

- **Title** = verb + object, one atomic action. If it needs the word "and", split it.
- Set the **Done-when** property — a testable success criterion.
- In the page **body**, fill: 🎯 Outcome · 📥 Inputs · 🚧 Constraints. Link prerequisites
  via the **Depends on** relation.

Full body template and worked examples: `reference/well-formed-task.md`.
When the user hands you a rough task ("I need to do X"), upgrade it into this shape — ask
only for what you genuinely can't infer.

## 2. Daily ritual

- **Morning** — triggers: "good morning", "daily update", "plan my day".
- **Evening** — triggers: "end of day", "wrap up", "evening review".

Exact step-by-step (including refreshing the Today view's date): `reference/daily-ritual.md`.
Summary:

- **Morning:** refresh the **Today & Overdue** view filter to today's date; pull tasks
  due ≤ today & not Done; propose an ordered plan; create today's **Daily Log** entry with a
  Morning Plan section.
- **Evening:** mark finished tasks **Done**; roll incomplete ones to tomorrow (update **Due**);
  fill the **End-of-Day Review** in today's Daily Log; set **Tasks Done** + **Mood**.

## 3. Adding & importing tasks

- **One task:** create it well-formed (section 1).
- **Import from Todoist:** the user pastes their list or a CSV (the Todoist MCP only exposes a
  UI widget here — there is no readable task tool, so import is always a manual paste). Bulk
  `notion-create-pages` into the Tasks data source, mapping: `p1..p4` → `P1..P4`, due date →
  **Due**, project/label → **Project**, and infer a **Done-when** where obvious.

## 4. Verify & repair the setup

Triggers: "check my task command center", "verify my setup", "audit my tasks".

Audit the live Notion against `reference/command-center-structure.md`:

1. Load IDs from `config.local.json`.
2. `notion-fetch` the **Tasks** data source — confirm every expected property exists with the
   right type/options, and that the **Board / List / Calendar / Today & Overdue** views exist and
   match their specified config (grouping, sorts, filters).
3. `notion-fetch` the **Daily Log** data source — confirm its properties.
4. `notion-fetch` the **Command Center page** — confirm Tasks (its own page) + Daily Log, the
   dashboard linked-view embeds (📅 Today / 🗓️ This Week / 📥 Inbox), and the 🧱 template row.
5. Report a checklist grouped by item: ✅ ok · ⚠️ drifted · ❌ missing.

Then offer to **repair** (ask before changing anything):

- Missing/renamed property → `notion-update-data-source` `ADD COLUMN` / `RENAME COLUMN`.
- Missing or misconfigured view → `notion-create-view` / `notion-update-view` per spec.
- Stale **📅 Today** filter → refresh to today's literal date; **🗓️ This Week** → end-of-week date.
  (Refresh both the DB view tabs AND the matching dashboard embeds — they're separate views.)
- Missing template row → recreate it.

**Never delete data to "fix" drift.** Flag unexpected extra properties/views/rows and let the
user decide. Make additive repairs by default; confirm before any rename or removal.

## 5. Schema & gotchas

Tasks data source properties:
`Task` (title) · `Status` (Not started / In progress / Done) · `Priority` (P1–P4) ·
`Project` (Inbox / Work / Personal / Errands) · `Due` (date) · `Notes` (text) ·
`Done-when` (text) · `Depends on` ↔ `Blocks` (dual self-relation).

- **Dates:** use expanded props — `date:Due:start`, `date:Due:end`, `date:Due:is_datetime`.
- **Relations:** value is a JSON array of related page URLs, e.g. `"[\"https://app.notion.com/p/…\"]"`.
- **Today view has no auto-"today":** the view DSL rejects the `today` keyword, so each morning
  update the filter with a literal date via `notion-update-view`:
  `FILTER "Due" <= "YYYY-MM-DD"`.
- **View DSL limits:** no relative-date keywords; `status`-type sub-filters silently drop;
  for self-relations, add ONE side only (two `DUAL` statements create duplicate columns).
- **Checkbox values** in updates use `__YES__` / `__NO__`; properties literally named `id`/`url`
  need a `userDefined:` prefix.
- **API-created boards group a status property "by group", not "by option".** `notion-create-view`
  with `GROUP BY "Status"` produces columns for the status *meta-groups* (To-do/In Progress/Complete)
  and renders awkwardly; the DSL can't set "by option". **Prefer having the user create the board in
  the Notion UI** (it groups by option = real Not started/In progress/Done columns), then just
  `SHOW`/`FILTER`-configure it via the API. Jason's Board is his own UI-made one.
- **A sort on a board disables drag-and-drop.** With any `SORT BY` active, Notion locks card
  order to the sort, so cards can't be dragged. Board views must have **no sort** (group-by uses
  manual order) for Kanban dragging to work. Keep Priority etc. as a shown card property instead.
- **Dashboard pattern — keep the source DB on its OWN page, then embed linked views.** This is the
  key architecture (validated 2026-06-09). API-created linked views (`notion-create-view` +
  `parent_page_id`) **DO render reliably — as long as the source database is NOT also rendered
  inline on the same page.** The earlier "Something went wrong" crashes were a *self-reference*:
  linked views pointing at a DB that was inline on that very page. Fix → set the DB
  `is_inline: false` (its own page), then add linked-view embeds on the dashboard. Jason's Command
  Center works this way: Tasks lives on its own page; the dashboard embeds 📅 Today / 🗓️ This Week /
  📥 Inbox. (Bonus: this also avoids the inline-database *lazy-render* quirk, where an inline board
  can show blank until you switch tabs.)
- **Embed lists, not boards, via the API.** API-made board views still group "by group" (see above),
  so embed **list** linked views; add a board to the dashboard via the UI `/linked` if wanted.

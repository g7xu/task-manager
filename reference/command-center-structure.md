# The Task Command Center — expected structure

This is what the Notion **Task Command Center** should look like. Use it to **verify** the setup,
**repair** anything missing, or **rebuild from scratch** (then write the new IDs into
`config.local.json`). All IDs come from `config.local.json`.

## Page tree

```
📋 Task Command Center  (page = dashboard — command_center_page_id)
├── Intro + "🔁 Daily ritual" + "✍️ Anatomy of a well-formed task" guidance
├── 📅 Today      (linked-view list embed → Tasks)
├── 🗓️ This Week  (linked-view list embed → Tasks)
├── 📥 Inbox      (linked-view list embed → Tasks)
├── 📋 Tasks      (database, OWN PAGE / not inline — tasks_database_id / tasks_data_source_id)
└── 🌙 Daily Log  (database — daily_log_database_id / daily_log_data_source_id)
```

> ✅ Architecture (validated 2026-06-09): the Tasks DB lives on its **own page**
> (`is_inline: false`); the dashboard shows **linked-view list embeds** of it. This is what makes
> embeds render reliably — the earlier "Something went wrong" crashes were a *self-reference*
> (a linked view pointing at a DB that was **inline on the same page**). Embed **lists** via API,
> not boards (API boards group by-group). See SKILL.md §5.

## Tasks database

**Properties**

| Property | Type | Notes |
| --- | --- | --- |
| `Task` | title | verb-led, atomic |
| `Status` | status | `Not started` / `In progress` / `Done` |
| `Priority` | select | `P1` red · `P2` orange · `P3` blue · `P4` gray |
| `Project` | select | `Inbox` · `Work` blue · `Personal` green · `Errands` yellow · `School` purple |
| `Due` | date | deadline |
| `Notes` | text | freeform |
| `Done-when` | text | measurable success criteria (shown on cards) |
| `Depends on` ↔ `Blocks` | relation | dual self-relation = dependencies |

**Views**

Tabs on the Tasks database page:

| View | Type | Config |
| --- | --- | --- |
| 📥 Inbox | list | `Project = Inbox` |
| 📅 Today | list | `Due <= "<today>"` (refresh daily), sort `Priority` ASC |
| 🗓️ This Week | list | `Due <= "<end-of-week>"` (refresh weekly), sort `Due` ASC |
| 🌫️ Unscheduled | list | `Due is empty` |
| 🗂️ All Active | list | sort `Due` ASC, show Status/Priority/Project/Due/Done-when |
| ✅ Done | list | `Status = Done` (set in UI) |
| Board | board | **UI-made**, group `Status` *by option*, **no sort** (so cards drag), show Priority/Project/Due/Done-when |
| Calendar | calendar | by `Due` |
| Default | table | all properties |

**Dashboard embeds** (on the Command Center page, separate linked views): 📅 Today · 🗓️ This Week ·
📥 Inbox — same filters as the matching tabs. Their date filters need the **same daily/weekly
refresh** as the DB tabs (they're independent views).

All list views also carry `FILTER "Task" != "🧱 Template — Well-formed task (duplicate me)"`.
**Status filters** (`is not Done` on the active views; `is Done` on ✅ Done) must be set **in the
UI** — the view DSL silently drops `status`-type filters.

**Template row** — a regular row titled `🧱 Template — Well-formed task (duplicate me)` whose body
holds the 🎯 Outcome / 📥 Inputs / 🚧 Constraints / 🔗 Dependencies / ✅ Done-when sections (see
`reference/well-formed-task.md`). Native Notion templates can't be created via the API, so this is
a duplicate-me row; the user may convert it to a real default template in the UI.

## Daily Log database

**Properties:** `Log` (title, e.g. "Wed, Jun 4 2026") · `Date` (date) · `Tasks Done` (number) ·
`Mood` (select: `On track` green / `Behind` orange / `Blocked` red).

**Body of each entry:** a `☀️ Morning Plan` section and a `🌙 End-of-Day Review` section. One entry
per day — always check for an existing entry before creating one.

## Rebuilding from scratch

If the page/databases don't exist:

1. `notion-create-pages` → the Command Center dashboard page.
2. `notion-create-database` → Tasks and Daily Log. Keep Tasks on its **own page**
   (`is_inline: false`) — do NOT render it inline on the dashboard.
3. `notion-create-view` → the Tasks list views (Inbox/Today/This Week/Unscheduled/All Active/Done)
   + Calendar. Add the **Board in the UI** (group by option, no sort) — API boards group by-group.
4. `notion-create-view` with `parent_page_id` (the dashboard page) → the **linked-view list embeds**
   (📅 Today, 🗓️ This Week, 📥 Inbox). These render reliably because Tasks is NOT inline here.
5. Create the template row.
6. Set the **Status** filters in the UI (API drops them). Write every new ID into `config.local.json`.

> When adding the `Depends on` self-relation, add **one** side only — two `DUAL` statements create
> duplicate columns that then need cleanup.

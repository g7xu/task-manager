# Daily Ritual

All IDs referenced here come from `config.local.json`.

## Morning plan

Triggers: "good morning", "daily update", "plan my day".

1. **Refresh the Today views.** The view DSL has no `today` keyword, so set a literal date on
   **both** the DB tab (`views.today_tab`) **and** the dashboard embed (`views.today_embed`) —
   they're separate views. For each, `notion-update-view`:
   `CLEAR FILTER; FILTER "Due" <= "YYYY-MM-DD"` (today's date).
   (Weekly, also refresh `views.this_week_tab` + `views.this_week_embed` to the end-of-week date.)
2. **Pull the work.** `notion-fetch` the `views.today_tab` (or query the Tasks data source)
   for tasks with `Due ≤ today` and `Status != Done`. Also surface anything `In progress`.
3. **Propose a plan.** Order by Priority (P1→P4) then Due. Flag tasks blocked by an unfinished
   **Depends on**. Keep it to a realistic few; call out overload.
4. **Write the Daily Log entry.** `notion-create-pages` into the Daily Log data source:
   - `Log` = e.g. "Wed, Jun 4 2026", `date:Date:start` = today, `Mood` = "On track".
   - Body: a **☀️ Morning Plan** section listing today's chosen tasks (as a checklist), and an
     empty **🌙 End-of-Day Review** section to fill in tonight.
   - If today's entry already exists, update it instead of creating a duplicate.

## End-of-day review

Triggers: "end of day", "wrap up", "evening review".

1. **Ask what got done** (or take the user's list).
2. **Mark finished tasks Done** — `notion-update-page` set `Status` = "Done" on each.
3. **Roll incomplete tasks forward** — for tasks due today that aren't done, update
   `date:Due:start` to tomorrow (or ask). Preserve their other fields.
4. **Complete today's Daily Log** — fill the End-of-Day Review: what got done, what rolled over,
   notes/blockers. Set `Tasks Done` (count) and `Mood` (On track / Behind / Blocked).
5. **Preview tomorrow** — one line on what's due next.

## Notes

- Keep the Daily Log to one entry per day; always check for an existing entry first.
- Be concise and encouraging in the log — it's a personal record, not a status report.

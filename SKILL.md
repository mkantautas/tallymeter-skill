---
name: tallymeter
description: Track billable time and run the kanban board in Tallymeter (tallymeter.com) – start/stop timers, log completed work, fix entries, check unbilled totals, and create, rank, move and comment on board tickets. Use when the user asks to track time, log hours or worked time, start/stop/check a timer, fix a logged entry's description or project, see what's unbilled, prepare data for a client invoice, or asks how many tickets and hours are left, what moved to a column today, to file or move a ticket, or what is on the board.
---

# Tallymeter time tracking

Tallymeter is a time tracker + invoicer for people who bill for their time. This skill lets you track work as it happens and log finished work, so it can become a client invoice later (invoices are generated in the web app at tallymeter.com).

## Auth

Every call needs a personal access token as `Authorization: Bearer $TALLYMETER_API_TOKEN`. The user creates tokens at <https://tallymeter.com/settings> (Settings → API tokens). Tokens are scoped: `timer`, `projects:read`, `entries:write`, `reports:read`, `board:read`, `board:write`. A 401 means bad/missing token; a 403 names the missing scope (or an expired plan). If the env var is unset, ask the user to create a token and export it — never paste tokens into files or code.

## Two ways to call — prefer MCP

**MCP (preferred, full capability):** if the `tallymeter` MCP server is connected, use its tools directly: `timer-status`, `start-timer`, `stop-timer`, `list-projects`, `log-time`, `update-entry`, `unbilled-summary`, `board-remaining`, `list-moves`, `list-tickets`, `get-ticket`, `create-ticket`, `update-ticket`, `move-ticket`, `comment-ticket`, `delete-ticket`. If it isn't connected, the user can add it:

```bash
claude mcp add --transport http tallymeter https://tallymeter.com/mcp \
  --header "Authorization: Bearer $TALLYMETER_API_TOKEN"
```

**REST (no MCP needed):** base URL `https://tallymeter.com/api/v1`. Endpoints: `GET /timer`, `POST /timer/start`, `POST /timer/stop`, `GET /projects`, `PATCH /entries/{id}`, `GET /tickets/{key}/summary`, `GET /board/remaining`, `GET /board/moves`, `GET /board/tickets`, `GET /board/tickets/{key}`, `DELETE /board/tickets/{key}`, `GET /me`. Note: logging *completed* entries and unbilled summaries are MCP-only – over plain REST you can track live via the timer but not backfill finished work.

```bash
# Start (stops any running timer first)
curl -X POST -H "Authorization: Bearer $TALLYMETER_API_TOKEN" -H "Content-Type: application/json" \
  -d '{"description": "ABC-1 fix login redirect", "project_id": 1}' \
  https://tallymeter.com/api/v1/timer/start

# Stop (idempotent — safe when nothing is running)
curl -X POST -H "Authorization: Bearer $TALLYMETER_API_TOKEN" https://tallymeter.com/api/v1/timer/stop
```

## Workflows

**Live tracking:** call `start-timer` the moment work begins (it auto-stops any running timer — never stop first "to be safe", you'd lose nothing but it's a wasted call), `stop-timer` when it ends. Check `timer-status` before assuming state.

**Logging finished work (MCP `log-time`):** description plus either `duration_minutes` or `started_at`/`ended_at` (ISO-8601 UTC). Use this when the user says "log 2h on X yesterday" or when you finish a task the user asked you to time.

**Fixing an entry (MCP `update-entry` / REST `PATCH /entries/{id}`):** rewrite a running or finished entry's description, project, `started_at` and/or `ended_at` in place — e.g. set the ticket key on a timer that was started without one, or correct a timer that kept running after the work was done. Setting `ended_at` on a running entry finishes it at that moment instead of now. Neither time may be in the future, and the end must be after the start. Never stop-and-restart a timer just to fix its description. Entry ids come back from every timer/log call. Entries already billed on an invoice are frozen (web app only).

**The board:** Tallymeter's tickets are where the time goes. A ticket carries the original estimate and the time entries carry the hours, so remaining work is a query rather than a reconciliation between two systems.

- **"How many tickets and hours are left?"** → `board-remaining`. It reports estimate minus logged, clamped to zero per ticket so one overrun cannot cancel out another ticket's real remaining work, with done and backlog columns excluded. Tickets with no estimate appear in `unestimated_tickets` and contribute no hours – say so, rather than letting the total imply they are free. `since_last_snapshot` gives the movement since the last nightly snapshot ("up 16h and 10 tickets since Monday").
- **"How many tickets moved to Review today?"** → `list-moves` with `column: "Review"` (REST `GET /board/moves?column=Review`). With no dates it reads today in the user's timezone; `since`/`until` (YYYY-MM-DD, both included) cover other days, and `from_column` filters on where a ticket came from. `total` is moves and `ticket_count` is tickets; each move says when, from, to, `now_in` (the column it is in now), who and `via` (board, api or mcp). A ticket sent on from Review the same day still counts as moved to Review. Never infer moves from `updated_at` or rank: any edit changes those.
- **Before you rank or amend anything**, call `list-tickets` for that column. Its rank order is the thing you are about to change, and a column you have not read is a column you will reorder by accident. Each ticket's `entered_column_at` says when it arrived in its column (null: not moved since it was created).
- **Paging:** `list-tickets` and `list-moves` return at most 200 per call. `total` is every match and `count` is this page; while `next_offset` is not null, call again with `offset` set to it.
- **Reading a ticket:** `get-ticket` (REST `GET /board/tickets/{key}`) returns the markdown description, comments, labels, parent, children, links, attachments, estimate against logged time and `history` (every move, rank, estimate, assignee, priority, type and label change, newest first). Read it before working on or amending a ticket; `list-tickets` returns no description.
- **Filing a ticket:** `create-ticket` with a title in the board's own voice, a markdown body, the column and an estimate in minutes. It is assigned to the token's user and returns the new key.
- **Amending someone else's ticket:** `update-ticket` with `append_description`, which adds to the body instead of replacing it. Pass only the fields you mean to change — anything omitted is left alone.
- **Ranking:** `move-ticket` with `above`, naming the ticket the card should sit on top of, which is how a column is actually ordered. Check the `column_order` it reads back: a rank that silently no-ops is the classic failure here.
- **Recording a decision or a finding:** `comment-ticket`, not an edit to somebody else's description. Whoever filed the ticket, is assigned it or has commented on it gets an email with the comment, so post one considered comment rather than several small ones.
- **Deleting a ticket:** `delete-ticket` (REST `DELETE /board/tickets/{key}`), only when the user asks for that ticket to be deleted. It is the board's own Delete: the card leaves the board, its comments, attachments and history are kept and it can be restored, and its key is never reused – the next ticket takes the next number.
- **Mentioning someone:** write `@Full Name` (or just the first name when no one else in the workspace shares it) in a comment or description. That person gets an email, and every later comment on the ticket. Mention only when you need that person; a mention is a notification, not a formality.
- **Layout:** one newline is a line break and a blank line starts a paragraph. A ticket reads best as a short numbered list, a bold label and a line or two per item; long paragraphs don't get read.
- **Checklists:** a markdown task list in the description (`- [ ] Write the migration`, `- [x] Done item`). The card shows `2/5`, and people tick items on the board. To tick one, rewrite that line through `update-ticket`; leave the rest of the body as it is.
- **Attach time to its ticket:** pass `ticket_id` when starting a timer. Time logged without one still bills, but it never counts toward that ticket's remaining work.

**Billing prep (MCP `unbilled-summary`):** totals of finished, not-yet-invoiced time with per-task breakdown. `GET /tickets/{key}/summary` gives tracked-vs-unbilled for one ticket key. The invoice itself is generated by the user in the web app — point them there, don't try to build one.

## Conventions that matter

- **Always give entries a meaningful description** — ticket key + what was done (`"JWIE-563 email integration: gmail mirror"`). Undescribed time is never billed.
- Timestamps are ISO-8601 UTC everywhere.
- Rate limit: 120 requests/minute. Errors are JSON with a `message` field.
- Give each agent its own named token – every entry records which agent tracked it, so a fleet's work stays attributable.

Full machine-readable docs: <https://tallymeter.com/llms.txt> and <https://tallymeter.com/openapi.json>.

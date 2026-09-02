# ZenSched Mystery-Shopping Reference Kit

A copy-pasteable setup for a small mystery shopping agency (an owner plus one to three schedulers and a pool of independent-contractor shoppers) or an in-house brand-audit team that wants an AI assistant to run shop assignment, GPS-verified store visits, evaluation forms with receipt-photo proof, shopper pay reconciliation, and client billing. ZenSched handles the live schedule, the shopper's phone app, GPS check-ins at the store, the evaluation form, and every submission. A small local database on your computer holds your clients, programs, the store list, the shopper pool, each shop's status and result, QA decisions, invoices, and pay runs.

**You do not need to know how to program or write SQL to use this.** You type plain English to your AI assistant ("set up the Burger Barn program from this brief", "give Dana the Austin shops next week", "pull this week's results", "invoice Burger Barn", "run shopper pay") and the AI does the work using two tools you set up once. Setup takes about 15 minutes and is the only technical part.

If you *are* a developer, skip to [For developers](#for-developers).

## What ZenSched verifies and what it does not — read this first

**What this kit is:** a way for a small agency to prove that a real person stood in the right store, inside the assigned window, for a plausible length of time, and filled in the client's questionnaire with a photographed receipt, and to turn those proofs into client invoices and shopper pay with an AI doing the clerical work.

**What it is not:**

- **ZenSched is not a questionnaire engine.** It stores a form (select, multi-select, text, number, photo, section) and its submissions. There is no branching logic you can rely on (conditionals are documented as web-only and may render unconditionally on the phone), no validation of a time typed as text, and no scoring. Scores, if you want them, are computed by the AI from a weights table in your local database.
- **ZenSched is not a scoring or reporting system.** You get the raw answers as JSON or a CSV download per form and date range. The AI summarizes; the client report is yours to write.
- **ZenSched is not a shopper marketplace.** It does not recruit, vet, or rate shoppers. You bring the shoppers; the kit tracks their reliability (no-shows, on-time check-ins, rejections) locally.
- **ZenSched is not a client portal.** Clients never log in. The AI produces a CSV and a plain-text invoice you send.
- **The proof is presence and time, not honesty.** GPS says the shopper was within the check-in radius of the store pin at check-in and check-out; the receipt photo and its printed time say they bought something; the form says what they claim they saw. The kit flags shops that check in outside the slot, skip the check-in, or run too short. It cannot tell you a narrative is made up.

If you need branching questionnaires, automatic scoring, or a client login, this kit is not for you yet. If you need enforceable proof-of-visit for a few dozen shops a month and an assistant that keeps the books, read on.

## What lives where

**ZenSched (source of truth for what happened, when, and where):**

- Locations (stores with GPS coordinates; the check-in radius is a policy setting)
- Workers (shoppers with the mobile app)
- Events (one per store per program wave, at most 60 days)
- Shifts (each assigned shop: one shopper, one store, one time slot, with a push notification)
- GPS punches (check-in / check-out with distance-from-the-store verification)
- One evaluation form per program, and every submission with its receipt photo
- Timesheets (rarely needed here; shoppers are paid per shop)

**Local SQLite database (`shop-ops.db`, on your computer):**

- Clients: contact, billing email, payment terms
- Programs: wave dates, daily window, allowed weekdays, quota per store, client fee, shopper fee, reimbursement cap, minimum minutes, the brief, which ZenSched form
- Stores: the client's store list with a normalized address so each store is geocoded once, its own time zone, and its ZenSched location id
- Program × store rows with the ZenSched event for that wave
- Shoppers: contact, home city, pay handle, ZenSched worker id
- Shops: one row per shop, open → assigned → completed / no-show / rejected / cancelled, with the check-in and check-out stamps, duration, receipt total, reimbursement, score, QA status and notes, invoiced and paid flags
- Optional question weights for scoring
- Client invoices and shopper payouts
- Your settings (default time zone, default shop length, invoice prefix and terms)

**Never duplicated:** the live schedule, punches, and the original submissions and receipt photos stay in ZenSched. The local database stores *references* to them plus the handful of values billing and QA need.

### Confidentiality note

Shoppers see the store, the time slot, the form, and any cancellation reason. They do not see the client contact, the fees, QA notes, scores, or who else is shopping. `SKILL.md` forbids the AI from putting any of those into a ZenSched field. Clients, in turn, get store-and-date results, never shopper names. The store label that crosses to ZenSched is the banner plus the street (`Burger Barn - Elm St`); no personal data is involved.

## How it works day to day

Your AI assistant has two sets of tools:

1. **ZenSched tools** (`location_create`, `event_create`, `form_create`, `shift_create`, `shift_list`, `shift_status`, `form_export`, ...) that talk to ZenSched over the internet.
2. **A SQLite tool** (`sqlite_query`, `sqlite_execute`) that reads and writes `shop-ops.db` on your computer.

When a client sends a brief, you paste it. The AI creates the client and program locally, adds each store to the store cache (geocoding only the ones it has never seen), turns the questionnaire into a ZenSched form, creates one 60-day-or-shorter event per store with the form attached, and generates the open shops from the quota. You say "give Dana the Austin shops next week, lunch window" and the AI picks dates inside the allowed weekdays and window, creates one shift per shop, and confirms. Dana sees the shops in the app, checks in at the store (GPS-verified), does the shop, fills in the form with a receipt photo, and checks out. Later you say "pull results" and the AI matches completed shifts to shops, records the punch times and receipt total, computes a score if you gave it weights, and leads with anything suspicious: a check-in 40 minutes after the slot, a 9-minute "visit", a form with no check-in at all. You approve or reject; approved shops flow to the client invoice and the shopper pay sheet. You never run SQL yourself. `SKILL.md` in this repo is the instruction sheet that teaches the AI how to do all of this; you paste it into your AI tool once.

## Setup

### 0. What you need

- **An AI tool that supports MCP.** These instructions use Claude Desktop (Windows or Mac). Cursor works too.
- **Node.js 20 or newer.** The SQLite tool runs on it. Download the LTS installer from [nodejs.org](https://nodejs.org/) and run it with the defaults. This is the only software install.
- You do **not** need the `sqlite3` command-line program, Python, or Git.

### 1. Make a folder for your data

Create a folder where the database will live and write down its full path. Examples:

- Windows: `C:\Users\YourName\shop-ops`
- Mac: `/Users/yourname/shop-ops`

The database file will be created automatically inside this folder the first time the AI uses it. It will hold client contracts and shopper pay details; keep it backed up.

### 2. Add both tools to your AI's config file

Open the MCP configuration file for your AI tool:

- **Claude Desktop, Windows:** `%APPDATA%\Claude\claude_desktop_config.json` (paste that into the File Explorer address bar)
- **Claude Desktop, Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json` (in Claude Desktop: Settings → Developer → Edit Config)
- **Cursor:** Settings → MCP → Add new global MCP server

Paste in the contents of `mcp.json.example` from this repo, then change one line, the `SQLITE_PATH`, to point at your folder from step 1 plus `\shop-ops.db` (Windows) or `/shop-ops.db` (Mac):

```json
{
  "mcpServers": {
    "zensched": {
      "url": "https://mcp.zensched.com/mcp",
      "headers": { "Authorization": "Bearer zsc_your_key_here" }
    },
    "shop-ops-db": {
      "command": "npx",
      "args": ["-y", "easy-sqlite-mcp"],
      "env": { "SQLITE_PATH": "/Users/yourname/shop-ops/shop-ops.db" }
    }
  }
}
```

**Windows path gotcha:** inside a JSON file every backslash must be doubled. Write `"C:\\Users\\YourName\\shop-ops\\shop-ops.db"`, not `"C:\Users\..."`. A single backslash will silently break the config.

**Leave `zsc_your_key_here` exactly as it is for now.** You do not have a key yet. The ZenSched tools that create your account work without one, and you will fill this in during step 3.

Save the file and **fully quit and reopen** your AI tool (on Mac, Cmd-Q; on Windows, right-click the tray icon → Quit). It only reads this file on startup.

### 3. Create your ZenSched account

In a new chat, type:

> Call `zensched_guide`, then call `account_create` with org_name "My Mystery Shopping Agency" (use my real business name if I told you one). Show me the `zsc_` key it returns.

Copy the `zsc_` key. Go back to the config file from step 2, replace `zsc_your_key_here` with your real key, save, and fully quit and reopen the AI tool again.

Some clients can adopt the key mid-session with `account_use_key`; you can ask the AI to try that to keep going immediately, but still update the config file so the key survives restarts. Keep the key private; it is the password to your account.

### 4. Create the database tables

Open `schema.sql` from this repo in any text editor, copy the whole thing, and paste it into the chat with this message in front of it:

> Create these tables in my shop-ops database. Run each statement one at a time using the SQLite tool, then list the tables to confirm.

The AI will run 46 statements and confirm the tables exist. The `shop-ops.db` file now exists in your folder with default settings (Central time, 45-minute shops, net-30 invoices) you can change.

If you happen to have the `sqlite3` command-line tool, `sqlite3 shop-ops.db < schema.sql` does the same thing, but it is not required.

### 5. Teach the AI the workflow

Paste the contents of `SKILL.md` into your AI tool as standing instructions. In Claude Desktop, create a Project and put it in the project instructions; in Cursor, save it as a rule. Then tell it your basics once:

> We're Northline Field Insights in Austin, Texas, Central time. Set the check-in radius to 150 m and give shoppers a 30-minute reminder.

It writes the name and default time zone to `settings` and sets the check-in policy on ZenSched (free). There is no form to create at this point; each client program gets its own form when you set the program up.

**Check-in radius.** ZenSched enforces the radius through the account's **policy** (`policy_update`), not per store, and with geofencing on it raises anything under 100 m to about 91 m (300 ft). The kit recommends 150 m because a mall entrance, a food court, or a big-box parking lot routinely puts the shopper 50–150 m from the geocoded pin. If a store's pin lands on the road rather than the building, the AI can move it (`location_update`, free) or have ZenSched pick the building (`location_refine`, $0.10). `remote_checkin` turns GPS verification off for **every** program in the account and should only be used if you run nothing but phone or web shops; see Troubleshooting.

**Reminders.** A `checkin_reminder_min_before` of 30 cuts no-shows. `checkout_reminder_min_after` (0–60) catches shoppers who forget to check out.

### 6. Funding (only when asked)

The first 200 ZenSched tool calls per day are free. Some things are metered: creating a store location (geocoding, $0.03, once per store ever), inviting a shopper ($0.25), each GPS-verified check-in or check-out ($0.10), and reading a submission ($0.05, or $0.15 when it has a photo; each submission is billed once, ever, so a later CSV export for the client is free). Creating forms, events, and shifts, and listing them, is free. When a metered call happens without funds, the AI gets a `payment_required` response and tells you how to add the $5 activation deposit, which is credited to your balance. You will not be charged without seeing this first.

Per completed shop that is about **$0.35** (two punches and one form read with a receipt), plus $0.03 for each store you have never shopped before. A 40-shop wave across 12 new stores is about $14.36; the AI states the cost before it spends.

## Using it

Everything after setup is plain English. Examples:

- "New program. Client is Burger Barn, contact Priya Nair. Lunch shops, one per store, weekdays 11 to 2, Sep 8 through Sep 30, $45 per shop, shoppers get $18 plus meal up to $12. Stores: (paste the list). Questionnaire: (paste it). Score it: greeting 2, accuracy 3, friendliness 4+ is 2, upsell 1, clean 2."
- "Invite Dana Ruiz, dana@example.com, Austin, PayPal same email."
- "Give Dana the three Austin shops next week, lunch window."
- "Fill the open shops." (the AI proposes shopper-to-store matches by home city and reliability, and asks)
- "Pull this week's results."
- "Approve Dana's two. Reject Marcus, the client won't take anything ordered after 2. Reopen it."
- "How's the Burger Barn program doing?"
- "Send Burger Barn their results and invoice."
- "Run shopper pay."
- "Who's my most reliable shopper?"
- "Move Dana's Congress Ave shop to Thursday 12:30."
- "Marcus can't do Wednesday, give it to someone else."
- "Burger Barn paid INV-2026-0001."

See `QUICKSTART.md` for the first-wave walkthrough and `example-workflow.md` for exactly which tools the AI calls behind each of these.

### What "invoice" means here

"Invoice Burger Barn" records the invoice in your database (number, date, due date from the client's payment terms, shop count, fees, reimbursements, which shops) and the AI writes out a plain-text invoice you can paste into an email, with a line per approved shop (date, store code and label, fee, reimbursement) and a note that every shop was GPS-verified with a timestamped receipt. It does **not** generate a PDF, email it for you, or collect payment. Shopper names never appear on it. When the client pays, tell the AI ("Burger Barn paid INV-2026-0001") and it marks it paid. Reimbursements are passed through at cost by default; tell the AI if your fee is all-inclusive.

### What "shopper pay" means here

"Run shopper pay" totals approved shops per shopper (fee plus reimbursement, capped at the program's cap), records a payout row with the shop list, and writes a pay sheet with each shopper's pay handle (PayPal email, Venmo, bank nickname). You pay them outside the kit and say "paid". Rejected and no-show shops are not on the sheet; if you want to pay one anyway, say so. The kit does not handle 1099s or taxes.

### What "results" means here

"Send Burger Barn their results" runs `form_export` for the program's form and wave dates and gives you a CSV download link plus a summary from the local database (shops per store, scores, what was rejected). The CSV includes a `worker_name` column and any rejected submissions; the AI reminds you to remove both before it goes to the client.

## Mobile app for shoppers

- **Android:** [Google Play](https://play.google.com/store/apps/details?id=com.zensched.app)
- **iOS:** [TestFlight](https://testflight.apple.com/join/Wp51m5Yq)

When you invite a shopper, they get an email, install the app, and can immediately see their assigned shops, check in and out with GPS verification, and fill in the evaluation form. The receipt photo is a required field; the form cannot be submitted without it. The scenario ("order a combo, ask about dessert") is something you tell the shopper yourself; ZenSched shows them the store, the slot, and the form.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| AI says it has no ZenSched tools | Config file not saved, or the app was not fully restarted | Check the JSON is valid (paste it into [jsonlint.com](https://jsonlint.com)), then quit and reopen the app |
| AI says it has no SQLite / `shop-ops-db` tools | Node.js not installed, or bad `SQLITE_PATH` | Install Node.js LTS; on Windows check every backslash is doubled |
| `SQLITE_PATH` points nowhere / "unable to open database" | Folder from step 1 does not exist | Create the folder; the file is created automatically but the folder is not |
| ZenSched tools return an auth error | Key still says `zsc_your_key_here`, or was pasted with a space | Re-paste the key, restart |
| `payment_required` | Metered call with no balance | Follow the instructions in the response; $5 deposit |
| Denver shop created at the wrong hour | Store has the wrong `tz_offset` | "Set the Pearl St store to Mountain time (-06:00)"; the AI fixes the store and updates the shift |
| AI refuses a program longer than 60 days | Working as intended; ZenSched events are capped at 60 days | Ask for it as monthly waves; the AI creates one program row and one set of events per wave |
| Shopper's check-in not GPS-verified at a mall or big-box store | Shopper was outside the policy radius, or the pin is on the road | "Set the check-in radius to 200 m" (`policy_update`, account-wide), or "move the Congress Ave pin onto the building" (`location_update`, free), or `location_refine` ($0.10). Do **not** ask for `remote_checkin`: it turns GPS proof off for every program in the account |
| You genuinely run phone or web shops | Those cannot have a GPS punch | Keep policy 0 geofenced for in-person work; ask the AI to give phone programs their own brand and policy with `remote_checkin` rather than flipping it account-wide |
| Shopper forgot to check out | Shift still `checked_in` | Tell the AI to use the form's submission time as check-out; ask for a check-out reminder (`checkout_reminder_min_after`) |
| Shopper does not see the form | Form not assigned to that store's event | "Attach the Lunch Experience form to the Pearl St event" (`form_assign`) |
| "What was wrong?" shows even when the order was accurate | Conditional fields are web-only on ZenSched | Harmless; shoppers leave it blank. The label says "(if not accurate)" |
| A shop is flagged but the client is fine with it | Working as intended; flags are for you, not automatic rejections | Approve it and put the reason in the QA note |
| Same store pasted twice in a brief | Address written differently | The AI normalizes addresses; if it still created two stores, say "these are the same store" and it merges them (one geocode is wasted, $0.03) |
| Invoice total includes meals you don't bill | Reimbursement pass-through is the default | "Our fee is all-inclusive"; the AI drops reimbursements from client invoices |
| AI asks you to run SQL yourself | It does not have `SKILL.md` loaded | Re-paste `SKILL.md` as project instructions |

If something is confusing or broken in ZenSched itself, ask the AI to call `feedback_submit` with a description. It is free, needs no account, and a human reads every submission.

## For developers

**Architecture.** Two MCP servers, no application code. The agent is the integration layer; `SKILL.md` is the spec it follows. ZenSched is authoritative for operations (locations, events, shifts, punches, forms, submissions); SQLite is authoritative for the commercial model (clients, programs, fees, quotas, store cache, shopper pool, shop lifecycle, QA, billing, payouts); each side stores only the other's IDs plus the few per-shop values billing needs (`submission_dc_id`, punch stamps, receipt total, reimbursement, score). The confidentiality boundary is enforced by data placement (client and fee columns exist only locally) and by `SKILL.md` rule 1; there is no technical control stopping a misbehaving agent, so review the rule if you swap models.

**Data model decisions.**

- **Stores are a cache, not a per-program list.** `stores` is `UNIQUE (client_id, normalized_address)`; `SKILL.md` gives the normalization recipe. One `location_create(name=<store_label>, street_address=..., checkin_radius_m=<settings default>, idempotency_key="loc-store-{store_id}")` per store, ever ($0.03), stored on `stores.zensched_location_id`. `checkin_radius_m` on `location_create` is informational; the enforced radius is `policy_update(0, '{"checkin_radius_m": N}')`, and with geofencing on the platform raises values under 100 m to 300 ft.
- **Per-store time zone.** `stores.tz_offset` is `CHECK`-constrained to `[+-]HH:MM`; `settings.timezone_offset` is only the default for new stores. `shops.scheduled_start` / `scheduled_end` are store-local wall-clock strings (`YYYY-MM-DDTHH:MM:SS`, `CHECK`-constrained to have no offset) and `shops_upcoming` builds `start_iso` / `end_iso` as `scheduled_start || tz_offset`. `checkin_at` / `checkout_at` carry an explicit offset (whatever ZenSched returned, or UTC `+00:00` if converted from epoch seconds), and every comparison against the slot uses `julianday(scheduled_x || tz_offset)`, so late-check-in detection is correct for a Denver store scheduled by a Texas agency.
- **One event per store per wave.** `programs.wave_start` / `wave_end` are the event dates; a `CHECK` rejects `wave_end` more than 59 days after `wave_start`, which forces long engagements into one program row per wave (the kit's answer to the 60-day event cap; there is no rolling `event_needs_roll` machinery because a wave never outlives its events). `program_stores` (`UNIQUE (program_id, store_id)`) holds `zensched_event_id` and `event_valid_until`; `event_create(location_id, title="{banner} {program} - {street}", start_date=wave_start, end_date=wave_end, idempotency_key="event-ps-{program_store_id}-{YYYYMMDD}")`, then `form_assign(form_id, event_id=...)`. `shops_upcoming.needs_event` flips when the row has no event or `event_valid_until` is before the shop date; `needs_location` when the store was never geocoded.
- **One form per program**, id on `programs.zensched_form_id`, created with `form_create(title, fields_json, idempotency_key="form-program-{program_id}")`. `SKILL.md` contains the translation table (yes/no → `select`, rating → `select` of `"1".."5"`, narrative → `textarea`, receipt → required `photo` with `max_images` 2, time → `text`, minutes → `number`) and a worked example validated against ZenSched's form validator (15 fields). Every field carries an explicit `identifier`. Option keys are derived by ZenSched from labels (lowercase, non-alphanumerics → `_`): `2-5 min` → `2_5_min`, `Under 2 min` → `under_2_min`, `Trash overflowing` → `trash_overflowing`. One `show_if` (`order_issue` shown when `order_accuracy equals no`); conditionals are documented as web-only. **No `signature` field**: on ZenSched a signature replaces the Submit button, which is wrong for a shopper submitting alone.
- **Scoring is local and optional.** `question_weights` (`UNIQUE (program_id, identifier)`) holds `weight`, `expected_value` (an option key or number as text), and `match_rule` (`equals | not_equals | gte | lte | contains | not_contains`, `CHECK`-constrained). The agent computes `100 × Σ passing weight ÷ Σ weight` and writes `shops.score`. ZenSched is never asked to score.
- **Shop lifecycle.** `shops.status` is `open | assigned | completed | no_show | rejected | cancelled`; `qa_status` is `pending | approved | rejected`; both `CHECK`-constrained. `zensched_shift_id` is `UNIQUE`, and the rule is never to reuse a row that has one: no-shows, rejections, and cancellations keep their row (for `shopper_reliability`) and the agent inserts a fresh `open` row. `program_progress` therefore counts approved shops against `SUM(program_stores.shops_required)`, not against row counts.
- **Fraud flags are a view, not a status.** `shops_flagged` returns completed shops with `checkin_late` (check-in after `scheduled_end`), `checkin_early` (more than 15 minutes before `scheduled_start`), `no_checkin` (completed with no punch), `short_visit` (`duration_minutes < programs.min_minutes`), and `minutes_after_slot_end`. A flag is information for QA; the owner decides, and approving a flagged shop with a note is a supported path.
- `duration_minutes` is filled by two triggers (`AFTER INSERT`, `AFTER UPDATE OF checkin_at, checkout_at`) as `round((julianday(out) − julianday(in)) × 1440)` whenever it is NULL and both stamps exist; an explicit value is never overwritten.
- **Money.** `shops_to_invoice` groups approved, uninvoiced shops per client with `SUM(client_fee)`, `SUM(reimbursement)`, and their total; `shopper_pay_due` does the same per shopper with `SUM(shopper_fee) + SUM(reimbursement)` and a JSON `shop_ids` array. Fees are read from the program at query time, not snapshotted; `SKILL.md` tells the agent to change fees only between runs or by starting a new wave. `invoices.invoice_number` is auto-assigned by trigger as `{prefix}-{YYYY}-{0001}`; `invoices_outstanding` adds `days_overdue` and an `aging_bucket` (`current | 1-30 | 31-60 | 61-90 | 90+`).
- `shopper_reliability` aggregates with `FILTER` clauses (SQLite ≥ 3.30): shops given, completed, approved, no-shows, rejected, upcoming, `on_time_pct` over completed and rejected shops with a punch (within 15 minutes before the slot start and before the slot end), `avg_score`, last completed date. Shoppers with no shops get zeros and a NULL percentage.
- `PRAGMA foreign_keys = ON` is in `schema.sql` and `SKILL.md` tells the agent to run it per session; SQLite does not persist it. Deleting a client cascades to programs, stores, program-store rows, shops, question weights, and invoices; deleting a shopper sets `shops.shopper_id` NULL and cascades payouts.

**Idempotency keys.** Deterministic, derived from local IDs so a retried or re-run agent turn cannot duplicate:

- location: `loc-store-{store_id}`
- event: `event-ps-{program_store_id}-{YYYYMMDD wave start}`
- shift: `shift-shop-{shop_id}` (one shift per shop row; a redo is a new row)
- cancel: `cancel-shift-{shift_id}`
- worker: `worker-{email}`
- form: `form-program-{program_id}`; assignment: `assign-form-{program_id}-{event_id}`

ZenSched caches idempotent responses for 24 hours.

**Timestamps.** `shift_create` takes `start` and `end` in ISO 8601 with an explicit offset, never `Z`. The offset is the **store's** (`stores.tz_offset`), which is why `shops_upcoming` builds the strings and the agent is told not to. `shift_list` takes `date_from` / `date_to` as `YYYY-MM-DD`; `form_export` / `form_submissions` take `since` / `until` as dates; `timesheet_export` takes `period="YYYY-MM-DD:YYYY-MM-DD"`.

**Metered reads.** `form_submissions` and `form_export` bill $0.05 per submission read ($0.15 with media; the receipt photo makes every shop a media read), once per submission ever; replays, including the CSV export for the client after the JSON pull for QA, are free. `form_export(format="json")` for a wave is the intended pull; `form_submissions(form_id, event_id=...)` for one store. `shift_list`, `shift_status`, `event_get`, and `timesheet_export(mode="hours"|"raw")` are free.

**SQLite MCP server.** `mcp.json.example` uses [`easy-sqlite-mcp`](https://github.com/chenkumi/easy-sqlite-mcp) (Node, `better-sqlite3`, `SQLITE_PATH` env var). Its `sqlite_execute` calls `prepare()`, so it accepts **one statement per call**; `schema.sql` is written so every statement stands alone and is idempotent. Any SQLite MCP server with read and write tools will work; adjust the tool names in `SKILL.md`.

**Schema test.** The schema was verified by splitting the file into its 46 statements with `sqlite3.complete_statement` and executing each individually (as the MCP server does) twice for idempotency (seed rows not duplicated), then exercising: all 10 tables, 9 views, and 8 triggers present; every view on an empty database; `UNIQUE (client_id, normalized_address)` (and the same address allowed for a different client), `UNIQUE (program_id, store_id)`, `UNIQUE` on `zensched_shift_id`, `shoppers.email`, `shoppers.zensched_worker_id`, and `(program_id, identifier)`; `shops_upcoming` building `start_iso` / `end_iso` from a `-05:00` and a `-06:00` store, the `shift-shop-{id}` key, `needs_location`, and `needs_event` flipping on a stale `event_valid_until`; `shops_open` days-left math; `shops_overdue`; `shops_flagged` catching a 40-minute-late check-in, a short visit, an early check-in, and a completed shop with no punch while ignoring on-time shops and a 10-minute-early one; the duration trigger on insert and update, across mixed offsets, refilling after NULL, and never overwriting an explicit value; `program_progress` counts and percentage before and after QA; `shops_to_invoice`, `shopper_pay_due`, and `shopper_reliability` math including a shopper with no shops; invoice numbering with prefix and year and an explicit number kept; all five aging buckets and `days_overdue`; every `CHECK` (wave length both ways, window times, weekday mask length and characters, quota, program status, `tz_offset` format including rejecting `Z`, `scheduled_*` format rejecting an offset and a space separator, shop status, QA status, match rule, weight); the five `updated_at` triggers; foreign keys, cascade from client and program-store, and set-null from shopper. 94 checks, all passing. The evaluation form in `SKILL.md` was run through ZenSched's `_validate_fields` and accepted with the option keys listed there.

## Support

- ZenSched docs: <https://www.zensched.com/docs/>
- Tool reference: <https://www.zensched.com/docs/tools/>
- Feedback: ask your AI to call `feedback_submit` (categories: `bug`, `friction`, `missing_capability`, `docs`, `billing`, `feature`, `other`)

## License

MIT. See `LICENSE`.

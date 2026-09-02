# Quickstart

Setup is about 15 minutes, once. After that everything is plain English to your AI. Each step below tells you what to do and, where relevant, exactly what to type to the AI.

You need: Claude Desktop (or Cursor) and [Node.js LTS](https://nodejs.org/) installed. Nothing else.

Before you start, read the "What ZenSched verifies and what it does not" section of `README.md`. Short version: ZenSched proves a shopper was at the store, inside the slot, for a plausible time, and collected the form with a receipt photo. It does not score, branch, or report; the AI and your local database do that.

## 1. Make a data folder

Create a folder such as `C:\Users\YourName\shop-ops` (Windows) or `/Users/yourname/shop-ops` (Mac). Note the full path. It will hold client fees and shopper pay details, so keep it backed up.

## 2. Add the two tools to your AI's config

Open the config file:

- **Claude Desktop, Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Claude Desktop, Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Cursor:** Settings → MCP → Add new global MCP server

Paste this in and fix only the `SQLITE_PATH` line to match your folder from step 1:

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

- On Windows, double every backslash: `"C:\\Users\\YourName\\shop-ops\\shop-ops.db"`.
- Leave `zsc_your_key_here` as it is. You get the real key in the next step.

Save, then **fully quit and reopen** the AI app.

## 3. Create your ZenSched account

Type to the AI:

> Call zensched_guide, then account_create with org_name "My Mystery Shopping Agency". Show me the zsc_ key.

Copy the key into the config file in place of `zsc_your_key_here`. Save. Quit and reopen the app once more. (You can also ask the AI to call `account_use_key` with the key to continue right away, but update the file anyway so it sticks.)

## 4. Create the database tables

Copy the full contents of `schema.sql` and paste it into the chat with this line above it:

> Create these tables in my shop-ops database. Run each statement one at a time with the SQLite tool, then list the tables to confirm.

## 5. Give the AI its instructions

Paste `SKILL.md` into the AI as standing instructions (Claude Desktop: a Project's instructions; Cursor: a rule). Then:

> We're Northline Field Insights in Austin, Texas, Central time. Set the check-in radius to 150 m and give shoppers a 30-minute reminder.

The AI saves your name and default time zone and sets the ZenSched check-in policy (free): shoppers must be within 150 m of the store pin, GPS verification stays on, and they get a reminder before each shop. There is no form yet; each client program gets its own.

## 6. Set up your first program

Paste the client's brief. For example:

> New program. Client is Burger Barn, contact Priya Nair, priya@burgerbarn.example, billing ap@burgerbarn.example, net 30. Lunch shops, one per store, weekdays 11 to 2, Sep 8 through Sep 30. $45 per shop to them, shoppers get $18 plus meal up to $12, shops at least 20 minutes. Stores: 1042, 118 Elm Street, Austin TX 78704 / 1057, 2400 S Lamar Blvd, Austin TX 78704 / 1088, 615 Congress Ave, Austin TX 78701 / 2210, 900 Pearl Street, Denver CO 80203. Scenario: order a combo at the counter, ask about dessert, eat in. Questionnaire: arrival time; greeted within 30 seconds; wait to order; order accurate and what was wrong; staff friendliness 1–5; offered a combo or dessert; cleanliness issues; receipt photo; receipt total; short narrative.

Behind the scenes the AI saves the client and program locally, adds the four stores to its store cache (the Denver one in Mountain time), turns the questionnaire into a ZenSched form with a required receipt photo (free), asks you before geocoding the four stores ($0.12, may trigger the $5 activation deposit the first time), creates one event per store for the wave with the form attached, and creates four open shops. You just see a confirmation.

## 7. Invite your shoppers

> Invite Dana Ruiz, dana@example.com, Austin, PayPal same email. And Marcus Lee, marcus@example.com, Denver, Venmo @marcuslee.

Each gets an email ($0.25), installs the app ([Android](https://play.google.com/store/apps/details?id=com.zensched.app) / [iOS TestFlight](https://testflight.apple.com/join/Wp51m5Yq)), and activates. Send them the scenario yourself; ZenSched shows them the store, the slot, and the form.

## 8. Assign the shops

> Give Dana the three Austin shops next week, lunch, and Marcus the Denver one on Tuesday.

The AI picks weekdays inside the 11–2 window, creates one shift per shop on ZenSched in each store's own time zone, and confirms by shopper and day. Dana gets a push notification per shop with the form attached. She checks in at the store (GPS-verified), does the shop, fills in the form, photographs the receipt, and checks out.

Or say "fill the open shops" and the AI proposes who should take what, by home city and track record, and waits for your yes.

## 9. After the shops are done

> Pull this week's results.

The AI pulls the completed, GPS-verified shifts and their punch times from ZenSched (free), then the submissions (metered, so it tells you the cost first, about $0.15 per shop), records receipt totals and scores, and leads with anything suspicious: a check-in 40 minutes after the slot ended, a form with no check-in, a 9-minute visit. Missed shifts become no-shows and the shop is reopened.

> Approve Dana's two. Reject Marcus, the client won't take anything ordered after 2. Reopen it.

Approved shops go on the client invoice and the shopper pay sheet. Rejected ones are reopened for someone to redo.

> How's the Burger Barn program doing?

Required, assigned, completed, approved, percent complete, days left. From the local database, free.

> Send Burger Barn their results and invoice.

A CSV download of the wave's submissions (already-read submissions are not billed again) plus a plain-text invoice with one line per approved shop and a note that every shop was GPS-verified with a timestamped receipt. No shopper names.

> Run shopper pay.

A pay sheet per shopper (fee plus reimbursement, with their pay handle). You pay them through PayPal or Venmo and say "paid".

> Burger Barn paid INV-2026-0001.

Marks it paid.

## What next

- `README.md` for the full explanation, what ZenSched does and does not verify, troubleshooting table, and developer notes
- `example-workflow.md` to see the exact tool calls behind each step above

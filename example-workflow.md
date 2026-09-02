# Example Workflow: What the AI Does Behind Each Request

This shows the exact tool calls the agent makes for the first wave of a program, following `SKILL.md`. The owner only types the quoted lines; everything else is the agent's work. Assumes setup from `QUICKSTART.md` is complete (both MCP servers configured, `schema.sql` loaded, `SKILL.md` given as instructions).

IDs and responses are illustrative. ZenSched IDs are integers. Watch what does **not** cross to ZenSched: the client contact, the fees, the shopper's pay handle, QA notes, scores. ZenSched sees `Burger Barn - Elm St`, a street address, an event title, a form, and two shopper accounts. Watch also the Denver store: its shifts are created in `-06:00` while the agency runs on `-05:00`.

The agency is **Northline Field Insights**, Austin, Texas, Central time. Today is Tuesday, September 8, 2026.

## Session start (every session)

```
sqlite_execute: PRAGMA foreign_keys = ON;
sqlite_query:   SELECT key, value FROM settings;
  → business_name = My Mystery Shopping Agency, timezone_offset = -05:00, invoice_due_days = 30,
    invoice_prefix = INV, default_shop_minutes = 45, default_checkin_radius_m = 150
sqlite_query:   SELECT * FROM shops_overdue;                              → (none)
sqlite_query:   SELECT * FROM shops_flagged WHERE qa_status = 'pending';  → (none)
```

## "We're Northline Field Insights in Austin, Central time. Set the check-in radius to 150 m and give shoppers a 30-minute reminder."

```
sqlite_execute: UPDATE settings SET value = 'Northline Field Insights' WHERE key = 'business_name';
sqlite_execute: UPDATE settings SET value = '-05:00' WHERE key = 'timezone_offset';

policy_get: policy_id 0
  → settings: { geofence_enabled: true, require_on_site: true, remote_checkin: false, checkin_radius_m: 91, ... }

policy_update:
  policy_id: 0
  settings_json: "{\"checkin_radius_m\": 150, \"checkin_reminder_min_before\": 30}"
  → settings: { checkin_radius_m: 150, checkin_reminder_min_before: 30, geofence_enabled: true, require_on_site: true, ... }

sqlite_execute: UPDATE settings SET value = '150' WHERE key = 'default_checkin_radius_m';
```

> Saved. Shoppers must check in within 150 m of the store pin (that covers a mall entrance or a big parking lot) and get a reminder 30 minutes before each shop. GPS verification stays on for every program; that is the proof your clients pay for. Central time is the default for new stores, but each store keeps its own time zone.

## "New program. Client is Burger Barn, contact Priya Nair, priya@burgerbarn.example, billing ap@burgerbarn.example, net 30. Lunch experience shops, one per store, any weekday 11 to 2, Sep 8 through Sep 30. $45 per shop to them, shoppers get $18 plus meal reimbursed up to $12. Shops should take at least 20 minutes. Stores: 1042, 118 Elm Street, Austin TX 78704 / 1057, 2400 S Lamar Blvd, Austin TX 78704 / 1088, 615 Congress Ave, Austin TX 78701 / 2210, 900 Pearl Street, Denver CO 80203. Shopper scenario: order a combo at the counter, ask about dessert, eat in. Questionnaire: arrival time; greeted within 30 seconds; wait to order (under 2, 2–5, over 5 min); order accurate, and what was wrong if not; staff friendliness 1–5; offered a combo or dessert; cleanliness issues (tables, floor, restroom, trash, condiment station); receipt photo; receipt total; a 3–5 sentence narrative. Score it: greeting 2 points, accuracy 3, friendliness 4+ is 2, upsell 1, no cleanliness issues 2."

```
sqlite_query:   SELECT client_id FROM clients WHERE client_name = 'Burger Barn';   → (none)
sqlite_execute:
  INSERT INTO clients (client_name, contact_name, contact_email, billing_email, payment_terms_days)
  VALUES ('Burger Barn', 'Priya Nair', 'priya@burgerbarn.example', 'ap@burgerbarn.example', 30);
  → lastInsertRowid = 1

sqlite_execute:
  INSERT INTO programs (client_id, program_name, wave_start, wave_end, window_start_time, window_end_time, allowed_weekdays,
                        quota_per_store, client_fee, shopper_fee, reimbursement_cap, min_minutes, questionnaire_notes, shopper_brief, status)
  VALUES (1, 'Burger Barn Lunch - Sep', '2026-09-08', '2026-09-30', '11:00', '14:00', '1111100',
          1, 45.00, 18.00, 12.00, 20,
          'Client contact Priya Nair. Lunch experience, counter service. Client scoring: greeting 2, accuracy 3, friendliness>=4 2, upsell 1, clean 2.',
          'Order a combo at the counter, ask about dessert, eat in. Keep the receipt; the receipt photo is required. Arrive inside your assigned slot.',
          'draft');
  → lastInsertRowid = 1
```

Stores: normalize, look up, insert. Denver gets `-06:00`.

```
sqlite_query: SELECT store_id, zensched_location_id FROM stores WHERE client_id = 1 AND normalized_address = 'burger barn 118 elm st austin tx 78704';   → (none)
sqlite_execute:
  INSERT INTO stores (client_id, banner, store_code, address, city, region, country, postal, normalized_address, tz_offset, store_label)
  VALUES (1, 'Burger Barn', '1042', '118 Elm Street', 'Austin', 'TX', 'US', '78704', 'burger barn 118 elm st austin tx 78704', '-05:00', 'Burger Barn - Elm St');   → 1
sqlite_execute: INSERT INTO stores (...) VALUES (1, 'Burger Barn', '1057', '2400 S Lamar Blvd', 'Austin', 'TX', 'US', '78704', 'burger barn 2400 s lamar blvd austin tx 78704', '-05:00', 'Burger Barn - S Lamar Blvd');   → 2
sqlite_execute: INSERT INTO stores (...) VALUES (1, 'Burger Barn', '1088', '615 Congress Ave', 'Austin', 'TX', 'US', '78701', 'burger barn 615 congress ave austin tx 78701', '-05:00', 'Burger Barn - Congress Ave');   → 3
sqlite_execute: INSERT INTO stores (...) VALUES (1, 'Burger Barn', '2210', '900 Pearl Street', 'Denver', 'CO', 'US', '80203', 'burger barn 900 pearl st denver co 80203', '-06:00', 'Burger Barn - Pearl St');   → 4

sqlite_execute: INSERT INTO program_stores (program_id, store_id, shops_required) VALUES (1, 1, 1);   → 1
sqlite_execute: INSERT INTO program_stores (program_id, store_id, shops_required) VALUES (1, 2, 1);   → 2
sqlite_execute: INSERT INTO program_stores (program_id, store_id, shops_required) VALUES (1, 3, 1);   → 3
sqlite_execute: INSERT INTO program_stores (program_id, store_id, shops_required) VALUES (1, 4, 1);   → 4
```

The questionnaire becomes this program's form (free):

```
form_create:
  title: "Burger Barn Lunch Experience"
  idempotency_key: "form-program-1"
  fields_json: (the JSON below, passed as one string)
```

```json
[
  {"type": "section", "label": "Arrival", "text": "Complete this before you leave the parking lot. Answer from what you saw, not what you expected."},
  {"type": "text", "label": "Time you entered (HH:MM)", "identifier": "arrival_time", "required": true, "placeholder": "12:07"},
  {"type": "select", "label": "Greeted within 30 seconds?", "identifier": "greeted_30s", "required": true, "options": ["Yes", "No"]},
  {"type": "select", "label": "Wait time to order", "identifier": "line_wait", "required": true, "options": ["Under 2 min", "2-5 min", "Over 5 min"]},
  {"type": "section", "label": "Service"},
  {"type": "select", "label": "Order accurate?", "identifier": "order_accuracy", "required": true, "options": ["Yes", "No"]},
  {"type": "textarea", "label": "What was wrong?", "identifier": "order_issue",
   "show_if": {"field": "order_accuracy", "op": "equals", "value": "no", "action": "show"}},
  {"type": "select", "label": "Staff friendliness (1-5)", "identifier": "staff_friendliness", "required": true, "options": ["1", "2", "3", "4", "5"]},
  {"type": "select", "label": "Were you offered a combo or dessert?", "identifier": "upsell_offered", "required": true, "options": ["Yes", "No"]},
  {"type": "section", "label": "Cleanliness"},
  {"type": "multi_select", "label": "Cleanliness issues observed", "identifier": "clean_issues", "required": true,
   "options": ["None", "Dirty tables", "Dirty floor", "Restroom", "Trash overflowing", "Condiment station"]},
  {"type": "section", "label": "Proof"},
  {"type": "photo", "label": "Receipt photo (required)", "identifier": "receipt", "required": true, "max_images": 2},
  {"type": "number", "label": "Receipt total", "identifier": "receipt_total", "required": true},
  {"type": "textarea", "label": "Narrative (3-5 sentences, what happened)", "identifier": "narrative", "required": true}
]
```

```
  → form_id: 301, fields: [ ...15 fields echoed with identifiers... ]

sqlite_execute: UPDATE programs SET zensched_form_id = 301 WHERE program_id = 1;

sqlite_execute: INSERT INTO question_weights (program_id, identifier, weight, expected_value, match_rule) VALUES (1, 'greeted_30s', 2, 'yes', 'equals');
sqlite_execute: INSERT INTO question_weights (program_id, identifier, weight, expected_value, match_rule) VALUES (1, 'order_accuracy', 3, 'yes', 'equals');
sqlite_execute: INSERT INTO question_weights (program_id, identifier, weight, expected_value, match_rule) VALUES (1, 'staff_friendliness', 2, '4', 'gte');
sqlite_execute: INSERT INTO question_weights (program_id, identifier, weight, expected_value, match_rule) VALUES (1, 'upsell_offered', 1, 'yes', 'equals');
sqlite_execute: INSERT INTO question_weights (program_id, identifier, weight, expected_value, match_rule) VALUES (1, 'clean_issues', 2, 'none', 'contains');
```

Agent (first metered call this session, rule 9):

> Four new stores cost $0.12 to geocode now. Each completed shop will cost about $0.35 on ZenSched (two GPS punches and one form read with the receipt photo), so about $1.40 for the wave. If the account isn't funded yet you'll get a prompt for the $5 activation deposit. Go ahead?

Owner: *"yes"*

```
location_create:
  name: "Burger Barn - Elm St"
  street_address: "118 Elm Street, Austin, TX 78704"
  checkin_radius_m: 150
  idempotency_key: "loc-store-1"
  → location_id: 9001, lat: 30.2452, lng: -97.7618, pin_quality: "place",
    radius_note: "Check-in radius is enforced by the talent policy ...",
    billing: { meter: "geocode", units: 1, price: 0.03 }

location_create: name "Burger Barn - S Lamar Blvd", street_address "2400 S Lamar Blvd, Austin, TX 78704", checkin_radius_m 150, idempotency_key "loc-store-2"  → 9002, pin_quality "place"
location_create: name "Burger Barn - Congress Ave", street_address "615 Congress Ave, Austin, TX 78701", checkin_radius_m 150, idempotency_key "loc-store-3"  → 9003, pin_quality "street"
location_create: name "Burger Barn - Pearl St",     street_address "900 Pearl Street, Denver, CO 80203",   checkin_radius_m 150, idempotency_key "loc-store-4"  → 9004, pin_quality "place"

sqlite_execute: UPDATE stores SET zensched_location_id = 9001 WHERE store_id = 1;
sqlite_execute: UPDATE stores SET zensched_location_id = 9002 WHERE store_id = 2;
sqlite_execute: UPDATE stores SET zensched_location_id = 9003 WHERE store_id = 3;
sqlite_execute: UPDATE stores SET zensched_location_id = 9004 WHERE store_id = 4;
```

Nothing about Priya, the fees, or the scoring is in `name` or `notes`. One event per store for the wave, the form attached to each:

```
event_create:
  location_id: 9001
  title: "Burger Barn Lunch - Elm St"
  start_date: "2026-09-08"
  end_date:   "2026-09-30"          ← the wave (23 days; the cap is 60)
  idempotency_key: "event-ps-1-20260908"
  → event_id: 7001

form_assign:
  form_id: 301
  event_id: 7001
  idempotency_key: "assign-form-1-7001"
  → status: "assigned", form_id: 301, event_id: 7001, policy_id: 0

sqlite_execute: UPDATE program_stores SET zensched_event_id = 7001, event_valid_until = '2026-09-30' WHERE program_store_id = 1;

event_create: location_id 9002, title "Burger Barn Lunch - S Lamar Blvd", 2026-09-08..2026-09-30, key "event-ps-2-20260908"  → 7002;  form_assign 301 → 7002, key "assign-form-1-7002"
event_create: location_id 9003, title "Burger Barn Lunch - Congress Ave", 2026-09-08..2026-09-30, key "event-ps-3-20260908"  → 7003;  form_assign 301 → 7003, key "assign-form-1-7003"
event_create: location_id 9004, title "Burger Barn Lunch - Pearl St",     2026-09-08..2026-09-30, key "event-ps-4-20260908"  → 7004;  form_assign 301 → 7004, key "assign-form-1-7004"
sqlite_execute: UPDATE program_stores SET zensched_event_id = 7002, event_valid_until = '2026-09-30' WHERE program_store_id = 2;
sqlite_execute: UPDATE program_stores SET zensched_event_id = 7003, event_valid_until = '2026-09-30' WHERE program_store_id = 3;
sqlite_execute: UPDATE program_stores SET zensched_event_id = 7004, event_valid_until = '2026-09-30' WHERE program_store_id = 4;

sqlite_execute: INSERT INTO shops (program_store_id) VALUES (1);   → shop 1 (open)
sqlite_execute: INSERT INTO shops (program_store_id) VALUES (2);   → shop 2
sqlite_execute: INSERT INTO shops (program_store_id) VALUES (3);   → shop 3
sqlite_execute: INSERT INTO shops (program_store_id) VALUES (4);   → shop 4

sqlite_execute: UPDATE programs SET status = 'active' WHERE program_id = 1;
```

> Burger Barn Lunch - Sep is live: 4 stores (3 Austin, 1 Denver), 4 shops, weekdays 11:00–14:00 through Sep 30. $45 per shop to Burger Barn; shoppers earn $18 plus the meal up to $12. The Lunch Experience form has 15 fields with a required receipt photo, and I'll score each shop out of 100 from your weights. Congress Ave geocoded to the street rather than the building; with the 150 m radius that's fine, tell me if a shopper has trouble checking in there. No shoppers yet; who should I invite?

## "Invite Dana Ruiz, dana@example.com, Austin, PayPal same email. And Marcus Lee, marcus@example.com, Denver, Venmo @marcuslee."

> Inviting two shoppers costs $0.50. OK?

Owner: *"yes"*

```
worker_invite:
  email: "dana@example.com"
  first_name: "Dana"
  last_name: "Ruiz"
  idempotency_key: "worker-dana@example.com"
  → worker_id: 501, status: "invited", invitation_sent: true
    billing: { meter: "worker_invite", units: 1, price: 0.25 }

worker_invite: email "marcus@example.com", first_name "Marcus", last_name "Lee", idempotency_key "worker-marcus@example.com"  → worker_id 502

sqlite_execute: INSERT INTO shoppers (shopper_name, email, home_city, home_region, zensched_worker_id, pay_handle) VALUES ('Dana Ruiz', 'dana@example.com', 'Austin', 'TX', 501, 'paypal:dana@example.com');   → 1
sqlite_execute: INSERT INTO shoppers (shopper_name, email, home_city, home_region, zensched_worker_id, pay_handle) VALUES ('Marcus Lee', 'marcus@example.com', 'Denver', 'CO', 502, 'venmo:@marcuslee');   → 2
```

> Invited Dana and Marcus. Each gets an email with the app link and an activation code. They'll see the store, their time slot, and the form; the scenario ("order a combo at the counter, ask about dessert, eat in") you send them yourself, or tell me to include it when I confirm assignments.

## "Give Dana the three Austin shops next week, lunch, and Marcus the Denver one on Tuesday."

```
sqlite_query: SELECT * FROM shops_open;
  → shop 1 | Burger Barn - Elm St        | Austin | -05:00 | evt 7001 | 11:00–14:00 | 1111100 | 22 days left
    shop 2 | Burger Barn - S Lamar Blvd  | Austin | -05:00 | evt 7002 | ...
    shop 3 | Burger Barn - Congress Ave  | Austin | -05:00 | evt 7003 | ...
    shop 4 | Burger Barn - Pearl St      | Denver | -06:00 | evt 7004 | ...
sqlite_query: SELECT shopper_id, zensched_worker_id FROM shoppers WHERE is_active = 1;   → (1, 501), (2, 502)
```

Next week is Sep 14–18, all weekdays. Dana's three shops go on Mon, Wed, Fri so she is never at two Burger Barns in one day; 45-minute slots inside 11:00–14:00.

```
sqlite_execute: UPDATE shops SET shopper_id = 1, scheduled_start = '2026-09-14T12:00:00', scheduled_end = '2026-09-14T12:45:00', status = 'assigned' WHERE shop_id = 1;
sqlite_execute: UPDATE shops SET shopper_id = 1, scheduled_start = '2026-09-16T12:00:00', scheduled_end = '2026-09-16T12:45:00', status = 'assigned' WHERE shop_id = 2;
sqlite_execute: UPDATE shops SET shopper_id = 1, scheduled_start = '2026-09-18T12:30:00', scheduled_end = '2026-09-18T13:15:00', status = 'assigned' WHERE shop_id = 3;
sqlite_execute: UPDATE shops SET shopper_id = 2, scheduled_start = '2026-09-15T12:30:00', scheduled_end = '2026-09-15T13:15:00', status = 'assigned' WHERE shop_id = 4;

sqlite_query: SELECT * FROM shops_upcoming WHERE zensched_shift_id IS NULL;
  → shop 1 | Elm St       | worker 501 | start_iso 2026-09-14T12:00:00-05:00 | end_iso 2026-09-14T12:45:00-05:00 | shift-shop-1 | needs_location 0 | needs_event 0 | evt 7001
    shop 4 | Pearl St     | worker 502 | start_iso 2026-09-15T12:30:00-06:00 | end_iso 2026-09-15T13:15:00-06:00 | shift-shop-4 | needs_location 0 | needs_event 0 | evt 7004
  (shops 2 and 3 are 8 and 10 days out, past the view's 7-day horizon; the agent builds their ISO strings the same way: scheduled_start || tz_offset)

shift_create:
  event_id: 7001
  worker_id: 501
  start: "2026-09-14T12:00:00-05:00"
  end:   "2026-09-14T12:45:00-05:00"
  idempotency_key: "shift-shop-1"
  → shift_id: 88001, status: "scheduled", forms_installed: [301]

shift_create: event_id 7002, worker_id 501, start "2026-09-16T12:00:00-05:00", end "2026-09-16T12:45:00-05:00", idempotency_key "shift-shop-2"  → 88002
shift_create: event_id 7003, worker_id 501, start "2026-09-18T12:30:00-05:00", end "2026-09-18T13:15:00-05:00", idempotency_key "shift-shop-3"  → 88003
shift_create:
  event_id: 7004
  worker_id: 502
  start: "2026-09-15T12:30:00-06:00"        ← Denver offset, not the agency's
  end:   "2026-09-15T13:15:00-06:00"
  idempotency_key: "shift-shop-4"
  → shift_id: 88004, status: "scheduled", forms_installed: [301]

sqlite_execute: UPDATE shops SET zensched_shift_id = 88001 WHERE shop_id = 1;
sqlite_execute: UPDATE shops SET zensched_shift_id = 88002 WHERE shop_id = 2;
sqlite_execute: UPDATE shops SET zensched_shift_id = 88003 WHERE shop_id = 3;
sqlite_execute: UPDATE shops SET zensched_shift_id = 88004 WHERE shop_id = 4;
```

> Assigned:
> - Dana: Mon 9/14 12:00–12:45 Elm St; Wed 9/16 12:00–12:45 S Lamar Blvd; Fri 9/18 12:30–13:15 Congress Ave
> - Marcus: Tue 9/15 12:30–13:15 Pearl St (Mountain time)
>
> Both notified in the app with the Lunch Experience form attached and a reminder 30 minutes before. Send them the scenario; remind them the receipt photo is required and to arrive inside the slot.

Nothing but the shift IDs is written locally. Asking again for the same shops returns the same four shifts because of `shift-shop-{shop_id}`.

## The shoppers do the work

Monday 12:03, Dana at 118 Elm Street: **Check in** (GPS: 22 m from the pin, verified, $0.10). Orders a combo, is asked about dessert, eats. Opens **Burger Barn Lunch Experience**: arrival 12:04, greeted Yes, wait Under 2 min, order accurate Yes, friendliness 5, upsell Yes, cleanliness None, receipt photo, total 11.85, narrative. Submit. **Check out** 12:41.

Tuesday, Marcus in Denver runs late. Slot was 12:30–13:15 Mountain; he checks in at 13:55 (GPS verified, 40 m), orders at 14:07 per the receipt, fills in the form (greeted Yes, wait 2-5 min, order accurate **No**: "asked for no pickles, got pickles", friendliness 3, upsell Yes, cleanliness None, total 13.20), checks out 14:27.

Wednesday, Dana at S Lamar: in 12:01, greeted No, wait 2-5 min, accurate Yes, friendliness 4, upsell No, cleanliness Dirty tables, total 12.40, out 12:44.

Friday, Dana does not go to Congress Ave (car trouble; she texts the owner). Shift 88003 goes `missed`.

## "Pull this week's results." (Saturday, Sep 19)

```
shift_list:
  date_from: "2026-09-14"
  date_to:   "2026-09-18"
  status:    "checked_out"
  → shifts: [
      { shift_id: 88001, event_id: 7001, worker_id: 501, date: "2026-09-14", start: "12:00", end: "12:45", status: "checked_out" },
      { shift_id: 88004, event_id: 7004, worker_id: 502, date: "2026-09-15", start: "12:30", end: "13:15", status: "checked_out" },
      { shift_id: 88002, event_id: 7002, worker_id: 501, date: "2026-09-16", start: "12:00", end: "12:45", status: "checked_out" }
    ], count: 3

sqlite_query: SELECT shop_id, program_store_id, shopper_id FROM shops WHERE zensched_shift_id IN (88001, 88002, 88004) AND status = 'assigned';
  → (1, 1, 1), (2, 2, 1), (4, 4, 2)

shift_status: shift_id 88001
  → status "checked_out", actual_in "2026-09-14T12:03:40-05:00", actual_out "2026-09-14T12:41:20-05:00",
    punches: [ { type: "in", gps_verified: true, distance_from_site_m: 22 }, { type: "out", gps_verified: true, distance_from_site_m: 18 } ]
shift_status: shift_id 88004
  → actual_in "2026-09-15T13:55:10-06:00", actual_out "2026-09-15T14:27:05-06:00", both punches gps_verified, 40 m / 35 m
shift_status: shift_id 88002
  → actual_in "2026-09-16T12:01:05-05:00", actual_out "2026-09-16T12:44:15-05:00", both punches gps_verified, 15 m / 12 m
(free)

sqlite_query: SELECT * FROM shops_overdue;
  → shop 3 | Congress Ave | Dana Ruiz | 2026-09-18T12:30:00 .. 13:15:00 | shift 88003 | 27 hours overdue
```

Agent (metered read, rule 9):

> Three shops have results. Reading them costs $0.15 each because each has a receipt photo, $0.45 total; I read each once and keep what matters locally. OK? Also, Dana's Congress Ave shop on Friday shows as missed with no check-in — was that the car trouble?

Owner: *"go. yes, no-show, reopen it."*

```
form_export:
  form_id: 301
  since: "2026-09-08"
  until: "2026-09-19"
  format: "json"
  → row_count: 3, rows: [
      { submission_id: 3301, worker_id: 501, event_id: 7001, submitted_at: "2026-09-14 12:39:50",
        data: { arrival_time: "12:04", greeted_30s: "yes", line_wait: "under_2_min", order_accuracy: "yes",
                staff_friendliness: "5", upsell_offered: "yes", clean_issues: ["none"], receipt_total: 11.85,
                narrative: "Greeted at the door by the cashier. Ordered the #2 combo; she asked if I wanted the pie. Food out in 4 minutes, correct, hot. Dining room clean, condiments stocked." },
        media: [ { field: "receipt", cdn_url: "https://cdn.zensched.com/m/.../r3301.jpg" } ] },
      { submission_id: 3302, worker_id: 502, event_id: 7004, submitted_at: "2026-09-15 14:25:30",
        data: { arrival_time: "13:55", greeted_30s: "yes", line_wait: "2_5_min", order_accuracy: "no",
                order_issue: "Asked for no pickles, got pickles.", staff_friendliness: "3", upsell_offered: "yes",
                clean_issues: ["none"], receipt_total: 13.20,
                narrative: "Short line, about 3 minutes. Cashier friendly but rushed. Order came with pickles after I asked for none; I did not send it back. Restaurant was clean." },
        media: [ { field: "receipt", cdn_url: "https://cdn.zensched.com/m/.../r3302.jpg" } ] },
      { submission_id: 3303, worker_id: 501, event_id: 7002, submitted_at: "2026-09-16 12:42:10",
        data: { arrival_time: "12:01", greeted_30s: "no", line_wait: "2_5_min", order_accuracy: "yes",
                staff_friendliness: "4", upsell_offered: "no", clean_issues: ["dirty_tables"], receipt_total: 12.40,
                narrative: "No greeting; two staff talking at the drive-thru window. Ordered after about 3 minutes, not offered a combo or dessert. Food correct. Three tables had trays and wrappers left on them the whole visit." },
        media: [ { field: "receipt", cdn_url: "https://cdn.zensched.com/m/.../r3303.jpg" } ] }
    ]
    billing: { meter: "form_media", units: 3, price: 0.45 }
```

Match: 3301 → event 7001 / worker 501 / Sep 14 → shop 1. 3302 → 7004 / 502 / Sep 15 → shop 4. 3303 → 7002 / 501 / Sep 16 → shop 2. Score from `question_weights` (total weight 10): shop 1 = 2+3+2+1+2 = 10 → **100**; shop 4 = 2+0+0+1+2 = 5 → **50**; shop 2 = 0+3+2+0+0 = 5 → **50**. Reimbursement = receipt capped at $12.

```
sqlite_execute:
  UPDATE shops SET status = 'completed', submission_dc_id = 3301,
         checkin_at = '2026-09-14T12:03:40-05:00', checkout_at = '2026-09-14T12:41:20-05:00',
         receipt_total = 11.85, reimbursement = 11.85, score = 100, qa_status = 'pending'
  WHERE shop_id = 1;
  (trigger fills duration_minutes = 38)

sqlite_execute:
  UPDATE shops SET status = 'completed', submission_dc_id = 3302,
         checkin_at = '2026-09-15T13:55:10-06:00', checkout_at = '2026-09-15T14:27:05-06:00',
         receipt_total = 13.20, reimbursement = 12.00, score = 50, qa_status = 'pending'
  WHERE shop_id = 4;
  (duration_minutes = 32)

sqlite_execute:
  UPDATE shops SET status = 'completed', submission_dc_id = 3303,
         checkin_at = '2026-09-16T12:01:05-05:00', checkout_at = '2026-09-16T12:44:15-05:00',
         receipt_total = 12.40, reimbursement = 12.00, score = 50, qa_status = 'pending'
  WHERE shop_id = 2;
  (duration_minutes = 43)

sqlite_execute: UPDATE shops SET status = 'no_show', notes = 'car trouble, texted owner Fri morning' WHERE shop_id = 3;
sqlite_execute: INSERT INTO shops (program_store_id) VALUES (3);   → shop 5 (open, Congress Ave again)

sqlite_query: SELECT * FROM shops_flagged WHERE qa_status = 'pending';
  → shop 4 | Pearl St | Marcus Lee | slot 2026-09-15T12:30:00..13:15:00 | checkin 13:55:10-06:00 | 32 min
           | checkin_late 1 | checkin_early 0 | no_checkin 0 | short_visit 0 | minutes_after_slot_end 40
```

> Pulled 3 shops, all GPS-verified at the store.
>
> **Flag — Pearl St (Denver), Marcus, Tue.** He checked in at 13:55, 40 minutes after his 12:30–13:15 slot ended, and the receipt is stamped 14:07, past the client's 2 pm window. The shop itself: greeted, 2–5 min wait, order wrong (pickles), friendliness 3, upsell offered, clean. Score 50.
>
> On time:
> - Elm St, Dana, Mon: 38 min. Greeted, under 2 min, accurate, friendliness 5, upsell offered, clean. Score 100. Receipt $11.85.
> - S Lamar Blvd, Dana, Wed: 43 min. **Not greeted, no upsell, dirty tables**; order accurate, friendliness 4. Score 50. Receipt $12.40 (reimbursing $12.00).
>
> Congress Ave is marked a no-show and reopened as a new shop. Approve, reject, or ask me for the receipt links.

## "Approve Dana's two. Reject Marcus — Burger Barn won't take anything ordered after 2. Reopen it. Then give Congress Ave to Dana Tuesday 12:00 and Pearl St to Marcus Wednesday 12:00."

```
sqlite_execute: UPDATE shops SET qa_status = 'approved' WHERE shop_id = 1;
sqlite_execute: UPDATE shops SET qa_status = 'approved', qa_notes = 'No greeting, no upsell, dirty tables — genuine findings, approved.' WHERE shop_id = 2;
sqlite_execute: UPDATE shops SET qa_status = 'rejected', status = 'rejected', qa_notes = 'Checked in 40 min after slot; receipt 14:07 is after the 11-2 window. Client will not accept.' WHERE shop_id = 4;
sqlite_execute: INSERT INTO shops (program_store_id) VALUES (4);   → shop 6 (open, Pearl St again)

sqlite_execute: UPDATE shops SET shopper_id = 1, scheduled_start = '2026-09-22T12:00:00', scheduled_end = '2026-09-22T12:45:00', status = 'assigned' WHERE shop_id = 5;
sqlite_execute: UPDATE shops SET shopper_id = 2, scheduled_start = '2026-09-23T12:00:00', scheduled_end = '2026-09-23T12:45:00', status = 'assigned' WHERE shop_id = 6;

sqlite_query: SELECT * FROM shops_upcoming WHERE zensched_shift_id IS NULL;
  → shop 5 | Congress Ave | worker 501 | 2026-09-22T12:00:00-05:00 .. 12:45:00-05:00 | shift-shop-5 | evt 7003
    shop 6 | Pearl St     | worker 502 | 2026-09-23T12:00:00-06:00 .. 12:45:00-06:00 | shift-shop-6 | evt 7004

shift_create: event_id 7003, worker_id 501, start "2026-09-22T12:00:00-05:00", end "2026-09-22T12:45:00-05:00", idempotency_key "shift-shop-5"  → 88005
shift_create: event_id 7004, worker_id 502, start "2026-09-23T12:00:00-06:00", end "2026-09-23T12:45:00-06:00", idempotency_key "shift-shop-6"  → 88006
sqlite_execute: UPDATE shops SET zensched_shift_id = 88005 WHERE shop_id = 5;
sqlite_execute: UPDATE shops SET zensched_shift_id = 88006 WHERE shop_id = 6;
```

> Done. Elm St and S Lamar approved. Pearl St rejected (not billed, not paid; his reimbursement isn't owed either — say so if you want to cover the $12 anyway). Congress Ave: Dana, Tue 9/22 12:00–12:45. Pearl St: Marcus, Wed 9/23 12:00–12:45 Mountain. Both notified. Marcus's reliability now shows one rejected shop; worth a word to him about arriving inside the slot.

The rejected and no-show rows keep their `zensched_shift_id`s for history; the replacements are new rows 5 and 6.

## "How's the Burger Barn program doing?"

```
sqlite_query: SELECT * FROM program_progress WHERE program_id = 1;
  → Burger Barn Lunch - Sep | active | 4 stores | required 4 | open 0 | assigned 2 | completed 2 | approved 2 | no_show 1 | qa_pending 0 | 50.0% | 11 days left
```

> Half done: 2 of 4 shops approved (Elm St 100, S Lamar 50). Congress Ave and Pearl St are assigned for Tue and Wed. One no-show and one rejection so far, both redone. 11 days left in the wave.

## The following week: "Pull results."

Same pattern, abbreviated. Both redo shops go fine.

```
shift_list: date_from "2026-09-21", date_to "2026-09-27", status "checked_out"  → 88005 (501, 7003, 9/22), 88006 (502, 7004, 9/23)
shift_status: 88005 → in "2026-09-22T11:58:30-05:00", out "2026-09-22T12:39:00-05:00", verified 9 m / 11 m
shift_status: 88006 → in "2026-09-23T12:02:15-06:00", out "2026-09-23T12:36:40-06:00", verified 28 m / 30 m

form_export: form_id 301, since "2026-09-20", until "2026-09-27", format "json"
  → 3305 (501, 7003, 9/22: greeted yes, under_2_min, accurate yes, friendliness 4, upsell yes, clean none, receipt 10.95) → score 100
    3306 (502, 7004, 9/23: greeted yes, 2_5_min, accurate yes, friendliness 5, upsell no, clean none, receipt 11.60) → score 90
  billing: { meter: "form_media", units: 2, price: 0.30 }

sqlite_execute: UPDATE shops SET status = 'completed', submission_dc_id = 3305, checkin_at = '2026-09-22T11:58:30-05:00', checkout_at = '2026-09-22T12:39:00-05:00', receipt_total = 10.95, reimbursement = 10.95, score = 100, qa_status = 'pending' WHERE shop_id = 5;   (duration 40)
sqlite_execute: UPDATE shops SET status = 'completed', submission_dc_id = 3306, checkin_at = '2026-09-23T12:02:15-06:00', checkout_at = '2026-09-23T12:36:40-06:00', receipt_total = 11.60, reimbursement = 11.60, score = 90, qa_status = 'pending' WHERE shop_id = 6;   (duration 34)
sqlite_query: SELECT * FROM shops_flagged WHERE qa_status = 'pending';   → (none)
```

> Two shops, both on time and GPS-verified, nothing flagged. Congress Ave (Dana): score 100, receipt $10.95. Pearl St (Marcus): score 90, only miss was no upsell, receipt $11.60. Approve?

Owner: *"approve both"*

```
sqlite_execute: UPDATE shops SET qa_status = 'approved' WHERE shop_id IN (5, 6);
```

## "Send Burger Barn their results and invoice."

Submissions were already read, so the CSV export bills nothing new.

```
form_export:
  form_id: 301
  since: "2026-09-08"
  until: "2026-09-30"
  format: "csv"
  → download_url: "https://exports.zensched.com/.../burger-barn-lunch-experience.csv" (expires), row_count: 5
    billing: { meter: "form_media", units: 0, price: 0.00 }   ← all five already billed
```

> Results CSV: (link). It has 5 rows: the 4 approved shops plus Marcus's rejected Sep 15 shop; delete that row, and delete the `worker_name` column before it goes to Burger Barn. Summary for the cover email: 4 of 4 stores shopped; average score 85; Elm St 100, Congress Ave 100, Pearl St 90, S Lamar Blvd 50 (no greeting, no upsell, dirty tables).

```
sqlite_query: SELECT * FROM shops_to_invoice;
  → Burger Barn | ap@burgerbarn.example | net 30 | 4 shops | 1 program | fees 180.00 | reimbursements 46.40 | total 226.40 | 2026-09-14 .. 2026-09-23

sqlite_execute:
  INSERT INTO invoices (client_id, program_id, invoice_date, due_date, shop_count, fees_amount, reimbursement_amount, total_amount, line_items)
  SELECT p.client_id, CASE WHEN COUNT(DISTINCT p.program_id) = 1 THEN MIN(p.program_id) END, date('now'),
         date('now', '+' || COALESCE(MAX(c.payment_terms_days), (SELECT value FROM settings WHERE key = 'invoice_due_days')) || ' days'),
         COUNT(*), SUM(p.client_fee), SUM(COALESCE(sh.reimbursement, 0)), SUM(p.client_fee) + SUM(COALESCE(sh.reimbursement, 0)),
         json_group_array(json_object('shop_id', sh.shop_id, 'date', date(sh.scheduled_start), 'store', st.store_label, 'store_code', st.store_code,
                                      'program', p.program_name, 'fee', p.client_fee, 'reimbursement', sh.reimbursement, 'score', sh.score))
  FROM shops sh JOIN program_stores ps ON ps.program_store_id = sh.program_store_id JOIN programs p ON p.program_id = ps.program_id
       JOIN stores st ON st.store_id = ps.store_id JOIN clients c ON c.client_id = p.client_id
  WHERE sh.status = 'completed' AND sh.qa_status = 'approved' AND sh.client_invoiced = 0 AND p.client_id = 1 GROUP BY p.client_id;

sqlite_execute:
  UPDATE shops SET client_invoiced = 1 WHERE shop_id IN (SELECT sh.shop_id FROM shops sh JOIN program_stores ps ON ps.program_store_id = sh.program_store_id
                                                          JOIN programs p ON p.program_id = ps.program_id
                                                          WHERE sh.status = 'completed' AND sh.qa_status = 'approved' AND sh.client_invoiced = 0 AND p.client_id = 1);
sqlite_query:   SELECT invoice_number, invoice_date, due_date, shop_count, total_amount FROM invoices WHERE invoice_id = last_insert_rowid();
  → INV-2026-0001 | 2026-09-26 | 2026-10-26 | 4 | 226.40     (number assigned by trigger)
```

> Here's the invoice, ready to paste into an email to ap@burgerbarn.example:
>
> ```
> Northline Field Insights
> Invoice INV-2026-0001
> To: Burger Barn — Burger Barn Lunch - Sep
> Date: Sep 26, 2026    Due: Oct 26, 2026 (net 30)
>
>   Sep 14   #1042  Burger Barn - Elm St           shop fee  $45.00   meal  $11.85
>   Sep 16   #1057  Burger Barn - S Lamar Blvd     shop fee  $45.00   meal  $12.00
>   Sep 22   #1088  Burger Barn - Congress Ave     shop fee  $45.00   meal  $10.95
>   Sep 23   #2210  Burger Barn - Pearl St         shop fee  $45.00   meal  $11.60
>
> Shop fees:       $180.00
> Reimbursements:   $46.40
> Total due:       $226.40
> Every shop was GPS-verified at the store with a timestamped receipt. Results CSV sent separately.
> ```
>
> No shopper names on it. Say "sent" and I'll note the date.

## "Sent. Now do shopper pay."

```
sqlite_execute: UPDATE invoices SET sent_date = date('now') WHERE invoice_number = 'INV-2026-0001';

sqlite_query: SELECT * FROM shopper_pay_due;
  → Dana Ruiz  | paypal:dana@example.com | 3 shops | fees 54.00 | reimbursements 34.80 | total 88.80 | 2026-09-14 .. 2026-09-22 | [1,2,5]
    Marcus Lee | venmo:@marcuslee        | 1 shop  | fees 18.00 | reimbursements 11.60 | total 29.60 | 2026-09-23 .. 2026-09-23 | [6]

sqlite_execute:
  INSERT INTO shopper_payouts (shopper_id, period_start, period_end, shop_count, fees_amount, reimbursement_amount, total_amount, shop_ids)
  SELECT shopper_id, '2026-09-14', '2026-09-27', shop_count, fees_amount, reimbursement_amount, total_due, shop_ids FROM shopper_pay_due WHERE shopper_id = 1;
sqlite_execute: UPDATE shops SET shopper_paid = 1 WHERE shopper_id = 1 AND status = 'completed' AND qa_status = 'approved' AND shopper_paid = 0;
sqlite_execute:
  INSERT INTO shopper_payouts (...) SELECT shopper_id, '2026-09-14', '2026-09-27', shop_count, fees_amount, reimbursement_amount, total_due, shop_ids FROM shopper_pay_due WHERE shopper_id = 2;
sqlite_execute: UPDATE shops SET shopper_paid = 1 WHERE shopper_id = 2 AND status = 'completed' AND qa_status = 'approved' AND shopper_paid = 0;
```

> Pay sheet for Sep 14–27:
>
> ```
> Dana Ruiz     paypal:dana@example.com
>   Sep 14  Elm St          $18.00 + $11.85
>   Sep 16  S Lamar Blvd    $18.00 + $12.00
>   Sep 22  Congress Ave    $18.00 + $10.95
>                                       $88.80
> Marcus Lee    venmo:@marcuslee
>   Sep 23  Pearl St        $18.00 + $11.60
>                                       $29.60
> Total                                $118.40
> ```
>
> Marcus's rejected Sep 15 shop is not on here. Tell me "paid" when the transfers are done and I'll mark both payouts.

Owner: *"paid"*

```
sqlite_execute: UPDATE shopper_payouts SET paid = 1, paid_date = date('now') WHERE paid = 0;
```

## "Who's my most reliable shopper?"

```
sqlite_query: SELECT * FROM shopper_reliability WHERE is_active = 1;
  → Dana Ruiz  | given 4 | completed 3 | approved 3 | no_shows 1 | rejected 0 | on_time 100.0% | avg_score 83.3 | last 2026-09-22
    Marcus Lee | given 2 | completed 2 | approved 1 | no_shows 0 | rejected 1 | on_time 50.0%  | avg_score 70.0 | last 2026-09-23
```

No ZenSched call, no cost.

> Dana: 3 of 4 done, every check-in on time, one no-show (Friday, car trouble, she called). Marcus: 2 shops, one rejected for arriving 40 minutes late, the redo was on time. Both fine for next month; I'd send Marcus the slot reminder in writing.

## Summary of who stored what

| Thing | Where | Why |
|---|---|---|
| Burger Barn's contact (Priya), billing email, payment terms | SQLite **only** | Client confidentiality; ZenSched does not model clients |
| The program: wave dates, window, weekdays, quota, $45 / $18 / $12 cap, min minutes, the client's brief | SQLite **only** | Fees and the brief never reach shoppers |
| Question weights and per-shop scores | SQLite **only** | ZenSched is not a scoring engine |
| Store cache with normalized addresses and per-store `tz_offset` | SQLite | Geocode each store once; Denver shifts in `-06:00` |
| Dana's and Marcus's pay handles, home cities, reliability | SQLite | Pay and assignment decisions; ZenSched holds only the worker account |
| QA decisions and notes, `client_invoiced` / `shopper_paid` flags | SQLite **only** | Money and judgment |
| Four store pins: `Burger Barn - Elm St` etc. with street addresses | ZenSched (IDs in `stores`) | Needed for geofenced check-in |
| Four events `Burger Barn Lunch - <street>`, Sep 8–30 | ZenSched (IDs + `event_valid_until` in `program_stores`) | Shifts hang off events; one per store per wave |
| The Lunch Experience form (15 fields, required receipt photo) | ZenSched (ID in `programs.zensched_form_id`) | Installed on the phone per shift; one form per program |
| Dana, Marcus, their invites, their apps | ZenSched (IDs in `shoppers`) | Workforce and notifications |
| Six shifts (four original, two redos) | ZenSched only (IDs in `shops`) | Live schedule; never copied |
| GPS punches, distance from the store, actual in/out times | ZenSched only | Verified record; stamps copied to `shops.checkin_at` / `checkout_at` |
| Five submissions with receipt photos | ZenSched (originals); `submission_dc_id`, receipt total, score in `shops` | Read once (metered), then answered from SQLite; CSV replay for the client was free |
| One client invoice, two shopper payouts | SQLite | Money |

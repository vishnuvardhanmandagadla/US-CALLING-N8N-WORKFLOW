# System Architecture & Execution Lifecycle

## Architecture (Merged Webhook)

```
[Workflow 5: Lead Merge]           Populates leads into Master Queue (3 phones per lead)
        |
        v
[Google Sheet: US_Leads_Master_Queue]     Central data store (46 columns)
        |
        v
[Workflow 1: Call Trigger]         Reads leads, picks best phone, initiates Retell calls
        |                          Falls back to phone_2/phone_3 on failure
        v
[Retell AI Cloud]                  Makes the actual phone call
        |
        v
[Merged Retell Webhook]            Single webhook receives ALL Retell events
        |
        +--[Switch Node]
        |       |
        |       +--> "call_ended" branch    Captures raw call outcome + basic sheet update
        |       |
        |       +--> "call_analyzed" branch  Captures AI analysis + phone status + alerts
        |
        +--> [Workflow 4: Retell Functions]  Separate webhook: handles real-time function calls during call
```

## How the Two Main Flows Interact

```
                    +-----------------------+
                    |   CALL TRIGGER (WF1)  |
                    |  Schedule: every 30m  |
                    +-----------+-----------+
                                |
                    Reads Sheet -> Filters -> Calls Retell API
                                |
                    +-----------v-----------+
                    |    RETELL AI CLOUD    |
                    |  Makes phone call     |
                    +-----------+-----------+
                                |
              Fires webhooks when call ends + after AI analysis
                                |
                    +-----------v-----------+
                    |  MERGED WEBHOOK       |
                    |  POST /retell-webhook |
                    +-----------+-----------+
                                |
                    +-----------v-----------+
                    |  Switch: event type   |
                    +-----+-----+----------+
                          |           |
                    call_ended   call_analyzed
                          |           |
                          v           v
                  Basic update   Full AI analysis
                  to Sheet       + alerts + Sheet update
```

---

## Complete Execution Lifecycle

### Step 1: Lead Ingestion (WF5 - Manual)
- Reads source sheets (TradeIndia, EventsEye, 10Times, ExpoData) + master queue
- Deduplicates across sources, extracts up to 3 phones per lead
- Eliminates leads without valid US phone numbers
- Parses date ranges (uses event END date for priority scoring)
- Past events hard-capped at score 50 (never outrank future events)
- Appends new leads with `phone_1/2/3`, statuses = `pending`, priority scores

### Step 2: Call Triggering (WF1 - Every 30 min)
- Checks business hours (9AM-6PM EST, Mon-Fri) — `TESTING_MODE=true` currently bypasses
- Reads all leads from Google Sheet
- Filters for callable leads: valid status + under 9 attempts + callable phone + eligible timing
- `getNextCallablePhone()` picks `current_phone_index` first, then tries others
- Sorts by `priority_score` descending, takes top 2
- For each lead:
  1. Updates sheet: `call_status='calling'`, `call_attempts+1`, `current_phone_index`
  2. Builds Retell API payload with `phone_index` in metadata
  3. Calls Retell API `POST /v2/create-phone-call`
  4. On success: sets `retell_call_id`, `call_status='initiated'`, `phone_X_status='calling'`
  5. On failure: sets `phone_X_status='failed/retry'`, advances phone or marks exhausted
  6. Waits 15 minutes, then processes next lead

### Step 3: Live Call (Retell AI)
- Uses dynamic variables (customer name, event, company)
- Metadata carries `lead_id` + `phone_index` for webhook tracking
- May trigger function calls handled by WF4

### Step 4: Real-Time Functions (WF4)
- `check_calendar` - returns available demo slots
- `book_demo` - confirms demo booking
- `get_pricing` - returns pricing tiers

### Step 5: Call Ended Event (Merged Webhook - call_ended branch)
- Receives webhook at `/retell-webhook`
- Switch routes to `call_ended` branch
- Extracts basic call data (duration, transcript, recording, disconnection reason)
- Updates sheet with raw call data

### Step 6: Call Analyzed Event (Merged Webhook - call_analyzed branch)
- Same webhook, Switch routes to `call_analyzed` branch
- Extracts ALL data: AI analysis + custom fields + tool calls
- Determines call outcome via reasonMap (20 disconnection reasons mapped)
- Determines phone-specific status + fallback logic
- Updates sheet with 22+ analysis fields + dynamic `phone_X_status`
- If HOT lead (`interest=HOT/CALLBACK` or meeting scheduled):
  1. Sends styled HTML email via Gmail
  2. Updates `alert_sent=TRUE` + `alert_sent_date`

---

## Credentials Required

| Credential             | Type                          | Used In                       |
|------------------------|-------------------------------|-------------------------------|
| Google Sheets OAuth2   | OAuth2                        | WF1, Merged Webhook, WF5     |
| Retell API Key         | HTTP Header Auth (`x-api-key`)| WF1                           |
| Gmail OAuth2           | OAuth2                        | Merged Webhook (email alerts) |

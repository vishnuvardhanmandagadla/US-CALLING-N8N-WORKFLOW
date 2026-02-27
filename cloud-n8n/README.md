# US Calling - Cloud n8n Workflows (LIVE PRODUCTION)

These are the **currently running production workflows** on cloud n8n that power EventTitans' automated US cold calling system using Retell AI.

---

## System Overview

```
Google Sheets (us-masterqueue)    Retell AI Platform
        |                                |
        v                                v
+--------------------+    +----------------------------+
| US Leads -         |    | US Leads - Retell Webhook  |
| Call Trigger       |--->| (Merged)                   |
| (Scheduled)        |    | (Webhook: POST)            |
+--------------------+    +----------------------------+
        |                         |            |
        v                         v            v
   Retell API            call_ended      call_analyzed
   Create Call           (raw data)      (AI analysis)
        |                    |                 |
        v                    v                 v
   Google Sheets        Google Sheets     Google Sheets
   (status update)      (call data)       (full analysis)
                                               |
                                               v
                                          HOT Lead?
                                           /     \
                                         Yes      No
                                          |        |
                                          v        v
                                      Gmail      [end]
                                     Alert
                                      |
                                      v
                                  Mark Alert
                                  in Sheet
```

---

## Workflow 1: US Leads - Call Trigger

**File:** `US-Leads-Call-Trigger.json`
**Type:** Scheduled (Cron) + Loop
**Schedule:** Mon-Fri, cron `30 20 * * 1-5` (maps to US Eastern business hours)
**Batch Size:** Up to 20 leads per execution

### Flow Chart

```
Schedule Trigger (Mon-Fri)
    |
    v
Get US Holidays (Google Calendar)
    |
    v
Is Holiday? ----Yes----> [STOP]
    |
    No
    v
Check Business Hours (10 AM - 5 PM ET)
    |
    v (throws error if outside hours)
Read Google Sheet (us-masterqueue)
    |
    v
Filter & Sort Leads (Code)
    |
    v
Has Leads? ----No----> No Leads [END]
    |
    Yes
    v
Loop Over Items (1 at a time)
    |
    v
Prepare Calling Update (Code)
    |
    v
Update Status to Calling (Google Sheets)
    |
    v
Prepare Retell Data (Code)
    |
    v
Call Retell API (HTTP POST)
    |
    v
Parse & Store Call ID (Code)
    |
    v
Call Started? ----No----> Prepare Failed Update --> Mark Call Failed --> Wait 15min --> Loop
    |
    Yes
    v
Prepare Call ID Update (Code)
    |
    v
Update Call ID in Sheet (Google Sheets)
    |
    v
Wait 15 minutes
    |
    v
Loop Over Items (next lead)
```

### Node-by-Node Breakdown

| # | Node Name | Type | What It Does |
|---|-----------|------|-------------|
| 1 | **Schedule Trigger** | scheduleTrigger | Fires on cron `30 20 * * 1-5`. Runs during US business hours Mon-Fri |
| 2 | **Get many events** | googleCalendar | Reads US public holidays from Google Calendar for today |
| 3 | **If** (Holiday Check) | if | Checks if `summary` is empty (no holiday). If holiday exists, workflow stops |
| 4 | **Check Business Hours** | code | Validates current time is 10 AM - 5 PM US Eastern. Throws error to stop if outside hours or weekend |
| 5 | **Read Google Sheet** | googleSheets | Reads entire `us-masterqueue` sheet (all leads) |
| 6 | **Filter & Sort Leads** | code | Filters leads by: `call_status` in [pending, scheduled, retry, callback_scheduled], `call_attempts < 9`, has callable phone, `next_call_date <= now`. Sorts by `priority_score` descending. Limits to 20 leads |
| 7 | **Has Leads?** | if | Checks if filtered lead count > 0 |
| 8 | **Loop Over Items** | splitInBatches | Processes leads one at a time with 15-min wait between each |
| 9 | **Prepare Calling Update** | code | Sets `call_status = 'calling'`, increments `call_attempts`, records `last_call_date`, sets `current_phone_index` |
| 10 | **Update Status to Calling** | googleSheets | Writes the "calling" status to Google Sheets (matched by `lead_id`) |
| 11 | **Prepare Retell Data** | code | Normalizes phone to E.164, builds `metadata` (lead_id, phone_index, attempt, max_phone_index), builds `dynamic_variables` (customer_name, event_name, event_date, venue, city). From number: `+19298284999` |
| 12 | **Call Retell API** | httpRequest | POST to `https://api.retellai.com/v2/create-phone-call` with agent_id, from/to numbers, metadata, and dynamic variables. 30s timeout, continues on error |
| 13 | **Parse & Store Call ID** | code | Checks if `call_id` exists in response. Returns `{success, lead_id, call_id, phone_index}` or `{success: false, error}` |
| 14 | **Call Started?** | if | Routes to success or failure path based on `success` boolean |
| 15 | **Prepare Call ID Update** | code | Builds update: `retell_call_id`, `call_status = 'initiated'`, and sets `phone_X_status = 'calling'` for the active phone |
| 16 | **Update Call ID in Sheet** | googleSheets | Writes call_id and initiated status to sheet |
| 17 | **Prepare Failed Update** | code | Handles API failures: retries same phone (30min delay), moves to next phone after 3 per-phone failures, marks `exhausted` when all phones tried |
| 18 | **Mark Call Failed** | googleSheets | Writes failure status, retry schedule, or exhausted status to sheet |
| 19 | **Wait** | wait | Waits 15 minutes before processing next lead in the loop |
| 20 | **No Leads** | noOp | End node when no eligible leads found |

### Fields Modified by Each Node

| Node | Fields Written to Google Sheets |
|------|-------------------------------|
| **Prepare Calling Update** | `call_status` = "calling", `call_attempts` +1, `last_call_date`, `current_phone_index` |
| **Prepare Call ID Update** | `lead_id`, `retell_call_id`, `call_status` = "initiated", `phone_X_status` = "calling" |
| **Prepare Failed Update** | `lead_id`, `call_status` = "retry"/"scheduled"/"exhausted", `last_call_date`, `call_attempts`, `current_phone_index`, `next_call_date`, `phone_X_status` = "failed"/"retry", `follow_up_status` |

### Phone Rotation Logic

```
Lead has up to 3 phones: phone_1, phone_2, phone_3
Each phone gets 3 attempts (MAX_RETRIES_PER_PHONE = 3)
Total max attempts per lead = 9 (3 phones x 3 attempts)

Flow:
  phone_1: attempt 1 --> attempt 2 --> attempt 3
      |
      v (after 3 failures)
  phone_2: attempt 4 --> attempt 5 --> attempt 6
      |
      v (after 3 failures)
  phone_3: attempt 7 --> attempt 8 --> attempt 9
      |
      v (after 3 failures)
  EXHAUSTED (no more calls)
```

### Phone Status Values

| Status | Meaning | Can Retry? |
|--------|---------|-----------|
| `pending` | Not yet called | Yes |
| `calling` | Currently being called | No (in progress) |
| `no_answer` | Phone rang, no answer | Yes |
| `busy` | Line busy | Yes |
| `voicemail` | Reached voicemail | Yes |
| `retry` | Scheduled for retry | Yes |
| `scheduled` | Scheduled callback | Yes |
| `connected` | Spoke with human | No (terminal) |
| `declined` | User declined/spam | No (terminal) |
| `invalid` | Invalid phone number | No (terminal) |
| `exhausted` | All attempts used | No (terminal) |
| `failed` | API/telephony error | No (moves to next phone) |

---

## Workflow 2: US Leads - Retell Webhook (Merged)

**File:** `US-Leads-Retell-Webhook-Merged.json`
**Type:** Webhook (POST)
**Endpoint:** `/webhook/retell-webhook`
**Triggered by:** Retell AI platform after each call

### Flow Chart

```
Retell Webhook (POST /webhook/retell-webhook)
    |
    v
Switch: call_ended or call_analyzed?
    |                        |
    v                        v
[CALL ENDED PATH]      [CALL ANALYZED PATH]
    |                        |
    v                        v
Filter call_ended       Filter call_analyzed
    |                        |
    v                        v
Extract Call Data       Extract All Data (Code - MAIN LOGIC)
    |                        |
    v                        v
Has Lead ID?            Has Lead ID?
   / \                     / \
  No  Yes                No   Yes
  |    |                 |     |
  v    v                 v     v
Log  Prepare           Log   Prepare Analysis Update
     Sheet Update             |
     |                        v
     v                   Update All Extracted Data (Sheet)
Update Sheet                  |
with Call Data                v
                         Is HOT or CALLBACK?
                            /        \
                          Yes         No
                           |           |
                           v           v
                     Set Email       No Alert
                     Fields              [END]
                        |
                        v
                   Send Gmail Alert
                        |
                        v
                   alert date marking (Set node)
                        |
                        v
                   Mark Alert Sent (Sheet)
```

### Node-by-Node Breakdown

| # | Node Name | Type | What It Does |
|---|-----------|------|-------------|
| 1 | **Retell Webhook** | webhook | Receives POST at `/webhook/retell-webhook` from Retell AI platform |
| 2 | **call ended or anlyzed** | switch | Routes to `call_ended` path or `call_analyzed` path based on `body.event` |
| 3 | **Filter call_ended Event** | if | Confirms event is `call_ended` |
| 4 | **Extract Call Data** | code | Extracts: `lead_id` (from metadata), `retell_call_id`, `call_status`, `call_duration` (ms to seconds), `disconnection_reason`, `recording_url`, `call_transcript` |
| 5 | **Has Lead ID?** | if | Checks if `lead_id` exists and is not empty |
| 6 | **Prepare Sheet Update** | code | Builds update object with raw call data fields |
| 7 | **Update Sheet with Call Data** | googleSheets | Writes raw call data to sheet (matched by `lead_id`) |
| 8 | **Filter call_analyzed Event** | if | Confirms event is `call_analyzed` |
| 9 | **Extract All Data** | code | **THE MAIN BRAIN** - Massive logic node that: extracts AI analysis, maps disconnection reasons to outcomes, determines retry/next-phone/exhausted logic, parses callback times, detects voicemail, flags HOT leads |
| 10 | **Has Lead ID?1** | if | Checks if `lead_id` exists |
| 11 | **Prepare Analysis Update** | code | Builds update with all analyzed fields + dynamic `phone_X_status` |
| 12 | **Update All Extracted Data** | googleSheets | Writes full analysis to sheet (matched by `lead_id`) |
| 13 | **Is HOT or CALLBACK?** | if | Checks `is_hot = TRUE` AND `reached_voicemail = "No"` |
| 14 | **Set Email Fields** | set | Prepares all fields needed for the email alert template |
| 15 | **Send a message** | gmail | Sends styled HTML hot lead alert email to `Vedhavyas@eventtitans.com` |
| 16 | **alert date marking** | set | Sets `lead_id`, `alert_sent = TRUE`, `alert_sent_date = now` |
| 17 | **Mark Alert Sent** | googleSheets | Updates sheet with alert_sent flag |
| 18 | **Ignore Event / Ignore Event1** | noOp | Silently drops non-matching events |
| 19 | **Log Missing ID / Log Missing** | noOp | Silently drops events without lead_id |
| 20 | **No Alert** | noOp | End node for non-HOT leads |

### Fields Modified by Each Node

#### Call Ended Path

| Node | Fields Written to Google Sheets |
|------|-------------------------------|
| **Prepare Sheet Update** | `lead_id`, `retell_call_id`, `call_duration`, `disconnection_reason`, `recording_url`, `call_transcript`, `last_call_date` |

#### Call Analyzed Path

| Node | Fields Written to Google Sheets |
|------|-------------------------------|
| **Prepare Analysis Update** | `lead_id`, `call_status`, `agent_id`, `current_phone_index`, `call_outcome`, `call_attempts`, `call_successful`, `next_call_date`, `call_cost`, `call_summary`, `user_sentiment`, `interest_level`, `objection_reason`, `callback_time`, `decision_maker`, `pain_points`, `next_steps`, `follow_up_status`, `meeting_scheduled`, `email_captured`, `phone_captured`, `demo_scheduled`, `is_hot`, `phone_X_status` |
| **Mark Alert Sent** | `lead_id`, `alert_sent` = "TRUE", `alert_sent_date` |

### Disconnection Reason Mapping (Extract All Data node)

| Disconnection Reason | Phone Status | Call Outcome | Retry? | Next Phone? | Stop All? |
|---------------------|-------------|-------------|--------|------------|----------|
| `user_hangup` | connected | CONNECTED | No | No | No |
| `agent_hangup` | connected | CONNECTED | No | No | No |
| `call_transfer` | connected | CONNECTED | No | No | No |
| `inactivity` | connected | CONNECTED | No | No | No |
| `max_duration_reached` | connected | CONNECTED | No | No | No |
| `voicemail_reached` | voicemail | VOICEMAIL | Yes (4h delay) | No | No |
| `machine_detected` | voicemail | VOICEMAIL | Yes (4h delay) | No | No |
| `dial_no_answer` | no_answer | NO_ANSWER | Yes (2h delay) | No | No |
| `registered_call_timeout` | no_answer | NO_ANSWER | Yes (2h delay) | No | No |
| `dial_busy` | busy | BUSY | Yes (1h delay) | No | No |
| `dial_failed` | failed | FAILED | No | Yes | No |
| `invalid_destination` | invalid | FAILED | No | Yes | No |
| `telephony_provider_permission_denied` | failed | FAILED | No | Yes | No |
| `telephony_provider_unavailable` | failed | FAILED | No | Yes | No |
| `sip_routing_error` | failed | FAILED | No | Yes | No |
| `user_declined` | declined | DECLINED | No | No | Yes |
| `marked_as_spam` | declined | DECLINED | No | No | Yes |
| `scam_detected` | declined | DECLINED | No | No | Yes |
| `no_valid_payment` | failed | FAILED | No | No | Yes |
| `concurrency_limit_reached` | retry | (retry) | Yes (30min) | No | No |

### Call Status State Machine

```
pending
    |
    v (Call Trigger picks it up)
calling
    |
    v (Retell API responds)
initiated
    |
    v (call_ended webhook)
    |
    +---> completed      (connected with human)
    +---> retry          (no_answer/busy/voicemail, retries left)
    +---> scheduled      (moving to next phone)
    +---> callback_scheduled  (prospect requested callback)
    +---> exhausted      (all phones tried, max attempts reached)
    +---> not_interested (user declined/spam)
```

### HOT Lead Detection

A lead is flagged as HOT when ANY of these are true:
- `interest_level` = "HOT" or "CALLBACK" (from Retell AI custom analysis)
- `meeting_scheduled` = true (from Retell AI custom analysis)
- Tool calls include meeting/schedule/calendar functions

HOT leads that are NOT voicemail trigger:
1. Styled HTML email alert to `Vedhavyas@eventtitans.com`
2. `alert_sent` = TRUE and `alert_sent_date` recorded in sheet

### Callback Time Parsing

The Extract All Data node parses callback requests in these formats:
- ISO date: `2026-03-01T15:00:00Z`
- Relative: `tomorrow at 3pm`, `tomorrow at 3:30 PM`
- Duration: `in 2 hours`, `in 30 minutes`
- Bare time: `3 PM`, `3:30 PM` (assumes today if future, tomorrow if past)

All times are converted to US Eastern timezone for scheduling.

---

## Google Sheets Schema (us-masterqueue)

The complete list of columns in the master queue sheet:

### Lead Identity Fields (never modified by workflows)

| Column | Description |
|--------|------------|
| `lead_id` | Unique lead identifier (match key for all updates) |
| `name` | Contact name |
| `company` | Company name |
| `email` | Contact email |
| `event_name` | Event the lead is associated with |
| `event_date` | Event date |
| `venue` | Event venue |
| `city` | Event city |
| `added_date` | Date lead was added to queue |
| `priority_score` | Lead priority (higher = called first) |

### Phone Fields

| Column | Set By | Description |
|--------|--------|------------|
| `phone_1` | (static) | Primary phone number |
| `phone_2` | (static) | Secondary phone number |
| `phone_3` | (static) | Tertiary phone number |
| `phone_1_status` | Call Trigger + Webhook | Status of phone_1 |
| `phone_2_status` | Call Trigger + Webhook | Status of phone_2 |
| `phone_3_status` | Call Trigger + Webhook | Status of phone_3 |
| `current_phone_index` | Call Trigger + Webhook | Which phone (1/2/3) is currently being tried |

### Call Tracking Fields

| Column | Set By | Description |
|--------|--------|------------|
| `call_status` | Call Trigger + Webhook | Current status (pending/calling/initiated/completed/retry/exhausted/not_interested) |
| `call_attempts` | Call Trigger + Webhook | Total number of call attempts made |
| `last_call_date` | Call Trigger + Webhook | Timestamp of most recent call |
| `next_call_date` | Call Trigger + Webhook | When to try calling again |
| `retell_call_id` | Call Trigger + Webhook | Retell platform call ID |
| `call_duration` | Webhook (call_ended) | Call duration in seconds |
| `call_transcript` | Webhook (call_ended) | Full call transcript |
| `recording_url` | Webhook (call_ended) | URL to call recording |
| `disconnection_reason` | Webhook (both) | Why the call ended |
| `call_cost` | Webhook (call_analyzed) | Cost of the call |
| `call_successful` | Webhook (call_analyzed) | Yes/No |
| `call_outcome` | Webhook (call_analyzed) | CONNECTED/VOICEMAIL/NO_ANSWER/BUSY/FAILED/DECLINED |
| `agent_id` | Webhook (call_analyzed) | Retell agent ID used |

### AI Analysis Fields (set by Webhook call_analyzed path only)

| Column | Source | Description |
|--------|--------|------------|
| `call_summary` | Retell `call_analysis.call_summary` | AI-generated call summary |
| `user_sentiment` | Retell `call_analysis.user_sentiment` | Detected sentiment (Positive/Negative/Neutral) |
| `interest_level` | Retell `custom_analysis_data.interest_level` | HOT / WARM / COLD / CALLBACK |
| `meeting_scheduled` | Retell `custom_analysis_data` + tool_calls | Yes if meeting was scheduled |
| `demo_scheduled` | Retell tool_calls (demo-related) | Yes if demo was scheduled |
| `email_captured` | Retell `custom_analysis_data.email_captured` | Email address if captured during call |
| `phone_captured` | Retell `custom_analysis_data.phone_captured` | Alternate phone if captured |
| `objection_reason` | Retell `custom_analysis_data.objection_reason` | Why prospect objected |
| `callback_time` | Retell `custom_analysis_data.callback_time` | When to call back |
| `decision_maker` | Retell `custom_analysis_data.decision_maker` | Yes if spoke with decision maker |
| `pain_points` | Retell `custom_analysis_data.pain_points` | Identified pain points |
| `next_steps` | Retell `custom_analysis_data.next_steps` | Recommended next actions |
| `follow_up_status` | Computed | pending_review / scheduled_callback / exhausted / closed |

### Alert Fields

| Column | Set By | Description |
|--------|--------|------------|
| `is_hot` | Webhook (call_analyzed) | TRUE/FALSE - hot lead flag |
| `alert_sent` | Webhook (alert path) | TRUE if email alert was sent |
| `alert_sent_date` | Webhook (alert path) | Timestamp of alert email |

---

## Credentials Required

| Credential | Used By | Purpose |
|-----------|---------|---------|
| **Google Sheets OAuth2** | Both workflows | Read/write us-masterqueue sheet |
| **Google Calendar OAuth2** | Call Trigger | Check US holidays |
| **Retell API (HTTP Header Auth)** | Call Trigger | Create phone calls via Retell API |
| **Gmail OAuth2** | Webhook | Send hot lead email alerts |

## Configuration

| Setting | Value |
|---------|-------|
| Retell Agent ID | `agent_9b6f6050f1902acf8c2676e242` |
| From Phone Number | `+19298284999` |
| Google Sheet ID | `1bKvWR2SN03ZjwSrs6cKrN_nkzaDo-PoMiC6vCfVQdz0` |
| Alert Email | `Vedhavyas@eventtitans.com` |
| Business Hours | 10 AM - 5 PM US Eastern, Mon-Fri |
| Max Attempts Per Phone | 3 |
| Max Total Attempts | 9 (3 phones x 3) |
| Leads Per Execution | 20 |
| Wait Between Leads | 15 minutes |
| Webhook Path | `/webhook/retell-webhook` |

---

## How the Two Workflows Work Together

```
TIME        CALL TRIGGER                    RETELL WEBHOOK (MERGED)
-------     --------------------------      ---------------------------------
T+0:00      Schedule fires
            Check holiday + hours
            Read sheet, filter leads
            Pick lead #1
            Set status = "calling"
            Call Retell API
            Set status = "initiated"

T+0:01                                      Retell starts ringing phone...

T+0:30                                      call_ended webhook fires
                                            --> Extract raw data (duration,
                                                transcript, recording)
                                            --> Write to sheet

T+1:00                                      call_analyzed webhook fires
                                            --> Extract AI analysis
                                            --> Map disconnection reason
                                            --> Determine retry/next/done
                                            --> Update sheet with full data
                                            --> If HOT: send email alert

T+15:00     Wait finishes
            Pick lead #2
            Repeat...
```

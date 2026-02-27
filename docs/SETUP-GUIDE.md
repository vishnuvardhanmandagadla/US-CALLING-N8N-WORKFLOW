# Setup Guide

## Prerequisites

- [n8n](https://n8n.io/) instance (self-hosted or cloud)
- Google account with Sheets and Gmail API access
- [Retell AI](https://www.retellai.com/) account with API key
- Retell AI agent configured for EventTitans cold calling

---

## Step 1: Import Workflows

Import each JSON file into n8n via **Workflows > Import from File**:

| Order | File                                | Workflow Name                                 |
|-------|-------------------------------------|-----------------------------------------------|
| 1     | `workflow5-us-lead-merge.json`      | US Leads - Lead Merge for Retell Calling      |
| 2     | `workflow1-call-trigger.json`       | US Leads - Call Trigger                       |
| 3     | `workflow-merged-webhook.json`      | US Leads - (Merged) Retell Webhook            |
| 4     | `workflow4-retell-functions.json`   | US Leads - Retell Functions                   |

> **Note:** Import WF5 first so the master queue exists before other workflows reference it.

---

## Step 2: Configure Credentials

### Google Sheets OAuth2

| Setting        | Value                                               |
|----------------|-----------------------------------------------------|
| **Used in**    | WF1, WF5, Merged Webhook                           |
| **Scopes**     | `https://www.googleapis.com/auth/spreadsheets`      |

### Gmail OAuth2

| Setting        | Value                                               |
|----------------|-----------------------------------------------------|
| **Used in**    | Merged Webhook (email alerts)                       |
| **Scopes**     | `https://www.googleapis.com/auth/gmail.send`        |

### Retell API Key

| Setting        | Value                                               |
|----------------|-----------------------------------------------------|
| **Used in**    | WF1 (Call Trigger)                                  |
| **Type**       | HTTP Header Auth                                    |
| **Header**     | `x-api-key`                                         |
| **Value**      | Your Retell API key                                 |

---

## Step 3: Set Up Google Sheet

Create a Google Sheet named `US_Leads_Master_Queue` with **46 columns** in this exact order:

```
lead_id | name | company | phone_1 | phone_2 | phone_3 | current_phone_index |
phone_1_status | phone_2_status | phone_3_status | email | event_name |
event_date | venue | city | added_date | priority_score | call_status |
call_attempts | last_call_date | next_call_date | retell_call_id | call_duration |
call_transcript | recording_url | user_sentiment | disconnection_reason |
call_outcome | interest_level | call_summary | meeting_scheduled |
email_captured | objection_reason | callback_time | decision_maker |
pain_points | next_steps | follow_up_status | alert_sent | alert_sent_date |
demo_scheduled | call_successful | call_cost | agent_id | phone_captured | is_hot
```

See [Data Model](DATA-MODEL.md) for full column descriptions and status values.

---

## Step 4: Configure Webhook URLs

After importing, n8n generates webhook URLs. Configure these in Retell:

| Webhook                          | n8n Path              | Retell Setting                       |
|----------------------------------|-----------------------|--------------------------------------|
| Call events (ended + analyzed)   | `/retell-webhook`     | Agent > Post-Call Webhook URL        |
| Real-time functions              | `/retell-functions`   | Agent > Custom Functions Webhook URL |

Both webhooks accept **POST** requests.

---

## Step 5: Update Sheet IDs

Replace the Google Sheet document IDs in each workflow:

| Workflow | Node                      | Current ID                                        | Replace With                      |
|----------|---------------------------|---------------------------------------------------|-----------------------------------|
| WF5      | Read Master Queue FIRST   | `1bKvWR2SN03ZjwSrs6cKrN_nkzaDo-PoMiC6vCfVQdz0`  | Your master queue sheet ID        |
| WF5      | Read USA-TradeIndia       | `1CNyXDfhYii-pfC5CQH9-F_7FxQGr8qiYSEp3jpYeIDg`  | Your TradeIndia source sheet ID   |
| WF5      | Read USA-EventsEye        | `YOUR_SOURCE_SHEET_2_ID`                          | Your EventsEye source sheet ID    |
| WF5      | Read USA-10Times          | `YOUR_SOURCE_SHEET_3_ID`                          | Your 10Times source sheet ID      |
| WF5      | Read USA-ExpoData         | `YOUR_SOURCE_SHEET_4_ID`                          | Your ExpoData source sheet ID     |
| WF5      | Append to Master Queue    | `1bKvWR2SN03ZjwSrs6cKrN_nkzaDo-PoMiC6vCfVQdz0`  | Your master queue sheet ID        |
| WF1      | Read Google Sheet         | *(same master queue ID)*                          | Your master queue sheet ID        |
| Merged   | All Google Sheets nodes   | *(same master queue ID)*                          | Your master queue sheet ID        |

---

## Step 6: Update Retell Configuration

| Setting        | Node                          | Current Value                              | Update To                  |
|----------------|-------------------------------|--------------------------------------------|----------------------------|
| Agent ID       | WF1 > Prepare Retell Data     | `agent_9b6f6050f1902acf8c2676e242`         | Your Retell agent ID       |
| From Number    | WF1 > Prepare Retell Data     | `+19298284999`                             | Your Retell phone number   |
| Alert Email    | Merged > Send Email           | `vishnuvardhanmandagadla@gmail.com`        | Your notification email    |

---

## Step 7: Disable Testing Mode

In **WF1 > Check Business Hours** code node, change:

```javascript
// Change this:
const TESTING_MODE = true;

// To this:
const TESTING_MODE = false;
```

This enables business hours enforcement (9 AM - 6 PM EST, Monday - Friday).

---

## Step 8: Activate Workflows

| Workflow               | Activation                                           |
|------------------------|------------------------------------------------------|
| WF1 (Call Trigger)     | Toggle **Active** — runs every 30 minutes            |
| Merged Webhook         | Toggle **Active** — listens for Retell webhooks      |
| WF4 (Retell Functions) | Toggle **Active** — listens for function calls       |
| WF5 (Lead Merge)       | Keep **inactive** — run manually as needed          |

---

## Test Flow Checklist

1. **Test Lead Merge (WF5)**
   - [ ] Add a test lead to your source sheet with a valid US phone
   - [ ] Run WF5 manually
   - [ ] Verify new lead appears in master queue with `USL-XXXXX` ID, `call_status=pending`

2. **Test Call Trigger (WF1)**
   - [ ] Ensure `TESTING_MODE = true` (bypasses business hours)
   - [ ] Run WF1 manually or wait for schedule
   - [ ] Verify lead status changes to `calling` then `initiated`
   - [ ] Verify `retell_call_id` is populated

3. **Test Webhooks (Merged)**
   - [ ] Make a test call via Retell
   - [ ] Check `call_ended` branch updates: `call_duration`, `call_transcript`, `recording_url`
   - [ ] Check `call_analyzed` branch updates: `call_outcome`, `interest_level`, `call_summary`

4. **Test Functions (WF4)**
   - [ ] During a test call, trigger "check calendar" — verify slots returned
   - [ ] Trigger "book demo" — verify confirmation response
   - [ ] Trigger "get pricing" — verify pricing tiers returned

5. **Test Hot Lead Alert**
   - [ ] Ensure a test call results in `interest_level=HOT` or `meeting_scheduled=true`
   - [ ] Verify email alert arrives at configured address
   - [ ] Verify `alert_sent=TRUE` and `alert_sent_date` updated in sheet

---

## Deprecated Workflows

These files are superseded by the Merged Webhook and can be removed:

| File                              | Replaced By                               |
|-----------------------------------|-------------------------------------------|
| `workflow2-call-ended.json`       | Merged Webhook (`call_ended` branch)      |
| `workflow3-call-analyzed.json`    | Merged Webhook (`call_analyzed` branch)   |

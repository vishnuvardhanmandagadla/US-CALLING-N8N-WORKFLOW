# Merged Retell Webhook (Replaces WF2 + WF3)

| Property        | Value                                                      |
|-----------------|-------------------------------------------------------------|
| **File**        | `workflow-merged-webhook.json`                              |
| **Name**        | US Leads - (Merged) Retell Webhook                          |
| **Nodes**       | 21                                                          |
| **Trigger**     | Single Webhook (POST to `/retell-webhook`)                  |
| **Replaces**    | ~~workflow2-call-ended.json~~ + ~~workflow3-call-analyzed.json~~ |

Both `call_ended` and `call_analyzed` events from Retell are sent to the **same** webhook URL. A Switch node routes them to the correct processing branch.

---

## Flow Diagram

```
Retell Webhook (POST /retell-webhook)
    |
    v
Switch: "call ended or anlyzed"
    |                              |
    | event = "call_ended"         | event = "call_analyzed"
    v                              v
+---------------------+    +----------------------+
|  CALL ENDED BRANCH  |    |  CALL ANALYZED BRANCH |
+----------+----------+    +-----------+----------+
           |                           |
           v                           v
Filter call_ended Event        Filter call_analyzed Event
    |YES        |NO                |YES         |NO
    v           v                  v            v
Extract     Ignore Event      Extract All    Ignore Event1
Call Data                     Data
    |                              |
    v                              v
Has Lead ID?                  Has Lead ID?1
    |YES      |NO                 |YES       |NO
    v         v                   v          v
Prepare   Log Missing ID     Prepare     Log Missing
Sheet                        Analysis
Update                       Update
    |                              |
    v                              v
Update Sheet              Update All Extracted Data
with Call Data                     |
    |                              v
    | (END)                   Edit Fields1 (pass data for alert check)
                                   |
                                   v
                            Is HOT or CALLBACK?
                                |TRUE          |FALSE
                                v              v
                          Send Email        No Alert
                          (Gmail)           (stop)
                                |
                                v
                          Edit Fields (prepare alert_sent data)
                                |
                                v
                          Mark Alert Sent (update sheet)
```

---

## Branch 1: Call Ended (Basic Data Capture)

Fires immediately when a call ends. Captures raw call data and writes it to the sheet.

### Node 1: Retell Webhook

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.webhook` (v2)            |
| **Path**       | `/retell-webhook`                         |
| **Method**     | POST                                      |

**Incoming Payload from Retell:**
```
{
  "event": "call_ended" | "call_analyzed",
  "call": {
    "call_id": "...",
    "call_status": "ended",
    "duration_ms": 45000,
    "disconnection_reason": "user_hangup",
    "transcript": "...",
    "recording_url": "...",
    "metadata": { "lead_id": "USL-00001", "phone_index": 1, "attempt": 1 },
    "call_analysis": { ... },        // only in call_analyzed
    "tool_calls": [ ... ],           // only in call_analyzed
    "call_cost": { "combined_cost": 0.0234 }
  }
}
```

---

### Node 2: Switch - "call ended or anlyzed"

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.switch` (v3.3)           |

| Output     | Condition                           | Destination                 |
|------------|-------------------------------------|-----------------------------|
| Output 0   | `body.event === "call_ended"`       | Filter call_ended Event     |
| Output 1   | `body.event === "call_analyzed"`    | Filter call_analyzed Event  |

---

### Node 3: Filter call_ended Event

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.if` (v2.2)               |
| **Condition**  | `body.event === "call_ended"`             |
| **TRUE**       | Extract Call Data                         |
| **FALSE**      | Ignore Event                              |

---

### Node 4: Extract Call Data

| Property       | Value                                              |
|----------------|----------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                        |
| **Purpose**    | Parse raw call data from Retell webhook            |

**Output Fields:**

| Field                  | Source                                  |
|------------------------|-----------------------------------------|
| `lead_id`              | `call.metadata.lead_id`                 |
| `has_lead_id`          | Computed boolean                        |
| `retell_call_id`       | `call.call_id`                          |
| `call_status`          | `call.call_status`                      |
| `last_call_date`       | `new Date().toISOString()`              |
| `call_duration`        | `Math.round(call.duration_ms / 1000)`   |
| `current_phone_index`  | `call.metadata.phone_index`             |
| `disconnection_reason` | `call.disconnection_reason`             |
| `recording_url`        | `call.recording_url`                    |
| `call_transcript`      | `call.transcript`                       |
| `phone_index`          | `call.metadata.phone_index`             |
| `call_attempts`        | `call.metadata.attempt`                 |

---

### Node 5: Has Lead ID?

| Property        | Value                                                                               |
|-----------------|-------------------------------------------------------------------------------------|
| **Type**        | `n8n-nodes-base.if` (v2.2)                                                         |
| **Conditions**  | (OR) `lead_id` not empty, OR `has_lead_id` true, OR `call_status === "ended"`       |
| **TRUE**        | Prepare Sheet Update                                                                |
| **FALSE**       | Log Missing ID                                                                      |

---

### Node 6: Prepare Sheet Update

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)               |

**Output Fields:**

| Field                  | Value                       |
|------------------------|-----------------------------|
| `lead_id`              | From extracted data         |
| `retell_call_id`       | Call ID                     |
| `call_status`          | Raw status                  |
| `call_duration`        | Duration in seconds         |
| `disconnection_reason` | Reason string               |
| `recording_url`        | Recording link              |
| `call_transcript`      | Full transcript             |
| `last_call_date`       | Current timestamp           |

---

### Node 7: Update Sheet with Call Data

| Property          | Value                                     |
|-------------------|-------------------------------------------|
| **Type**          | `n8n-nodes-base.googleSheets` (v4.5)     |
| **Operation**     | Update row (match on `lead_id`)           |
| **Retry on Fail** | Yes (5 second wait)                      |

**Columns Updated:** `retell_call_id`, `call_status`, `call_duration`, `disconnection_reason`, `recording_url`, `call_transcript`, `last_call_date`

---

## Branch 2: Call Analyzed (AI Analysis + Alerts)

Fires after Retell's AI analyzes the call. Captures all analysis fields, determines outcome, handles fallback, sends alerts.

### Node 8: Filter call_analyzed Event

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.if` (v2.2)               |
| **Condition**  | `body.event === "call_analyzed"`          |
| **TRUE**       | Extract All Data                          |
| **FALSE**      | Ignore Event1                             |

---

### Node 9: Extract All Data (CORE ANALYSIS NODE)

| Property       | Value                                                      |
|----------------|------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                                |
| **Purpose**    | Extract ALL data + determine outcome + fallback logic      |

**Data Sources:**

| Source            | Path                                  | Fields                                                                                                                         |
|-------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Metadata          | `call.metadata`                       | `lead_id`, `phone_index`, `attempt`                                                                                            |
| AI Analysis       | `call.call_analysis`                  | `call_summary`, `user_sentiment`, `in_voicemail`                                                                               |
| Custom Analysis   | `call.call_analysis.custom_analysis_data` | `interest_level`, `meeting_scheduled`, `email_captured`, `phone_captured`, `objection_reason`, `callback_time`, `decision_maker`, `pain_points`, `next_steps` |
| Tool Calls        | `call.tool_calls[]`                   | Checks for meeting/demo/email function calls                                                                                   |
| Call Data         | `call.*`                              | `duration_ms`, `disconnection_reason`, `recording_url`, `transcript`, `agent_id`, `call_cost`                                   |

**Disconnection reason mapping, call outcome logic, callback time parsing, and hot lead detection are documented in [Multi-Phone System](MULTI-PHONE-SYSTEM.md).**

**Key Logic in This Node:**
1. Maps `disconnection_reason` via `reasonMap` to determine phone status, retry behavior, and fallback
2. Determines `call_status` for WF1 filter pickup (`scheduled`, `completed`, `not_interested`, `exhausted`, `callback_scheduled`)
3. Parses `callback_time` from AI analysis — if user requested a specific callback, overrides `next_call_date` and sets `call_status = 'callback_scheduled'`
4. Detects hot leads for email alerts

**Complete Output (30+ fields):**

| Category         | Fields                                                                                                                                                                     |
|------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Identification   | `lead_id`, `has_lead_id`                                                                                                                                                    |
| Call metadata    | `agent_id`, `phone_index`, `phone_status`, `current_phone_index`, `call_status`, `call_outcome`, `call_attempts`, `disconnection_reason`, `call_successful`, `call_duration`, `call_transcript`, `recording_url`, `next_call_date`, `call_cost` |
| AI Analysis      | `call_summary`, `user_sentiment`, `interest_level`, `objection_reason`, `callback_time`, `decision_maker`, `pain_points`, `next_steps`, `follow_up_status`, `meeting_scheduled`, `email_captured`, `phone_captured`, `demo_scheduled` |
| Alert flag       | `is_hot`                                                                                                                                                                    |

---

### Node 10: Has Lead ID?1

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.if` (v2.2)               |
| **Condition**  | `lead_id` is not empty                    |
| **TRUE**       | Prepare Analysis Update                   |
| **FALSE**      | Log Missing                               |

---

### Node 11: Prepare Analysis Update

| Property       | Value                                                            |
|----------------|------------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                                      |
| **Purpose**    | Build sheet update with dynamic `phone_X_status`                 |

Takes all fields from Extract All Data and adds the correct `phone_X_status` field based on `phone_index`:
- `phone_index === 1` -> sets `phone_1_status`
- `phone_index === 2` -> sets `phone_2_status`
- `phone_index === 3` -> sets `phone_3_status`

---

### Node 12: Update All Extracted Data

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.googleSheets` (v4.5)     |
| **Operation**  | Update row (match on `lead_id`)           |

**Columns Updated (23+ fields):** `agent_id`, `current_phone_index`, `phone_X_status`, `call_status`, `call_outcome`, `call_attempts`, `call_successful`, `next_call_date`, `call_cost`, `call_summary`, `user_sentiment`, `interest_level`, `objection_reason`, `callback_time`, `decision_maker`, `pain_points`, `next_steps`, `follow_up_status`, `meeting_scheduled`, `email_captured`, `phone_captured`, `demo_scheduled`, `is_hot`

---

### Node 13: Edit Fields1 (Prepare Alert Data)

| Property       | Value                                                              |
|----------------|--------------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.set` (v3.4)                                       |
| **Purpose**    | Pass through key fields for alert check and email template         |

**Fields Passed (21 fields):** `is_hot`, `lead_id`, `interest_level`, `user_sentiment`, `call_duration`, `call_cost`, `call_outcome`, `decision_maker`, `call_attempts`, `disconnection_reason`, `recording_url`, `call_summary`, `meeting_scheduled`, `demo_scheduled`, `follow_up_status`, `callback_time`, `pain_points`, `objection_reason`, `next_steps`, `phone_captured`, `email_captured`

---

### Node 14: Is HOT or CALLBACK?

| Property                   | Value                                             |
|----------------------------|---------------------------------------------------|
| **Type**                   | `n8n-nodes-base.if` (v2.2)                       |
| **Condition**              | `is_hot === false` (loose type validation)        |
| **TRUE (is_hot match)**    | Send Email Alert                                  |
| **FALSE**                  | No Alert                                          |

---

### Node 15: Send Email Alert (Gmail)

| Property       | Value                                                                      |
|----------------|----------------------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.gmail` (v2.1)                                             |
| **To**         | `vishnuvardhanmandagadla@gmail.com`                                        |
| **Subject**    | Dynamic: `lead_id - interest_level Lead \| user_sentiment \| Follow Up Now`|
| **Body**       | Rich HTML email with EventTitans branding                                  |

**Email Template Sections:**
1. Header - HOT LEAD banner with interest level, sentiment, duration, cost
2. Lead Details - Decision maker, attempts, disconnect reason
3. Call Summary - AI-generated summary
4. Highlights - Meeting scheduled, demo scheduled, follow-up status
5. Contact Captured - Email, phone, callback time
6. Intelligence - Pain points, objections
7. Next Steps - Recommended actions
8. Recording Link - Button to play recording
9. Footer - "Follow up within 2 hours" urgency

---

### Node 16: Edit Fields (Prepare Alert Sent Data)

| Property       | Value                                          |
|----------------|------------------------------------------------|
| **Type**       | `n8n-nodes-base.set` (v3.4)                   |

| Field              | Value                              |
|--------------------|-------------------------------------|
| `lead_id`          | From Prepare Analysis Update       |
| `alert_sent`       | `TRUE`                             |
| `alert_sent_date`  | Current ISO timestamp              |

---

### Node 17: Mark Alert Sent

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.googleSheets` (v4.5)     |
| **Operation**  | Update row (match on `lead_id`)           |

**Columns Updated:** `alert_sent`, `alert_sent_date`

---

### Dead-End Nodes

| Node              | Purpose                                                |
|-------------------|--------------------------------------------------------|
| Ignore Event      | Non-call_ended events on call_ended branch             |
| Ignore Event1     | Non-call_analyzed events on call_analyzed branch       |
| Log Missing ID    | call_ended events without lead_id                      |
| Log Missing       | call_analyzed events without lead_id                   |
| No Alert          | Non-hot leads (no email needed)                        |

---

## Complete Data Flow Summary

```
Retell Webhook --[POST body]--> Switch

  CALL ENDED BRANCH:
  Filter --> Extract Call Data
    --[{lead_id, call_status, duration, transcript, recording_url, disconnection_reason}]-->
    Has Lead ID? --> Prepare Sheet Update --> Update Sheet with Call Data (END)

  CALL ANALYZED BRANCH:
  Filter --> Extract All Data
    --[{30+ fields: outcome, analysis, phone_status, is_hot}]-->
    Has Lead ID?1 --> Prepare Analysis Update
    --[{22+ fields + dynamic phone_X_status}]-->
    Update All Extracted Data --> Edit Fields1
    --[{21 fields for email + is_hot flag}]-->
    Is HOT?
      YES --> Send Email --> Edit Fields --> Mark Alert Sent (END)
      NO  --> No Alert (END)
```

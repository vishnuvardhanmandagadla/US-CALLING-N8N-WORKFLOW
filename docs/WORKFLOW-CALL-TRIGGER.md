# Workflow 1: Call Trigger

| Property             | Value                                                                       |
|----------------------|-----------------------------------------------------------------------------|
| **File**             | `workflow1-call-trigger.json`                                               |
| **Name**             | US Leads - Call Trigger                                                     |
| **Nodes**            | 18                                                                          |
| **Schedule**         | Every 30 minutes                                                            |
| **Business Hours**   | 9 AM - 6 PM EST, Monday - Friday (currently bypassed: `TESTING_MODE = true`)|
| **Batch Size**       | 2 leads per execution                                                       |
| **Wait Between Leads** | 15 minutes                                                               |

---

## Flow Diagram

```
Schedule Trigger (every 30 min)
    |
    v
Check Business Hours -----> (stops if outside 9AM-6PM EST Mon-Fri)
    |
    v
Read Google Sheet (all rows from us-masterqueue)
    |
    v
Filter & Sort Leads (multi-phone aware, top 2 by priority)
    |
    v
Has Leads? --NO--> No Leads (stop)
    |YES
    v
Loop Over Leads (1 at a time)
    |
    v
Prepare Calling Update (increment attempts, set current_phone_index)
    |
    v
Update Status to Calling (write to Google Sheet)
    |
    v
Prepare Retell Data (build API payload with phone + metadata)
    |
    v
Call Retell API (POST /v2/create-phone-call)
    |
    v
Parse & Store Call ID (check success/failure)
    |
    v
Call Started?
    |YES                               |NO
    v                                  v
Prepare Call ID Update          Prepare Failed Update
(phone_X_status = calling)      (phone_X_status = failed/retry)
    |                                  |
    v                                  v
Update Call ID in Sheet         Mark Call Failed
    |                                  |
    +----------------------------------+
    |
    v
Wait Between Calls (15 min)
    |
    v
(back to Loop Over Leads)
```

---

## Node-by-Node Breakdown

### 1. Schedule Trigger

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.scheduleTrigger` (v1.2)   |
| **Interval**   | Every 30 minutes                          |

**Output:** Trigger event (empty payload, starts the chain)

---

### 2. Check Business Hours

| Property       | Value                                                |
|----------------|------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                           |
| **Purpose**    | Gate: blocks execution outside US business hours     |

- `TESTING_MODE = true` currently bypasses all checks
- In production: converts current time to `America/New_York` timezone
- Checks: weekday (Mon-Fri), not a US federal holiday, hour between 9-17
- US holidays hardcoded for 2024-2025

| Input           | Output                                                                  |
|-----------------|-------------------------------------------------------------------------|
| Trigger event   | Pass-through if within hours, empty `[]` to stop if outside            |

---

### 3. Read Google Sheet

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.googleSheets` (v4.5)     |
| **Sheet**      | `us-masterqueue` (Sheet1)                 |
| **Operation**  | Read all rows                             |

| Input             | Output                                                    |
|-------------------|-----------------------------------------------------------|
| Trigger signal    | Array of ALL lead rows (all 46 columns per row)           |

---

### 4. Filter & Sort Leads

| Property       | Value                                                      |
|----------------|------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                                |
| **Purpose**    | Filter callable leads, find best phone, sort by priority   |
| **Limit**      | Top 2 leads per execution                                  |

**Key Functions:**

| Function                          | Purpose                                                                                                                                     |
|-----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| `isPhoneCallable(phone, status)`  | Returns `false` if status is `connected`, `declined`, `invalid`, `exhausted`. Returns `true` if `pending`, `no_answer`, `busy`, `voicemail`, `retry`, `scheduled`, or empty |
| `getNextCallablePhone(lead)`      | Tries `current_phone_index` first, then iterates phone_1 -> phone_2 -> phone_3                                                             |

**Filter Criteria (ALL must be true):**

| Criterion             | Check                                                            |
|-----------------------|------------------------------------------------------------------|
| Valid call_status     | In `['pending', 'scheduled', 'retry', 'callback_scheduled']`    |
| Under max attempts    | `call_attempts < 9`                                              |
| Has callable phone    | At least one phone passes `isPhoneCallable()`                    |
| Eligible timing       | `next_call_date` is empty or in the past                         |

| Input            | Output                                                                                                     |
|------------------|------------------------------------------------------------------------------------------------------------|
| All lead rows    | Filtered leads + `_next_phone` (number) + `_next_phone_index` (1/2/3), sorted by `priority_score` desc    |

---

### 5. Has Leads?

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.if` (v2.2)               |
| **Condition**  | `$input.all().length > 0`                 |

| TRUE              | FALSE                |
|-------------------|----------------------|
| Loop Over Leads   | No Leads (stop)      |

---

### 6. Loop Over Leads

| Property        | Value                                     |
|-----------------|-------------------------------------------|
| **Type**        | `n8n-nodes-base.splitInBatches` (v3)     |
| **Batch Size**  | 1 (one lead at a time)                    |

---

### 7. Prepare Calling Update

| Property       | Value                                                            |
|----------------|------------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                                      |
| **Purpose**    | Build update payload before marking lead as "calling"            |

**Output Fields:**

| Field                  | Value                              | Source                           |
|------------------------|------------------------------------|----------------------------------|
| `lead_id`              | Lead's unique ID                   | `lead.lead_id`                   |
| `call_status`          | `'calling'`                        | Hardcoded                        |
| `call_attempts`        | Previous + 1                       | `parseInt(lead.call_attempts) + 1` |
| `last_call_date`       | Current ISO timestamp              | `new Date().toISOString()`       |
| `current_phone_index`  | Selected phone index               | `lead._next_phone_index`         |

---

### 8. Update Status to Calling

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.googleSheets` (v4.5)     |
| **Operation**  | Update row (match on `lead_id`)           |
| **Mapping**    | `autoMapInputData`                        |

**Columns Updated:** `call_status`, `call_attempts`, `last_call_date`, `current_phone_index`

---

### 9. Prepare Retell Data

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)               |
| **Purpose**    | Build Retell API request payload          |

**Output Fields:**

| Field                | Source                                                                   |
|----------------------|--------------------------------------------------------------------------|
| `lead_id`            | `lead.lead_id`                                                           |
| `to_number`          | `lead._next_phone` (E.164 normalized)                                    |
| `from_number`        | `+19298284999`                                                           |
| `agent_id`           | `agent_9b6f6050f1902acf8c2676e242`                                       |
| `phone_index`        | `lead._next_phone_index`                                                 |
| `dynamic_variables`  | `{ customer_name, company_name, event_name, event_date, venue, city }`   |
| `metadata`           | `{ lead_id, phone_index, attempt, source: 'n8n_call_trigger' }`         |

---

### 10. Call Retell API

| Property       | Value                                              |
|----------------|----------------------------------------------------|
| **Type**       | `n8n-nodes-base.httpRequest` (v4.2)               |
| **Method**     | POST                                               |
| **URL**        | `https://api.retellai.com/v2/create-phone-call`   |
| **Auth**       | HTTP Header Auth (`x-api-key`)                     |
| **Timeout**    | 30 seconds                                         |
| **On Error**   | `continueRegularOutput`                            |

**Request Body:**
```json
{
  "from_number": "+19298284999",
  "to_number": "+1XXXXXXXXXX",
  "override_agent_id": "agent_XXXX",
  "metadata": { "lead_id": "USL-00001", "phone_index": 1, "attempt": 1, "source": "n8n_call_trigger" },
  "retell_llm_dynamic_variables": { "customer_name": "John", "company_name": "Acme" }
}
```

---

### 11. Parse & Store Call ID

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)               |
| **Purpose**    | Determine if Retell API call succeeded    |

| Output Field    | On Success          | On Failure         |
|-----------------|---------------------|--------------------|
| `success`       | `true`              | `false`            |
| `lead_id`       | From metadata       | From metadata      |
| `call_id`       | From Retell response| -                  |
| `call_status`   | `registered`        | -                  |
| `error`         | -                   | Error message      |
| `phone_index`   | Passed through      | Passed through     |

---

### 12. Call Started?

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.if` (v2.2)               |
| **Condition**  | `$json.success === true`                  |

| TRUE                      | FALSE                     |
|---------------------------|---------------------------|
| Prepare Call ID Update    | Prepare Failed Update     |

---

### 13. Prepare Call ID Update

| Property       | Value                                                  |
|----------------|--------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                            |
| **Purpose**    | Build sheet update for successful API call             |

| Field              | Value                                                                              |
|--------------------|------------------------------------------------------------------------------------|
| `lead_id`          | Lead ID                                                                            |
| `retell_call_id`   | Call ID from Retell                                                                |
| `call_status`      | `initiated`                                                                        |
| `phone_X_status`   | `calling` (dynamically sets correct phone based on `phone_index`)                  |

---

### 14. Update Call ID in Sheet

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.googleSheets` (v4.5)     |
| **Operation**  | Update row (match on `lead_id`)           |

**Columns Updated:** `retell_call_id`, `call_status`, `phone_X_status`

---

### 15. Prepare Failed Update

| Property       | Value                                                      |
|----------------|------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                                |
| **Purpose**    | Handle Retell API failure with fallback logic              |

> Note: This only fires when the Retell API itself failed (no `call_id` returned). Real call outcomes (no_answer, busy, voicemail) are handled by the Merged Webhook's call_analyzed branch.

**Fallback Decision Table:**

| Condition                               | Action                | Fields Set                                                                                               |
|-----------------------------------------|-----------------------|----------------------------------------------------------------------------------------------------------|
| `attempts < 3` (can retry same phone)   | Retry in 30 min       | `call_status='retry'`, `phone_X_status='retry'`, `next_call_date=+30min`                                |
| `attempts >= 3 AND phoneIndex < 3`      | Move to next phone    | `call_status='retry'`, `phone_X_status='failed'`, `current_phone_index=next`, `next_call_date=+1hr`     |
| `attempts >= 3 AND phoneIndex >= 3`     | All exhausted         | `call_status='exhausted'`, `phone_X_status='failed'`, `follow_up_status='exhausted'`                    |

---

### 16. Mark Call Failed

| Property       | Value                                          |
|----------------|------------------------------------------------|
| **Type**       | `n8n-nodes-base.googleSheets` (v4.5)          |
| **Operation**  | Update row (match on `lead_id`)                |

**Columns Updated:** All fields from Prepare Failed Update (varies by scenario)

---

### 17. Wait Between Calls

| Property       | Value                                          |
|----------------|------------------------------------------------|
| **Type**       | `n8n-nodes-base.wait` (v1.1)                  |
| **Duration**   | 15 minutes                                     |
| **Then**       | Loops back to Loop Over Leads for next lead    |

---

### 18. No Leads

| Property       | Value                                          |
|----------------|------------------------------------------------|
| **Type**       | `n8n-nodes-base.noOp` (v1)                    |
| **Purpose**    | Dead end when no callable leads found          |

---

## Complete Data Flow Summary

```
Schedule Trigger
  --[trigger]-->
Check Business Hours
  --[pass-through]-->
Read Google Sheet
  --[all 46-col rows]-->
Filter & Sort Leads
  --[top 2 leads + _next_phone + _next_phone_index]-->
Has Leads? --YES-->
Loop Over Leads
  --[single lead]-->
Prepare Calling Update
  --[{lead_id, call_status:'calling', call_attempts+1, last_call_date, current_phone_index}]-->
Update Status to Calling
  --[sheet updated]-->
Prepare Retell Data
  --[{to_number, from_number, agent_id, metadata, dynamic_variables}]-->
Call Retell API
  --[API response]-->
Parse & Store Call ID
  --[{success, call_id, phone_index}]-->
Call Started?
  YES --> Prepare Call ID Update
    --[{lead_id, retell_call_id, call_status:'initiated', phone_X_status:'calling'}]-->
    Update Call ID in Sheet
  NO  --> Prepare Failed Update
    --[{lead_id, call_status, phone_X_status, next_call_date, ...}]-->
    Mark Call Failed
  --> Wait 15 min --> Loop Over Leads (next lead)
```

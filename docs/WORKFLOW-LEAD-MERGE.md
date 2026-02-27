# Workflow 5: Lead Merge

| Property         | Value                                                                                   |
|------------------|-----------------------------------------------------------------------------------------|
| **File**         | `workflow5-us-lead-merge.json`                                                          |
| **Name**         | US Leads - Lead Merge for Retell Calling                                                |
| **Nodes**        | 16                                                                                      |
| **Trigger**      | Manual (run on demand)                                                                  |
| **Sources**      | TradeIndia (active), EventsEye, 10Times, ExpoData (disabled - ready for connection)     |
| **Batch Size**   | 10 leads per Google Sheets append                                                       |

This workflow reads leads from multiple source spreadsheets, deduplicates them within the batch and against the existing master queue, extracts up to 3 phone numbers per lead, calculates priority scores, and appends new leads to the master queue with all 46 columns initialized.

---

## Flow Diagram

```
Manual Trigger
    |
    v
Wait 3 Seconds (rate limit buffer)
    |
    +---> Read Master Queue FIRST (existing leads)
    |         |
    |         v
    |     Wait For Both Paths (Merge node) <---+
    |                                          |
    +---> Read USA-TradeIndia ---+              |
    +---> Read USA-EventsEye* --+              |
    +---> Read USA-10Times* ----+--> Merge Sources --> Dedupe Within Batch
    +---> Read USA-ExpoData* ---+
                                               |
                                               v
                                    Compare & Transform
                                    (dedup vs master queue + build 46-col rows)
                                               |
                                               v
                                    Has New Leads?
                                      |YES              |NO
                                      v                 v
                                Split In Batches    No New Leads (stop)
                                      |
                                      v
                                Wait 2 Seconds
                                      |
                                      v
                                Append to Master Queue
                                      |
                                      v
                                (loop back to Split In Batches)

* = currently disabled, ready for connection
```

---

## Node-by-Node Breakdown

### Node 1: Manual Trigger

| Property       | Value                                                |
|----------------|------------------------------------------------------|
| **Type**       | `n8n-nodes-base.manualTrigger` (v1)                 |
| **Purpose**    | Run manually when new source data is available       |

---

### Node 2: Wait 3 Seconds

| Property       | Value                                                      |
|----------------|--------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.wait` (v1.1)                               |
| **Duration**   | 3 seconds                                                   |
| **Purpose**    | Rate limit buffer before parallel sheet reads               |

---

### Nodes 3-7: Read Source Sheets (Parallel)

After the wait, **5 sheets are read in parallel** — master queue and all source sheets:

| Node                      | Sheet                      | Status       | Purpose                        |
|---------------------------|----------------------------|--------------|---------------------------------|
| Read Master Queue FIRST   | `us-masterqueue` (Sheet1)  | **Active**   | Existing leads for dedup       |
| Read USA-TradeIndia       | `usa-tradeindia`           | **Active**   | Primary source                 |
| Read USA-EventsEye        | *(placeholder ID)*         | **Disabled** | Ready for connection           |
| Read USA-10Times          | *(placeholder ID)*         | **Disabled** | Ready for connection           |
| Read USA-ExpoData         | *(placeholder ID)*         | **Disabled** | Ready for connection           |

All use Google Sheets OAuth2 credential `to1E3r4Jf6AFsLfF`.

---

### Node 8: Merge Sources

| Property       | Value                                                        |
|----------------|--------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.merge` (v3.2)                               |
| **Inputs**     | 4 (TradeIndia + EventsEye + 10Times + ExpoData)             |
| **Purpose**    | Combine all source leads into a single stream                |

---

### Node 9: Dedupe Within Batch (Cross-Source Deduplication)

| Property       | Value                                                      |
|----------------|--------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                                 |
| **Purpose**    | Find and merge duplicates across source sheets              |

**Dedup Key Generation:**

The system generates multiple keys per lead to catch duplicates across different sources:

| Key Type                | Format                              | Example                            |
|-------------------------|-------------------------------------|------------------------------------|
| Website domain          | `web_` + domain root                | `web_eventtitans`                  |
| Event + city + month    | `evt_` + name + city + monthyear    | `evt_techexpo_lasvegas_202603`     |
| Event + venue           | `evt_` + name + venue               | `evt_techexpo_venetianexpo`        |
| Event + city            | `evt_` + name + city                | `evt_techexpo_lasvegas`            |
| Phone number            | `phone_` + last 10 digits           | `phone_2125551234`                 |
| Email                   | `email_` + address                  | `email_john@acme.com`             |

**Phone Extraction:**
- Extracts ALL valid US phones from each lead (comma-separated or array)
- Normalizes to E.164 format (`+1XXXXXXXXXX`)
- Validates: 10 digits, starts with 2-9, not all repeated digits, not 555-555-XXXX
- Collects up to **3 unique phones** across all duplicate leads in a group

**Merge Strategy:**
- Groups leads that share any dedup key
- Selects "best lead" from each group based on scoring: phones (×3) + emails (×1) + website (×2) + event name length
- Collects phones from ALL duplicates in the group (not just the best lead)
- Tracks: `_total_occurrences`, `_duplicates_removed`, `_unique_sources`, `_is_cross_source_dupe`

**Output Fields Added:**

| Field                    | Description                                          |
|--------------------------|------------------------------------------------------|
| `_phone_1`               | First unique US phone (E.164)                        |
| `_phone_2`               | Second unique US phone (may be empty)                |
| `_phone_3`               | Third unique US phone (may be empty)                 |
| `phone_count`            | Number of valid phones (1-3)                         |
| `_has_duplicates`        | Boolean: was this lead a duplicate group?            |
| `_total_occurrences`     | How many source leads merged into this one           |
| `_unique_sources`        | Comma-separated list of sources                      |
| `_is_cross_source_dupe`  | Boolean: found in multiple source sheets?            |

Output is sorted by `phone_count` descending (leads with more phones first).

---

### Node 10: Wait For Both Paths

| Property       | Value                                                        |
|----------------|--------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.merge` (v3.2)                               |
| **Inputs**     | 2 (Master Queue + Deduplicated Sources)                      |
| **Purpose**    | Synchronize both parallel paths before comparison            |

---

### Node 11: Compare & Transform (Core Logic)

| Property       | Value                                                        |
|----------------|--------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                                  |
| **Purpose**    | Dedup against master queue + build final 46-column rows      |

**Step 1: Build Existing Keys from Master Queue**
- Extracts all phone digits from `phone_1`, `phone_2`, `phone_3` fields
- Extracts all emails
- Determines last `lead_id` number for sequential ID generation

**Step 2: Filter Source Leads**

Each source lead must pass ALL checks:

| Check                    | Rule                                                    |
|--------------------------|---------------------------------------------------------|
| Has phone_1              | Must have at least one valid US phone                   |
| Phone_1 not in master    | Primary phone not already in master queue               |
| Phone_1 not in batch     | Primary phone not already added this run                |
| Phone_2 not in master    | Secondary phone (if exists) not in master               |
| Phone_3 not in master    | Tertiary phone (if exists) not in master                |
| Email not in master      | Email (if exists) not already in master queue           |

**Step 3: Generate Lead ID**
- Format: `USL-XXXXX` (zero-padded 5 digits)
- Continues from the highest existing ID in master queue
- Example: If last ID is `USL-00042`, next new lead gets `USL-00043`

**Step 4: Calculate Priority Score**
- See [Priority Scoring](#priority-scoring) section below

**Step 5: Build 46-Column Row**

All 46 columns are initialized. Call-related fields are set to empty strings. Key fields:

| Field                                     | Value                                             |
|-------------------------------------------|---------------------------------------------------|
| `lead_id`                                 | Generated `USL-XXXXX`                             |
| `name`                                    | Contact person / organizer name / "Event Organizer"|
| `company`                                 | Organizer / company name                          |
| `phone_1` / `phone_2` / `phone_3`        | Extracted E.164 numbers                           |
| `current_phone_index`                     | `1`                                               |
| `phone_1_status`                          | `pending`                                         |
| `phone_2_status`                          | `pending` (if phone_2 exists) or empty            |
| `phone_3_status`                          | `pending` (if phone_3 exists) or empty            |
| `email`                                   | Primary email                                     |
| `event_name` / `event_date` / `venue` / `city` | From source data                            |
| `added_date`                              | Today's date (`YYYY-MM-DD`)                       |
| `priority_score`                          | Calculated score                                  |
| `call_status`                             | `pending`                                         |
| `call_attempts`                           | `0`                                               |

**If No New Leads:** Returns `{ _status: 'NO_NEW_LEADS' }` instead.

---

### Node 12: Has New Leads?

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.if` (v2.2)               |
| **Condition**  | `_status !== 'NO_NEW_LEADS'`              |
| **TRUE**       | Split In Batches                          |
| **FALSE**      | No New Leads (stop)                       |

---

### Node 13: Split In Batches

| Property        | Value                                                    |
|-----------------|----------------------------------------------------------|
| **Type**        | `n8n-nodes-base.splitInBatches` (v3)                    |
| **Batch Size**  | 10                                                       |
| **Purpose**     | Append leads in batches to avoid API rate limits         |

---

### Node 14: Wait 2 Seconds

| Property       | Value                                                      |
|----------------|--------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.wait` (v1.1)                               |
| **Duration**   | 2 seconds                                                   |
| **Purpose**    | Rate limit buffer between Google Sheets appends             |

---

### Node 15: Append to Master Queue

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.googleSheets` (v4.5)     |
| **Operation**  | Append rows                               |
| **Sheet**      | `us-masterqueue` (Sheet1)                 |
| **Mapping**    | `autoMapInputData`                        |

Appends all 46 columns per row. Loops back to Split In Batches for next batch.

---

### Node 16: No New Leads

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.noOp` (v1)               |
| **Purpose**    | Dead end when no new leads found          |

---

## Priority Scoring

Priority determines calling order in WF1. Higher score = called first.

### Event Date Proximity Scores

| Days Until Event  | Base Score                              |
|-------------------|-----------------------------------------|
| ≤ 7 days          | 500                                     |
| 8-14 days         | 450                                     |
| 15-30 days        | 400                                     |
| 31-60 days        | 300                                     |
| 61-90 days        | 250                                     |
| 91-120 days       | 200                                     |
| 121-180 days      | 150                                     |
| 180+ days         | 100-150 (decreasing by 10 per 30 days) |
| No date           | 100                                     |

### Phone Count Bonus

| Phones Available  | Bonus                 |
|-------------------|-----------------------|
| 1 phone           | +100                  |
| 2 phones          | +150 (100 + 50)       |
| 3 phones          | +175 (100 + 50 + 25)  |
| 0 phones          | +0 (lead is rejected) |

### Past Event Handling

| Condition                 | Score Cap              |
|---------------------------|------------------------|
| Past event (any)          | Hard capped at **50**  |
| Past event phone bonus    | Capped at **25**       |

Past events can never outrank any future event.

### Example Scores

| Lead                           | Calculation        | Score              |
|--------------------------------|--------------------|--------------------|
| Event in 5 days, 3 phones     | 500 + 175          | **675**            |
| Event in 45 days, 1 phone     | 300 + 100          | **400**            |
| Event in 45 days, 2 phones    | 300 + 150          | **450**            |
| No date, 1 phone              | 100 + 100          | **200**            |
| Past event, 3 phones          | 40 + 25 (capped)   | **50** (hard cap)  |

---

## Date Parsing

The Compare & Transform node handles multiple date formats:

| Format                    | Example                                                                |
|---------------------------|------------------------------------------------------------------------|
| ISO                       | `2026-03-15`                                                           |
| US format                 | `03/15/2026`                                                           |
| Named month               | `March 15, 2026`                                                       |
| Short month               | `15 Mar 2026`                                                          |
| Range (uses END date)     | `Fri, 30 Jan, 2026 - Sun, 01 Feb, 2026` → parses `01 Feb 2026`       |
| Month + Year only         | `March 2026` → defaults to 15th                                       |

Range separators supported: ` - `, ` to `, ` – `, ` — `

When a date range is provided, the **end date** is used for priority scoring (event is relevant until it's over).

---

## Complete Data Flow Summary

```
Manual Trigger
  --[trigger]-->
Wait 3 Seconds
  --[parallel reads]-->

  Path A: Read Master Queue FIRST
    --[existing leads with lead_id]-->
    Wait For Both Paths (input 0)

  Path B: Read Sources (TradeIndia + disabled sources)
    --[raw source leads]-->
    Merge Sources
    --[combined leads]-->
    Dedupe Within Batch
    --[{_phone_1, _phone_2, _phone_3, phone_count, dedup metadata}]-->
    Wait For Both Paths (input 1)

Wait For Both Paths
  --[master queue + deduplicated sources combined]-->
Compare & Transform
  --[{46-column rows with lead_id, priority_score, phones, statuses}]-->
Has New Leads?
  YES --> Split In Batches (10 per batch)
    --> Wait 2 Seconds
    --> Append to Master Queue
    --> (loop for next batch)
  NO  --> No New Leads (stop)
```

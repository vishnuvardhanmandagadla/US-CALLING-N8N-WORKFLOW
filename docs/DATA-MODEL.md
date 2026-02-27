# Data Model - Google Sheet Reference

## Google Sheet: `US_Leads_Master_Queue` (46 Columns)

| #    | Column                 | Set By         | Description                                       |
|------|------------------------|----------------|---------------------------------------------------|
| 1    | `lead_id`              | WF5            | Unique ID (USL-00001)                             |
| 2    | `name`                 | WF5            | Contact name                                      |
| 3    | `company`              | WF5            | Company/organizer name                            |
| 4    | `phone_1`              | WF5            | Primary phone (E.164: +1XXXXXXXXXX)               |
| 5    | `phone_2`              | WF5            | Secondary phone (may be empty)                    |
| 6    | `phone_3`              | WF5            | Tertiary phone (may be empty)                     |
| 7    | `current_phone_index`  | WF1, Merged    | Which phone to try next (1, 2, or 3)             |
| 8    | `phone_1_status`       | WF1, Merged    | Status of phone_1                                 |
| 9    | `phone_2_status`       | WF1, Merged    | Status of phone_2                                 |
| 10   | `phone_3_status`       | WF1, Merged    | Status of phone_3                                 |
| 11   | `email`                | WF5            | Primary email                                     |
| 12   | `event_name`           | WF5            | Event/fair name                                   |
| 13   | `event_date`           | WF5            | Event date                                        |
| 14   | `venue`                | WF5            | Event venue                                       |
| 15   | `city`                 | WF5            | Event city                                        |
| 16   | `added_date`           | WF5            | Date lead was added                               |
| 17   | `priority_score`       | WF5            | Calling priority (event proximity + phone bonus)  |
| 18   | `call_status`          | WF1, Merged    | Current status                                    |
| 19   | `call_attempts`        | WF1, Merged    | Total attempts across all phones                  |
| 20   | `last_call_date`       | WF1, Merged    | Last call timestamp                               |
| 21   | `next_call_date`       | WF1, Merged    | When to try next                                  |
| 22   | `retell_call_id`       | WF1, Merged    | Retell call ID                                    |
| 23   | `call_duration`        | Merged         | Call duration (seconds)                           |
| 24   | `call_transcript`      | Merged         | Full transcript                                   |
| 25   | `recording_url`        | Merged         | Call recording URL                                |
| 26   | `user_sentiment`       | Merged (anlzd) | AI-detected sentiment                            |
| 27   | `disconnection_reason` | Merged         | How call ended                                    |
| 28   | `call_outcome`         | Merged (anlzd) | Outcome (CONNECTED/FAILED/DECLINED/etc.)         |
| 29   | `interest_level`       | Merged (anlzd) | HOT/WARM/CALLBACK/COLD/NOT_INTERESTED            |
| 30   | `call_summary`         | Merged (anlzd) | AI-generated summary                             |
| 31   | `meeting_scheduled`    | Merged (anlzd) | Yes/No                                           |
| 32   | `email_captured`       | Merged (anlzd) | Email captured during call                       |
| 33   | `objection_reason`     | Merged (anlzd) | Objections raised                                |
| 34   | `callback_time`        | Merged (anlzd) | Requested callback time                          |
| 35   | `decision_maker`       | Merged (anlzd) | Yes/empty                                        |
| 36   | `pain_points`          | Merged (anlzd) | Pain points identified                           |
| 37   | `next_steps`           | Merged (anlzd) | Recommended next steps                           |
| 38   | `follow_up_status`     | Merged (anlzd) | urgent/pending_review/scheduled_callback/closed  |
| 39   | `alert_sent`           | Merged (anlzd) | TRUE if alert was sent                           |
| 40   | `alert_sent_date`      | Merged (anlzd) | When alert was sent                              |
| 41   | `demo_scheduled`       | WF5, Merged    | Demo scheduling flag                             |
| 42   | `call_successful`      | Merged (anlzd) | Yes/No                                           |
| 43   | `call_cost`            | Merged (anlzd) | Call cost from Retell                            |
| 44   | `agent_id`             | Merged (anlzd) | Retell agent ID used                             |
| 45   | `phone_captured`       | Merged (anlzd) | Phone number captured during call                |
| 46   | `is_hot`               | Merged (anlzd) | TRUE/FALSE - hot lead flag                       |

---

## Status Values

### Phone Status

| Status      | Meaning                          |
|-------------|----------------------------------|
| `pending`   | Not yet called                   |
| `calling`   | Call in progress                 |
| `connected` | Successfully connected           |
| `voicemail` | Reached voicemail                |
| `no_answer` | No answer                        |
| `busy`      | Line busy                        |
| `failed`    | Call failed (technical)          |
| `invalid`   | Invalid number                   |
| `declined`  | User declined/spam               |
| `retry`     | Pending retry (API/concurrency)  |

### Call Status

| Status               | Meaning                                 |
|----------------------|-----------------------------------------|
| `pending`            | New lead, not yet called                |
| `calling`            | Currently being called                  |
| `initiated`          | Retell API accepted, call starting      |
| `scheduled`          | Scheduled for retry (same phone)        |
| `retry`              | Retry pending (API failure)             |
| `callback_scheduled` | Callback requested by prospect          |
| `completed`          | Call connected successfully             |
| `not_interested`     | Declined/spam - stop calling            |
| `exhausted`          | All phones/attempts used up             |

### Follow-Up Status

| Status               | Meaning                                 |
|----------------------|-----------------------------------------|
| `pending_review`     | Connected call needs human review       |
| `scheduled_callback` | Retry or next phone scheduled           |
| `closed`             | Declined/spam - no follow-up            |
| `exhausted`          | All retry options used                  |

---

## Write Operations by Column

> WRITE = initial creation | UPDATE = modifies existing row | READ = reads but doesn't write

| Column                 | WF5 (Lead Merge) | WF1 (Call Trigger) | Merged - call_ended | Merged - call_analyzed |
|------------------------|:-----------------:|:------------------:|:-------------------:|:----------------------:|
| `lead_id`              | WRITE             | READ (match)       | READ (match)        | READ (match)           |
| `name`                 | WRITE             | -                  | -                   | -                      |
| `company`              | WRITE             | -                  | -                   | -                      |
| `phone_1`              | WRITE             | READ               | -                   | -                      |
| `phone_2`              | WRITE             | READ               | -                   | -                      |
| `phone_3`              | WRITE             | READ               | -                   | -                      |
| `current_phone_index`  | WRITE (`1`)       | UPDATE             | -                   | UPDATE                 |
| `phone_1_status`       | WRITE (`pending`) | UPDATE             | -                   | UPDATE                 |
| `phone_2_status`       | WRITE (or empty)  | UPDATE             | -                   | UPDATE                 |
| `phone_3_status`       | WRITE (or empty)  | UPDATE             | -                   | UPDATE                 |
| `email`                | WRITE             | -                  | -                   | -                      |
| `event_name`           | WRITE             | READ               | -                   | -                      |
| `event_date`           | WRITE             | READ               | -                   | -                      |
| `venue`                | WRITE             | READ               | -                   | -                      |
| `city`                 | WRITE             | READ               | -                   | -                      |
| `added_date`           | WRITE             | -                  | -                   | -                      |
| `priority_score`       | WRITE             | READ (sort)        | -                   | -                      |
| `call_status`          | WRITE (`pending`) | UPDATE             | UPDATE              | UPDATE                 |
| `call_attempts`        | WRITE (`0`)       | UPDATE (+1)        | -                   | UPDATE                 |
| `last_call_date`       | -                 | UPDATE             | UPDATE              | -                      |
| `next_call_date`       | -                 | UPDATE             | -                   | UPDATE                 |
| `retell_call_id`       | -                 | UPDATE             | UPDATE              | -                      |
| `call_duration`        | -                 | -                  | UPDATE              | UPDATE                 |
| `call_transcript`      | -                 | -                  | UPDATE              | UPDATE                 |
| `recording_url`        | -                 | -                  | UPDATE              | UPDATE                 |
| `user_sentiment`       | -                 | -                  | -                   | UPDATE                 |
| `disconnection_reason` | -                 | -                  | UPDATE              | UPDATE                 |
| `call_outcome`         | -                 | -                  | -                   | UPDATE                 |
| `interest_level`       | -                 | -                  | -                   | UPDATE                 |
| `call_summary`         | -                 | -                  | -                   | UPDATE                 |
| `meeting_scheduled`    | -                 | -                  | -                   | UPDATE                 |
| `email_captured`       | -                 | -                  | -                   | UPDATE                 |
| `objection_reason`     | -                 | -                  | -                   | UPDATE                 |
| `callback_time`        | -                 | -                  | -                   | UPDATE                 |
| `decision_maker`       | -                 | -                  | -                   | UPDATE                 |
| `pain_points`          | -                 | -                  | -                   | UPDATE                 |
| `next_steps`           | -                 | -                  | -                   | UPDATE                 |
| `follow_up_status`     | -                 | -                  | -                   | UPDATE                 |
| `alert_sent`           | -                 | -                  | -                   | UPDATE                 |
| `alert_sent_date`      | -                 | -                  | -                   | UPDATE                 |
| `demo_scheduled`       | WRITE (init)      | -                  | -                   | UPDATE                 |
| `call_successful`      | -                 | -                  | -                   | UPDATE                 |
| `call_cost`            | -                 | -                  | -                   | UPDATE                 |
| `agent_id`             | -                 | -                  | -                   | UPDATE                 |
| `phone_captured`       | -                 | -                  | -                   | UPDATE                 |
| `is_hot`               | -                 | -                  | -                   | UPDATE                 |

---

## Write Operations by Node

| Node Name                  | Workflow         | Operation | Columns Written                                                                                                   |
|----------------------------|------------------|-----------|-------------------------------------------------------------------------------------------------------------------|
| Append to Master Queue     | WF5              | Append    | All 46 columns (initial values)                                                                                   |
| Update Status to Calling   | WF1              | Update    | `call_status`, `call_attempts`, `last_call_date`, `current_phone_index`                                           |
| Update Call ID in Sheet    | WF1              | Update    | `retell_call_id`, `call_status`, `phone_X_status`                                                                 |
| Mark Call Failed           | WF1              | Update    | `call_status`, `call_attempts`, `last_call_date`, `current_phone_index`, `phone_X_status`, `next_call_date`, `follow_up_status` |
| Update Sheet with Call Data| Merged (ended)   | Update    | `retell_call_id`, `call_status`, `call_duration`, `disconnection_reason`, `recording_url`, `call_transcript`, `last_call_date`  |
| Update All Extracted Data  | Merged (analyzed)| Update    | `call_status`, `agent_id`, `current_phone_index`, `phone_X_status`, `call_outcome`, `call_attempts`, `call_successful`, `next_call_date`, `call_cost`, `call_summary`, `user_sentiment`, `interest_level`, `objection_reason`, `callback_time`, `decision_maker`, `pain_points`, `next_steps`, `follow_up_status`, `meeting_scheduled`, `email_captured`, `phone_captured`, `demo_scheduled`, `is_hot` |
| Mark Alert Sent            | Merged (analyzed)| Update    | `alert_sent`, `alert_sent_date`                                                                                   |

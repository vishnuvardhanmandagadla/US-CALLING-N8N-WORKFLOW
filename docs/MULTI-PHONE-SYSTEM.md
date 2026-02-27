# Multi-Phone Fallback System

Each lead can have up to 3 phone numbers (`phone_1`, `phone_2`, `phone_3`), each tracked independently.

## Phone Status Lifecycle

```
pending --> calling --> connected (success!)
                   --> voicemail (retry same phone in 4 hours)
                   --> no_answer (retry same phone in 2 hours)
                   --> busy (retry same phone in 1 hour)
                   --> failed (try next phone in 1 hour)
                   --> invalid (try next phone immediately)
                   --> declined (stop ALL phones)
```

## Fallback Logic

| Scenario                       | Action                    | Delay        |
|--------------------------------|---------------------------|--------------|
| Phone failed/invalid           | Try next phone            | 1 hour       |
| Voicemail/Machine detected     | Retry same phone          | 4 hours      |
| No answer/Timeout              | Retry same phone          | 2 hours      |
| Busy                           | Retry same phone          | 1 hour       |
| Declined/Spam/Scam             | Stop all phones           | Never retry  |
| Connected                      | Success                   | -            |
| All 3 phones exhausted         | Mark as `exhausted`       | -            |

## Max Attempts

**9 total** (3 phones x 3 attempts each) before `call_status = 'exhausted'`

---

## Disconnection Reason Map

This table maps every Retell `disconnection_reason` to a phone-level action. Used by the "Extract All Data" node in the Merged Webhook.

| Retell Reason                            | Phone Status  | Can Retry | Delay    | Try Next | Stop All | Connected |
|------------------------------------------|---------------|-----------|----------|----------|----------|-----------|
| `user_hangup`                            | connected     | No        | -        | No       | No       | Yes       |
| `agent_hangup`                           | connected     | No        | -        | No       | No       | Yes       |
| `call_transfer`                          | connected     | No        | -        | No       | No       | Yes       |
| `inactivity`                             | connected     | No        | -        | No       | No       | Yes       |
| `max_duration_reached`                   | connected     | No        | -        | No       | No       | Yes       |
| `voicemail_reached`                      | voicemail     | Yes       | 4 hrs    | No       | No       | No        |
| `machine_detected`                       | voicemail     | Yes       | 4 hrs    | No       | No       | No        |
| `dial_no_answer`                         | no_answer     | Yes       | 2 hrs    | No       | No       | No        |
| `registered_call_timeout`                | no_answer     | Yes       | 2 hrs    | No       | No       | No        |
| `dial_busy`                              | busy          | Yes       | 1 hr     | No       | No       | No        |
| `dial_failed`                            | failed        | No        | -        | Yes      | No       | No        |
| `invalid_destination`                    | invalid       | No        | -        | Yes      | No       | No        |
| `telephony_provider_permission_denied`   | failed        | No        | -        | Yes      | No       | No        |
| `telephony_provider_unavailable`         | failed        | No        | -        | Yes      | No       | No        |
| `sip_routing_error`                      | failed        | No        | -        | Yes      | No       | No        |
| `user_declined`                          | declined      | No        | -        | No       | Yes      | No        |
| `marked_as_spam`                         | declined      | No        | -        | No       | Yes      | No        |
| `scam_detected`                          | declined      | No        | -        | No       | Yes      | No        |
| `no_valid_payment`                       | failed        | No        | -        | No       | Yes      | No        |
| `concurrency_limit_reached`              | retry         | Yes       | 30 min   | No       | No       | No        |

---

## Call Outcome Decision Logic

After the disconnection reason is mapped, the system determines the overall call outcome using this priority order:

| Priority | Condition                                    | call_status          | call_outcome       | call_successful | follow_up_status     | next_call_date |
|----------|----------------------------------------------|----------------------|--------------------|-----------------|----------------------|----------------|
| 1        | `stopAll = true`                             | `not_interested`     | DECLINED           | No              | closed               | (empty)        |
| 2        | `connected = true`                           | `completed`          | CONNECTED          | Yes             | pending_review       | (empty)        |
| 3        | `tryNext AND phoneIndex < 3`                 | `scheduled`          | FAILED             | No              | scheduled_callback   | +1 hour        |
| 4        | `tryNext AND phoneIndex >= 3`                | `exhausted`          | FAILED             | No              | exhausted            | (empty)        |
| 5        | `canRetry AND attempts >= 3 AND idx < 3`     | `scheduled`          | (status uppercase) | No              | scheduled_callback   | +1 hour        |
| 6        | `canRetry AND attempts >= 3 AND idx >= 3`    | `exhausted`          | (status uppercase) | No              | exhausted            | (empty)        |
| 7        | `canRetry AND attempts < 3`                  | `scheduled`          | (status uppercase) | No              | scheduled_callback   | +delay hours   |
| 8        | Default                                      | `completed`          | UNKNOWN            | No              | pending_review       | (empty)        |
| 9        | `callbackTime` parsed successfully           | `callback_scheduled` | (unchanged)        | (unchanged)     | scheduled_callback   | Parsed time    |

> **Note:** Priority 9 (callback override) runs AFTER the main outcome logic. If the user requested a specific callback time during the call and it can be parsed, it overrides `call_status`, `next_call_date`, and `follow_up_status` regardless of the disconnection reason (except for `stopAll` and `connected` outcomes).

---

## Callback Time Parsing

When a user says "call me back at 3 PM tomorrow", the system extracts `callback_time` from Retell's `custom_analysis_data` and parses it to schedule the retry at the exact requested time.

**Supported Formats:**

| Format                      | Example                         | Parsed As                          |
|-----------------------------|---------------------------------|------------------------------------|
| ISO / standard date string  | `2026-03-15T15:00:00`           | Direct parse                       |
| Named date + time           | `March 15, 2026 3:00 PM`       | Direct parse                       |
| Tomorrow + time             | `tomorrow at 3pm`               | Tomorrow at 15:00                  |
| Relative hours              | `in 2 hours`                    | Now + 2 hours                      |
| Relative minutes            | `in 30 minutes`                 | Now + 30 minutes                   |
| Time only (today/tomorrow)  | `3 PM`                          | Today at 15:00 if future, else tomorrow |

If parsing fails or the time is in the past, the system falls back to the standard delay-based scheduling.

---

## Hot Lead Detection

A lead is flagged as "hot" (triggering email alerts) when ANY of these are true:

```
is_hot = (interest_level === 'HOT' ||
          interest_level === 'CALLBACK' ||
          meeting_scheduled === true ||
          tool_calls contain meeting/schedule/calendar)
```

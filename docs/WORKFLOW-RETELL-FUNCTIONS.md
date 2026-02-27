# Workflow 4: Retell Functions

| Property            | Value                                     |
|---------------------|-------------------------------------------|
| **File**            | `workflow4-retell-functions.json`         |
| **Name**            | US Leads - Retell Functions               |
| **Nodes**           | 11                                        |
| **Trigger**         | Webhook (POST to `/retell-functions`)     |
| **Response Mode**   | Synchronous (`responseNode`)              |

This workflow handles **real-time function calls** from Retell AI during a live phone conversation. When the AI agent needs to check calendars, book demos, or retrieve pricing, Retell sends a POST request to this webhook and waits for the JSON response.

---

## Flow Diagram

```
Retell AI (during live call)
    |
    | POST /retell-functions
    v
Function Webhook
    |
    v
Parse Function Request
    |
    v
Route by Function Name (Switch)
    |               |               |               |
    | check_calendar| book_demo     | get_pricing   | (fallback)
    v               v               v               v
Check Calendar   Book Demo      Get Pricing     Unknown Function
    |               |               |               |
    v               v               v               v
Respond          Respond         Respond         Respond
(Calendar)       (Demo)          (Pricing)       (Unknown)
```

---

## Node-by-Node Breakdown

### Node 1: Function Webhook

| Property            | Value                                             |
|---------------------|---------------------------------------------------|
| **Type**            | `n8n-nodes-base.webhook` (v2)                    |
| **Path**            | `/retell-functions`                               |
| **Method**          | POST                                              |
| **Response Mode**   | `responseNode` (waits for a Respond node)         |

**Incoming Payload from Retell:**
```json
{
  "name": "check_calendar",
  "arguments": {
    "time": "next Tuesday at 2 PM",
    "name": "John Smith",
    "email": "john@example.com"
  }
}
```

---

### Node 2: Parse Function Request

| Property       | Value                                                      |
|----------------|------------------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                                |
| **Purpose**    | Extract function name and arguments, set routing flags     |

**Output Fields:**

| Field                | Source                                                           |
|----------------------|------------------------------------------------------------------|
| `function_name`      | `body.name` or `body.function_name` (lowercased, trimmed)       |
| `arguments`          | `body.arguments` or `body.args`                                  |
| `is_check_calendar`  | `true` if function_name === `check_calendar`                     |
| `is_book_demo`       | `true` if function_name === `book_demo`                          |
| `is_get_pricing`     | `true` if function_name === `get_pricing`                        |

---

### Node 3: Route by Function Name

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.switch` (v3)             |
| **Fallback**   | `extra` output (Unknown Function)         |

| Output             | Condition                          | Destination        |
|--------------------|------------------------------------|--------------------|
| `check_calendar`   | `is_check_calendar === true`       | Check Calendar     |
| `book_demo`        | `is_book_demo === true`            | Book Demo          |
| `get_pricing`      | `is_get_pricing === true`          | Get Pricing        |
| Fallback           | No match                           | Unknown Function   |

---

### Node 4: Check Calendar

| Property       | Value                                          |
|----------------|------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                    |
| **Purpose**    | Generate next available demo time slots        |

**Logic:**
- Generates slots for the next 7 weekdays (skips weekends)
- Two slots per day: **10:00 AM EST** and **2:00 PM EST**
- Returns up to **6 slots**

**Response Example:**
```
"Available demo times: Monday, March 3 at 10:00 AM EST,
Monday, March 3 at 2:00 PM EST, Tuesday, March 4 at 10:00 AM EST, ...
Which works best for you?"
```

---

### Node 5: Book Demo

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)               |
| **Purpose**    | Confirm a demo booking                    |

**Input Arguments:**

| Argument                  | Fallback                   |
|---------------------------|----------------------------|
| `time` / `slot`           | `"the requested time"`     |
| `name` / `contact_name`   | `""`                       |
| `email` / `contact_email`  | `""`                      |

**Response Fields:**

| Field              | Value                                                  |
|--------------------|--------------------------------------------------------|
| `success`          | `true`                                                 |
| `message`          | Confirmation message with time and optional email      |
| `booked_time`      | The requested time slot                                |
| `contact_name`     | Contact name if provided                               |
| `contact_email`    | Contact email if provided                              |

---

### Node 6: Get Pricing

| Property       | Value                                     |
|----------------|-------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)               |
| **Purpose**    | Return EventTitans pricing tiers          |

**Pricing Tiers Returned:**

| Tier     | Range                           | Price              |
|----------|---------------------------------|--------------------|
| Small    | Up to 5,000 attendees           | From $500          |
| Medium   | 5,000 to 20,000 attendees      | Custom             |
| Large    | 20,000+ attendees               | Enterprise pricing |

Also recommends scheduling a demo for an exact quote.

---

### Node 7: Unknown Function

| Property       | Value                                              |
|----------------|----------------------------------------------------|
| **Type**       | `n8n-nodes-base.code` (v2)                        |
| **Purpose**    | Fallback for unrecognized function names           |

Returns a generic message offering to have the team follow up on the question.

---

### Nodes 8-11: Respond Nodes

| Node                  | Responds To        |
|-----------------------|--------------------|
| Respond (Calendar)    | Check Calendar     |
| Respond (Demo)        | Book Demo          |
| Respond (Pricing)     | Get Pricing        |
| Respond (Unknown)     | Unknown Function   |

All respond nodes:
- **Type:** `n8n-nodes-base.respondToWebhook` (v1.1)
- **Response:** JSON (`$json.result` serialized)

---

## Functions Summary

| Function           | Purpose                          | Arguments                | Response                                       |
|--------------------|----------------------------------|--------------------------|-------------------------------------------------|
| `check_calendar`   | List available demo slots        | None                     | Up to 6 time slots (weekdays, 10AM/2PM EST)    |
| `book_demo`        | Confirm a demo booking           | `time`, `name`, `email`  | Confirmation with booked time                   |
| `get_pricing`      | Return pricing information       | None                     | 3-tier pricing breakdown                        |
| *(unknown)*        | Fallback for any other function  | Any                      | Generic follow-up message                       |

---

## Complete Data Flow Summary

```
Retell AI (live call)
  --[POST {name, arguments}]-->
Function Webhook
  --[raw body]-->
Parse Function Request
  --[{function_name, arguments, is_check_calendar, is_book_demo, is_get_pricing}]-->
Route by Function Name
  --[matched output]-->
  check_calendar --> Check Calendar --[{available_slots, message}]--> Respond (Calendar)
  book_demo      --> Book Demo --[{success, message, booked_time}]--> Respond (Demo)
  get_pricing    --> Get Pricing --[{message, pricing_tiers}]--> Respond (Pricing)
  (fallback)     --> Unknown Function --[{message}]--> Respond (Unknown)
  --[JSON response back to Retell AI]-->
```

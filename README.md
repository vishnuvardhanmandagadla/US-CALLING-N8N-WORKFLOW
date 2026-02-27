# EventTitans - Retell AI Cold Calling System

Automated cold calling system for US leads using **Retell AI** + **n8n** + **Google Sheets** with **multi-phone fallback** (up to 3 phones per lead).

---

## Architecture Overview

```
[WF5: Lead Merge]  -->  [Google Sheet: Master Queue]  -->  [WF1: Call Trigger]
                              (46 columns)                       |
                                                          [Retell AI Cloud]
                                                                 |
                                                    [Merged Webhook] ── call_ended branch
                                                          |          ── call_analyzed branch
                                                          |
                                                    [WF4: Retell Functions] (live call tools)
```

---

## Workflow Files

| File                                    | Workflow               | Trigger                | Status                   |
|-----------------------------------------|------------------------|------------------------|--------------------------|
| `workflow1-call-trigger.json`           | Call Trigger           | Schedule (30 min)      | Active                   |
| `workflow-merged-webhook.json`          | Merged Retell Webhook  | POST `/retell-webhook` | Active                   |
| `workflow4-retell-functions.json`       | Retell Functions       | POST `/retell-functions`| Active                  |
| `workflow5-us-lead-merge.json`          | Lead Merge             | Manual                 | Active                   |
| ~~`workflow2-call-ended.json`~~         | ~~Call Ended~~         | -                      | Replaced by Merged       |
| ~~`workflow3-call-analyzed.json`~~      | ~~Call Analyzed~~      | -                      | Replaced by Merged       |

---

## Documentation

| Document                                                        | What it covers                                                                       |
|-----------------------------------------------------------------|--------------------------------------------------------------------------------------|
| [Architecture & Lifecycle](docs/ARCHITECTURE.md)                | System diagrams, how flows interact, full execution lifecycle, credentials            |
| [Data Model](docs/DATA-MODEL.md)                                | 46-column Google Sheet reference, status values, which workflow writes which column   |
| [Multi-Phone System](docs/MULTI-PHONE-SYSTEM.md)               | Phone fallback logic, retry rules, disconnection reason map, call outcome decisions  |
| [WF1: Call Trigger](docs/WORKFLOW-CALL-TRIGGER.md)              | Node-by-node breakdown, data flow, filter logic, Retell API call                     |
| [Merged Webhook](docs/WORKFLOW-MERGED-WEBHOOK.md)               | Both branches (call_ended + call_analyzed), analysis extraction, alert system        |
| [WF4: Retell Functions](docs/WORKFLOW-RETELL-FUNCTIONS.md)      | Live call functions: calendar, demo booking, pricing                                 |
| [WF5: Lead Merge](docs/WORKFLOW-LEAD-MERGE.md)                  | Source dedup, phone extraction, priority scoring, date parsing                       |
| [Setup Guide](docs/SETUP-GUIDE.md)                              | Import instructions, credential setup, webhook config, testing checklist             |

---

## Quick Start

1. **Import** all workflow JSON files into n8n ([Setup Guide](docs/SETUP-GUIDE.md))
2. **Configure** Google Sheet with 46 columns ([Data Model](docs/DATA-MODEL.md))
3. **Set** Retell webhooks to point to your n8n instance ([Setup Guide](docs/SETUP-GUIDE.md#webhook-configuration))
4. **Run** WF5 manually to populate leads, then activate WF1 + Merged Webhook + WF4

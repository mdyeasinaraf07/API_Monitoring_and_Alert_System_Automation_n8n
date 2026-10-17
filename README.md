# API Monitor System — Intelligent API Monitoring & Alert System

An API health monitoring and AI-powered alerting system built on **n8n** (a Node.js-based
workflow automation platform), **MongoDB Atlas**, and **Google Gemini** for alert text
generation. The system ingests batches of API health results, detects anomalies by rule,
deduplicates repeat issues, writes AI-generated human-readable alerts to a database,
and exposes REST endpoints to list and resolve active alerts. A static frontend displays
alerts in real time.

## Why n8n instead of a hand-written Express server

The task recommends Node.js/Express. n8n itself runs on Node.js, and all business logic
in this project (validation, anomaly detection, deduplication, AI alert generation, error
handling) is written as plain JavaScript inside n8n Code nodes — the same language, a
different execution shell. n8n was chosen because it gives a visually inspectable,
quickly testable implementation of the same REST + database + LLM + error-handling
architecture the task asks for, without sacrificing any functional requirement.

## Architecture

```
                         ┌─────────────────────┐
  POST /monitor  ──────▶ │  01 Monitor Intake   │
                         │  validate → detect   │
                         │  → dedupe → store     │
                         └─────────┬────────────┘
                                   │ new alert
                                   ▼
                         ┌─────────────────────┐
                         │ 02 AI Alert Writer   │──▶ Google Gemini
                         │ (LLM, with fallback) │
                         └─────────┬────────────┘
                                   ▼
                         ┌─────────────────────┐
                         │   MongoDB Atlas      │
                         │ alerts / logs        │
                         └─────────┬────────────┘
                                   ▲
   GET /alerts   ─────────────────┤
   POST /resolve ─────────────────┘
                         ┌─────────────────────┐
  any workflow error ──▶ │ 06 Error Handler     │──▶ log + email
                         └─────────────────────┘
```

### Workflows

| # | Workflow | Trigger | Responsibility |
|---|---|---|---|
| 01 | Monitor Intake | `POST /monitor` | Validates the batch, detects anomalies by rule, deduplicates against active alerts, writes to MongoDB, returns a summary |
| 02 | AI Alert Writer | Sub-workflow | Asks Gemini for a short human-readable alert; falls back to a template message if the model fails or returns something unusable |
| 03 | Write Log | Sub-workflow | Writes one structured entry to the `logs` collection — called by every other workflow |
| 04 | Get Alerts | `GET /alerts` | Returns all active alerts, newest first |
| 05 | Resolve Alert | `POST /resolve` | Marks an alert resolved (400 if `alertId` missing, 404 if not an active alert) |
| 06 | Error Handler | n8n Error Trigger | Fires automatically when 01/04/05 fail; logs the error and emails an alert |

## Anomaly detection rules

| Condition | Issue type | Severity contribution |
|---|---|---|
| `status_code >= 500` | `REQUEST_FAILED` | critical |
| `400 <= status_code < 500` | `REQUEST_FAILED` | high |
| `response_time_ms > 5000` | `HIGH_RESPONSE_TIME` | high |
| `response_time_ms > 3000` | `HIGH_RESPONSE_TIME` | medium |
| `records_returned` missing or `0` | `NO_RECORDS` | medium |
| malformed `response_time_ms` / `status_code` / `records_returned` | `INVALID_DATA` | high |
| `api_name` missing | — | row rejected, not alerted |

An alert's overall severity is the highest severity among its issues. Repeat anomalies
for the same API + issue combination (`alertKey = apiName|ISSUE+ISSUE`) update the
existing active alert's `occurrences` count instead of creating a duplicate, and skip
the LLM call entirely.

## Error handling

1. **Bad input** → `400` with a plain-text reason, no DB write for the batch.
2. **LLM failure or unusable output** → a template-built fallback message is used instead
   (`messageSource: "fallback"`); the batch still completes.
3. **Any other failure** (e.g. MongoDB unreachable) → n8n's Error Trigger (workflow 06)
   fires automatically: it logs the error to MongoDB and emails an alert with the
   workflow name, last node, error message, and a link to the failed execution.

## Setup

### 1. MongoDB Atlas
1. Create a free M0 cluster and a database named `api_monitor`.
2. Create a database user scoped to `readWrite` on `api_monitor` only.
3. Allow network access from `0.0.0.0/0` (required for n8n Cloud, whose outbound IPs
   are not fixed).
4. Create two collections: `alerts` and `logs`.

### 2. n8n
1. Import the six workflow files in `workflows/` into an n8n instance (Cloud or
   self-hosted; n8n runs on Node.js).
2. Create a MongoDB credential named **MongoDB Atlas** using your Atlas connection
   string and database name `api_monitor`; it will auto-attach to all MongoDB nodes
   that reference that credential name.
3. Create a Google Gemini (PaLM API) credential for `02 AI Alert Writer`'s language
   model node.
4. Create a Gmail OAuth2 credential for `06 Error Handler`'s email node, and set your
   own address in **Send Error Email**.
5. Publish (activate) all six workflows. In `01`, `04`, and `05`'s settings, set
   **Error Workflow** to `06 Error Handler`.

### 3. Frontend
1. Open `frontend/index.html` and set `API_BASE` to your n8n instance's webhook base
   URL, e.g. `https://your-instance.app.n8n.cloud/webhook`.
2. Serve it statically (GitHub Pages, or just open the file directly — CORS is open
   on all three endpoints).

## Running it

Send a batch:

```bash
curl -X POST https://your-instance.app.n8n.cloud/webhook/monitor \
  -H "Content-Type: application/json" \
  -d @data/sample.json
```

List active alerts:

```bash
curl https://your-instance.app.n8n.cloud/webhook/alerts
```

Resolve an alert:

```bash
curl -X POST https://your-instance.app.n8n.cloud/webhook/resolve \
  -H "Content-Type: application/json" \
  -d '{"alertId": "alt_xxxxxxx"}'
```

A larger `data/sample-batch.json` (8 entries, covering every severity level and one
rejected row) is included for broader testing. The frontend's **Run sample batch**
button sends `data/sample.json` directly from the browser.

## Known limitations

- **No authentication** on the webhook endpoints, and CORS is open to all origins —
  acceptable for this assessment, would need an API key or signed requests in
  production.
- **Severity does not change on repeat detection.** If the same `alertKey` reappears
  with a different severity, only `occurrences`, `lastSeenAt`, and `details` are
  updated; the original severity is kept.
- **LLM calls run sequentially**, one per newly detected anomaly — a very large batch
  of first-time anomalies will take proportionally longer. Deduplication avoids this
  cost entirely for repeat issues.
- **No row-level locking.** Two simultaneous requests for the same `alertKey` could
  both see "not found" and both insert — acceptable at assessment scale, would need
  an upsert or a unique index on `alertKey` + `status` in production.
- The `/webhook/` prefix in all production URLs is fixed by n8n and cannot be removed.

## Tech stack

- **Backend / orchestration:** n8n (Node.js-based workflow engine), JavaScript Code nodes
- **Database:** MongoDB Atlas
- **AI:** Google Gemini (`gemini-3.5-flash`) for alert text generation, with a
  template-based fallback
- **Email:** Gmail API (error alerts)
- **Frontend:** single static HTML/CSS/vanilla JS file, no build step

## AI usage disclosure

This project was built with AI assistance (Claude). See `AI_Prompts.docx` for the full
record of prompts used for logic design, code generation, and debugging.

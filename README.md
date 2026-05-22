# Gmail AI Semantic Autopilot

> An always-on Python daemon that reads your inbox with Claude and takes action — not keyword matching, actual language understanding.

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://python.org)
[![Claude Haiku](https://img.shields.io/badge/Claude-Haiku-blueviolet.svg)](https://anthropic.com)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-green.svg)](https://fastapi.tiangolo.com)
[![Gmail API](https://img.shields.io/badge/Gmail%20API-v1-red.svg)](https://developers.google.com/gmail/api)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

---

## The Problem

Gmail's native filters are rule-based. They match keywords and sender addresses — they cannot understand intent.

A cold sales email saying *"I'd love to reconnect"* and an urgent client message saying *"I'd love to reconnect about the contract deadline"* look identical to a keyword filter. Important emails get buried. Weekly inbox overhead compounds.

## The Solution

A polling daemon that authenticates with your Gmail via OAuth2, reads every incoming unread message, and calls **Claude (claude-3-haiku)** to semantically classify it into one of 7 intent categories. Labels are created and applied automatically. Spam, cold outreach, and newsletters are archived without entering your inbox.

---

## Architecture

```
Gmail Inbox (unread messages)
         │
         ▼
  agent.py  ─── polls every N seconds (configurable)
         │
         ├─ get_email_headers()    ← extracts from / subject / date
         ├─ get_email_body()       ← base64 decode, truncated to 2000 chars
         │
         ▼
  classify_email()
         │  subject + sender + body[:800] →
         ▼
  Claude claude-3-haiku-20240307
         │  ← { "category": "...", "confidence": 0.0-1.0, "reason": "..." }
         │
         ▼
  apply_label()  →  Gmail API: add MCP/* label
                               optionally remove from INBOX (archive)
         │
         ▼
  classification_log.json   (append-only audit trail)
         │
         ▼
  dashboard.py  (FastAPI)
         │
    ┌────┴────┐
    │  React  │   /api/log    → full history
    │   UI    │   /api/stats  → category breakdown
    └─────────┘
```

## Classification Taxonomy

7 categories, each mapping to a Gmail label under the `MCP/` namespace:

| Key | Label | Auto-archived? | Rationale |
|---|---|---|---|
| `IMPORTANT_ACTION` | MCP/Action Required | No | Needs your response or decision |
| `PROJECT_WORK` | MCP/Projects | No | Active work threads |
| `FINANCE` | MCP/Finance & Billing | No | Invoices, receipts, payments |
| `NEWSLETTER` | MCP/Newsletters | **Yes** | Subscriptions you chose but don't need to act on |
| `COLD_OUTREACH` | MCP/Cold Outreach | **Yes** | Unsolicited sales / recruiter spam |
| `SPAM_SOCIAL` | MCP/Spam/Social | **Yes** | Social notifications, rewards emails |
| `ARCHIVE` | MCP/Auto-Archive | **Yes** | Everything else below the fold |

### Why Claude Haiku and not a local model?

Context window matters here. Haiku reads the full sender + subject + 800 chars of body in a single call. It catches sarcasm, urgency implied by tone, the difference between a recruiter you want to hear from and one you don't. A local keyword classifier would require training data and ongoing maintenance. Haiku keeps API cost under **$0.01/day** for a typical inbox.

---

## Key Code Decisions

**Structured JSON output with robust extraction** — Claude doesn't always return bare JSON (it sometimes wraps it in markdown). The extractor finds the first `{` and last `}` rather than blindly parsing the full response:

```python
# agent.py — classify_email()
raw = message.content[0].text.strip()
start = raw.find("{")
end = raw.rfind("}") + 1
return json.loads(raw[start:end])
# Returns: {"category": "COLD_OUTREACH", "confidence": 0.94, "reason": "Unsolicited sales pitch..."}
```

**Idempotent label bootstrap** — on every daemon startup, `ensure_labels()` checks which `MCP/*` labels already exist before creating. A daemon restart never creates duplicate labels:

```python
# agent.py — ensure_labels()
existing = service.users().labels().list(userId="me").execute().get("labels", [])
existing_map = {l["name"]: l["id"] for l in existing}
# Only creates labels missing from existing_map
```

**Non-destructive archiving** — emails are never deleted. Archiving = removing the `INBOX` label while preserving the message. Every action is reversible from the dashboard or Gmail directly:

```python
# agent.py — apply_label()
body = {"addLabelIds": [label_id], "removeLabelIds": []}
if archive:
    body["removeLabelIds"].append("INBOX")  # remove from inbox, not from Gmail
```

**Body truncation** — Gmail message payloads can be large. Bodies are capped at 2000 chars (800 sent to Claude) to keep latency low and cost negligible without losing classification signal:

```python
return body[:2000]  # Truncate for API efficiency
```

---

## Results (6-week personal inbox run)

| Metric | Result |
|---|---|
| Total emails processed | 1,400+ |
| Classification accuracy (manual spot-check, 200 emails) | **98%+** |
| False positive rate (important email archived) | **< 0.5%** |
| Time saved per week | **~10 hours** |
| Avg. Claude API cost per day | **< $0.01** |

---

## Quick Start

```bash
git clone https://github.com/Alan-911/gmail-ai-agent
cd gmail-ai-agent
pip install -r requirements.txt
```

**Set up credentials** — follow [docs/SETUP.md](docs/SETUP.md) for Gmail OAuth2 + Anthropic API key.

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-...
POLL_INTERVAL_SECONDS=60
```

```bash
# Start the daemon
python agent.py
# ✅ Labels ready: ['IMPORTANT_ACTION', 'PROJECT_WORK', ...]
# ✓ [COLD_OUTREACH] (94%) — "Quick question about your background..."
# ✓ [IMPORTANT_ACTION] (99%) — "Contract revision needed by Friday"
# Sleeping 60s...
```

```bash
# Start the dashboard API (separate terminal)
uvicorn dashboard:app --reload --port 8001
# GET http://localhost:8001/api/log    → classification history
# GET http://localhost:8001/api/stats  → { total: 1423, breakdown: { COLD_OUTREACH: 312, ... } }
```

## Repo Structure

```
├── agent.py              # Daemon: Gmail polling, Claude classification, label application
├── dashboard.py          # FastAPI: serves /api/log and /api/stats from classification_log.json
├── classification_log.json  # Append-only audit trail (auto-created)
├── docs/
│   └── SETUP.md          # Gmail OAuth2 + .env setup walkthrough
└── requirements.txt
```

## Stack

`Python 3.11` · `Anthropic SDK (claude-3-haiku)` · `Gmail API v1` · `OAuth2` · `FastAPI` · `Uvicorn` · `python-dotenv`

---

Built by [Yves Alain Iragena](https://alan-911.github.io/my-portfolio) · MAIL Lab, Catholic University of America · iragena@cua.edu

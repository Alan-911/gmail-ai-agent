# Gmail AI Semantic Autopilot

> Autonomous email management daemon powered by Claude via MCP — semantic understanding beyond keyword filters.

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://python.org)
[![Claude API](https://img.shields.io/badge/Claude-MCP-blueviolet.svg)](https://anthropic.com)
[![Accuracy](https://img.shields.io/badge/Accuracy-98%25+-brightgreen.svg)]()
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

## The Problem

Gmail's native filters are rule-based — they can't understand context. "Can we reschedule?" looks identical to "Reschedule confirmed" to a keyword filter. Important emails get buried; cold outreach clutters the inbox.

## Solution

An always-on Python daemon that uses **Claude via the Model Context Protocol (MCP)** to read emails with genuine language understanding — labeling, archiving, and filtering based on meaning, not pattern matching.

## Features

- **Semantic labeling** — Claude classifies by intent, not keywords
- **Preference learning** — rules update based on your corrections over time
- **Cold outreach filtering** — detects and archives unsolicited sales/recruiter spam
- **Zero-touch archiving** — stateful rules handle recurring senders automatically
- **React audit dashboard** — see every action taken, override any decision, review audit trail
- **Non-destructive** — never deletes; all actions are reversible from the dashboard

## Results (personal inbox, 6-week run)

| Metric | Result |
|--------|--------|
| Classification accuracy | 98%+ |
| Time saved per week | ~10 hours |
| False positive rate (important marked as spam) | <0.5% |

## Architecture

```
Gmail API (OAuth2)
      │
      ▼
Python Daemon (agent.py)
      │
      ├── MCP Client ──► Claude (semantic reasoning)
      │
      ├── Rules Engine (preference store)
      │
      └── FastAPI ──► React Dashboard (dashboard.py)
```

## Quick Start

```bash
git clone https://github.com/Alan-911/gmail-ai-agent
cd gmail-ai-agent

# Backend
pip install -r requirements.txt
# Add ANTHROPIC_API_KEY and Gmail OAuth credentials to .env (see docs/SETUP.md)
python agent.py

# Dashboard (separate terminal)
cd dashboard && npm install && npm run dev
# Open http://localhost:3000
```

## Setup

Full OAuth2 setup guide: [docs/SETUP.md](docs/SETUP.md)

Required credentials:
- Anthropic API key (for Claude)
- Google OAuth2 client credentials (Gmail API)

## Repo Structure

```
├── agent.py          # Main daemon — fetches, classifies, acts
├── dashboard.py      # FastAPI backend for the React UI
├── docs/
│   └── SETUP.md      # OAuth and environment setup guide
└── requirements.txt
```

## Stack

`Python` `Claude API` `MCP` `FastAPI` `Gmail API` `OAuth2` `React` `TypeScript`

---

Built by [Yves Alain Iragena](https://alan-911.github.io/my-portfolio)

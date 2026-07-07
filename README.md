# 📬 AI-Powered Job Application Tracker

An automated workflow built with **n8n** that monitors your Gmail inbox, uses AI to extract and classify job application emails, and logs structured data into a **PostgreSQL** database — all running locally on your machine for free.

---

## 📌 Overview

Manually tracking job applications is tedious and easy to lose track of. This project automates the entire process by:

1. Monitoring your Gmail inbox for job-related emails
2. Filtering out job alerts and promotional emails
3. Using an AI model to extract key details (company, role, status, date)
4. Storing structured records in a PostgreSQL database with upsert logic to avoid duplicates

---

## 🔧 Tech Stack

| Tool | Purpose |
|---|---|
| [n8n](https://n8n.io) | Workflow automation (self-hosted via npm) |
| Gmail API | Email fetching via OAuth2 |
| Google Gemini API (Free Tier) | Primary AI model for email extraction |
| Ollama + LLaMA 3.2 | Fallback local AI model |
| PostgreSQL | Structured data storage |
| ngrok | Secure tunnel for OAuth callback (local dev) |

---

## 🗺️ Workflow Architecture

```
Gmail Trigger (every minute)
        ↓
Get Many Messages (keyword filtered)
        ↓
Edit Fields (normalize: Subject, From, snippet, internalDate)
        ↓
Loop Over Items (process one email at a time)
        ↓
Code Node (extract headers, parse date → dateStamp)
        ↓
AI Agent (Gemini primary / Ollama fallback)
        ↓
Code Node (parse AI JSON output, generate unique ID)
        ↓
Edit Fields (map fields for downstream nodes)
        ↓
IF Node (isJobRelated = true?)
    ↓ true                  ↓ false
Edit Fields1           Loop Over Items
    ↓                  (next email)
PostgreSQL Upsert
    ↓
Loop Over Items
(next email)
```

---

## 📂 Workflow Nodes

### 1. Gmail Trigger
Polls Gmail every minute for new emails. Acts as the entry point that kicks off the workflow whenever a new email arrives.

### 2. Get Many Messages
Fetches emails matching job-related keywords using Gmail search syntax:
```
("application submitted" OR "submitted successfully" OR interview OR unfortunately 
OR "we regret" OR "thank you for applying" OR "next steps" OR offer OR hired 
OR "not moving forward" OR "other candidates")
```
This pre-filters noise at the Gmail level before any AI processing occurs.

### 3. Edit Fields 2 (Normalize)
Extracts and normalizes four key fields from the raw Gmail output:
- `From` — sender address
- `internalDate` — raw timestamp in milliseconds
- `snippet` — email body preview
- `Subject` — email subject line

### 4. Loop Over Items
Processes each email one at a time in a loop, feeding them individually through the AI pipeline to avoid rate limits and timeouts.

### 5. Code in JavaScript (Header Parser)
Parses raw Gmail API headers to reliably extract `Subject` and `From` fields, and converts `internalDate` (Unix ms timestamp) to an ISO date string (`YYYY-MM-DD`).

```javascript
const internalDateMs = msg.internalDate;
const dateStamp = internalDateMs
  ? new Date(Number(internalDateMs)).toISOString().split('T')[0]
  : null;
```

### 6. AI Agent (Basic LLM Chain)
Sends email content to the AI model with a structured prompt that extracts:

| Field | Description |
|---|---|
| `isJobRelated` | Boolean — is this a real job application email? |
| `company` | Actual hiring company (not the ATS platform) |
| `role` | Job title, cleaned of requisition numbers |
| `status` | Applied / Interviewing / Offer / Rejected / Ghosted / N/A |
| `dateReceived` | YYYY-MM-DD |
| `notes` | One-sentence factual summary |
| `jobPoster` | LinkedIn / Indeed / Jobstreet / Workday / Company Website / Other |

**Primary model:** Google Gemini 2.5 Flash (free tier)
**Fallback model:** Ollama LLaMA 3.2 (local)

### 7. Code in JavaScript 1 (JSON Parser)
Parses the AI's raw text output into a structured JSON object. Handles edge cases like markdown code fences, missing fields, and malformed JSON. Generates a unique `id` field by combining company + role:

```javascript
id: `${parsed.company}_${parsed.role}`.toLowerCase().replace(/\s+/g, "_")
```

### 8. IF Node
Routes emails based on `isJobRelated`:
- **True** → proceed to database storage
- **False** → skip back to loop for next email

### 9. Execute a SQL Query (PostgreSQL Upsert)
Inserts new job application records or updates existing ones using `ON CONFLICT`:

```sql
INSERT INTO job_applications (id, company, role, status, date_received, notes, job_poster, updated_at)
VALUES (...)
ON CONFLICT (id)
DO UPDATE SET
  status = EXCLUDED.status,
  notes = EXCLUDED.notes,
  updated_at = CURRENT_TIMESTAMP;
```

---

## 🗄️ Database Schema

```sql
CREATE TABLE job_applications (
  id VARCHAR(255) PRIMARY KEY,
  company VARCHAR(255),
  role VARCHAR(255),
  status VARCHAR(50),
  date_received DATE,
  notes TEXT,
  job_poster VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## ⚙️ Setup Guide

### Prerequisites
- Windows 10/11
- Node.js (v18+)
- n8n installed globally (`npm install -g n8n`)
- PostgreSQL installed
- Ollama installed (optional fallback)
- ngrok account with a static domain
- Google Cloud account

### 1. Clone the Repository
```bash
git clone https://github.com/vnznthny/job-application-tracker.git
cd job-application-tracker
```

### 2. Set Up PostgreSQL
Open pgAdmin and run:
```sql
CREATE DATABASE job_tracker;

\c job_tracker

CREATE TABLE job_applications (
  id VARCHAR(255) PRIMARY KEY,
  company VARCHAR(255),
  role VARCHAR(255),
  status VARCHAR(50),
  date_received DATE,
  notes TEXT,
  job_poster VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3. Set Up ngrok (for Google OAuth)
```powershell
ngrok config add-authtoken YOUR_NGROK_TOKEN
ngrok http --domain=your-static-domain.ngrok-free.app 5678
```

### 4. Start n8n
```powershell
$env:N8N_EDITOR_BASE_URL="https://your-static-domain.ngrok-free.app"
$env:WEBHOOK_URL="https://your-static-domain.ngrok-free.app"
n8n start
```

### 5. Configure Google Cloud
1. Create a project in [Google Cloud Console](https://console.cloud.google.com)
2. Enable **Gmail API**
3. Create OAuth 2.0 credentials (Web Application)
4. Add authorized redirect URI:
```
https://your-static-domain.ngrok-free.app/rest/oauth2-credential/callback
```

### 6. Configure Credentials in n8n
Go to **Settings → Credentials** and add:
- **Gmail OAuth2** — using your Google Cloud client ID + secret
- **Google Gemini API** — from [Google AI Studio](https://aistudio.google.com)
- **PostgreSQL** — host: `localhost`, port: `5432`, database: `job_tracker`
- **Ollama** (optional) — `http://localhost:11434`

### 7. Import the Workflow
1. Open n8n at `http://localhost:5678`
2. Go to **Workflows → Import**
3. Upload `Job_Tracker_copy.json`
4. Reconnect all credentials
5. Activate the workflow

---

## 🚀 Daily Startup Routine

Save this as `start-n8n.bat` on your Desktop:

```batch
@echo off
start powershell -NoExit -Command "ngrok http --domain=your-static-domain.ngrok-free.app 5678"
timeout /t 3
start powershell -NoExit -Command "$env:N8N_EDITOR_BASE_URL='https://your-static-domain.ngrok-free.app'; $env:WEBHOOK_URL='https://your-static-domain.ngrok-free.app'; n8n start"
```

---

## 📊 Example Output

| ID | Company | Role | Status | Date Received | Notes | Job Poster |
|---|---|---|---|---|---|---|
| asticom_technology_inc_ece | Asticom Technology Inc | Electronics & Communication Engineer | Rejected | 2026-07-05 | Application rejected after initial review | Jobstreet |
| analog_devices_embedded_sw | Analog Devices | Associate Embedded Software Engineer | Rejected | 2026-06-19 | Regret email received after application review | Workday |

---

## ⚠️ Limitations

- **Requires PC to be running** — n8n is self-hosted via npm; the workflow won't trigger if the machine is off or n8n is closed
- **Gemini free tier rate limits** — capped at 5-15 requests/minute depending on model; a Wait node (15s delay) is included in the loop to stay within limits
- **Gmail snippet only** — the workflow uses Gmail's snippet (first ~200 characters) rather than the full email body, which may miss context in longer emails
- **English emails only** — the AI prompt is optimized for English-language job emails

---

## 🔮 Possible Improvements

- [ ] Deploy n8n to a cloud server (Oracle Free Tier, Railway) for 24/7 automation
- [ ] Add a Slack or email notification when a new application is logged
- [ ] Build a dashboard using Grafana or Metabase connected to PostgreSQL
- [ ] Add interview date tracking and calendar reminders
- [ ] Support for multiple Gmail accounts

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

## 🙏 Acknowledgements

- [n8n](https://n8n.io) — open source workflow automation
- [Google Gemini](https://ai.google.dev) — AI model API
- [Ollama](https://ollama.com) — local LLM runtime
- [PostgreSQL](https://www.postgresql.org) — open source database

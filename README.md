# Recruiter-Outreach — Weekend MVP

Automate the boring part of job-hunting: find the recruiter, grab their profile PDF, and draft a personalised e-mail—all from a single web UI.

## 🎯 Problem & Solution

**Problem**: Job seekers face an "ATS black hole" when applying through portals, with manual outreach being time-consuming and tedious.

**Solution**: Our tool finds recruiters behind job listings, captures their profiles, and generates personalized outreach messages in under 60 seconds (vs. 10-15 minutes manually).

## 🖥️ Key Features

- **Job Search**: Find relevant positions through LinkedIn's job search
- **Recruiter Discovery**: Automatically identify and capture recruiter profiles as PDFs
- **Email Finding**: Discover recruiter emails through Proxycurl API or intelligent guessing
- **Smart Drafting**: Generate personalized outreach based on job and recruiter context
- **Tracking**: Monitor open rates and responses to improve your approach

## 🛠️ Tech Stack

| Layer | Choice | Why |
| ----- | ------ | --- |
| Front-end | Next.js (React + TS) | Rapid UI development, Vercel preview |
| Back-end | FastAPI (Python 3.12) | Async-first, great documentation |
| Agent Orchestrator | LangGraph (Python) | Graph-style workflow for AI agents |
| LLMs | **Azure OpenAI `o4-mini`** (reasoning) · `gpt-4o` (general) · **Code Llama-14B-GGUF** (offline) | Free credits + offline fallback |
| Browser Automation | Playwright MCP | Headful browsing, deterministic operation |
| E-mail | Resend React-Email | Free dev tier available |
| Auth + DB + Storage | Supabase | Postgres, Storage + Row-level Security |

## ⚡ Local Setup

```bash
# 1. Install JS dependencies
npm install

# 2. Install Python dependencies (using uv for speed)
uv pip install -r requirements.txt

# 3. Run Playwright worker (requires LinkedIn cookies in .env)
python backend/worker.py

# 4. Start dev servers (concurrently)
uvicorn backend.main:app --reload & npm --workspace frontend run dev
```

## 📊 Success Metrics (MVP)

| KPI | Target |
| --- | ------ |
| Recruiter profile PDF captured | ≥ 95% of attempts |
| Outreach draft time | ≤ 60 seconds |
| Email open rate (Resend tracking) | ≥ 30% |
| Positive reply rate | ≥ 15% |

## 📁 Documentation

See the `/Docs` folder for detailed documentation:

- [Product Requirements Document](Docs/Product-Requirements-Document.md)
- [Architecture](Docs/Architecture.md)
- [Security Considerations](Docs/Security.md)
- [Module Documentation](Docs/Modules.md)
- [Prompt Library](Docs/Prompts.md)
- [Folder Structure](Docs/Folder-structure.md)
- [Deployment Guide](Docs/Deployments.md)
- [Database Schema](Docs/Database_Schema.md)

## 📷 Demo

*One-Minute Demo (gif placeholder)*

## 📅 10-Hour Development Plan

| Hour | Deliverable |
| ---- | ----------- |
| 0-1 | Repo init, env vars, Supabase SQL migration |
| 1-3 | Playwright login + job search proof-of-concept |
| 3-4 | Recruiter PDF capture + Supabase upload |
| 4-6 | LangGraph flow + FastAPI endpoints |
| 6-7 | Next.js Search & Lead List pages |
| 7-8 | PersonaliseAgent w/ Azure o4-mini prompt |
| 8-9 | Resend integration + send success callback |
| 9-10 | Polish UI, write README gifs, smoke tests |

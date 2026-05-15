# OutboundAI — AI Outbound Voice Calling Platform

Production-grade SaaS platform for automated outbound voice calls powered by Google Gemini Live, LiveKit Agents, and Vobiz SIP telephony.

## Features

- **Gemini Live Realtime Voice** — Sub-100ms latency AI conversations with native audio
- **SIP Outbound Dialing** — Dial-first pattern via Vobiz SIP trunk
- **Campaign Engine** — Schedule mass call campaigns (once / daily / weekdays)
- **Appointment Booking** — AI books appointments and syncs with Cal.com
- **Contact Memory** — AI remembers past interactions with each lead
- **Call Transfer** — SIP REFER to human agents when needed
- **S3 Recording** — Optional call recording to any S3-compatible storage
- **SMS Confirmation** — Twilio-powered booking confirmations
- **Agent Profiles** — Multiple AI personalities with different voices/models/prompts
- **Dashboard** — Full-featured web UI with analytics, CRM, logs, and settings

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Dashboard (ui/index.html)                      │
│  Vanilla HTML/CSS/JS — served by FastAPI        │
└────────────────────┬────────────────────────────┘
                     │ REST API
┌────────────────────▼────────────────────────────┐
│  server.py — FastAPI + APScheduler              │
│  Campaign scheduling, call dispatch, settings   │
└────────────────────┬────────────────────────────┘
                     │ LiveKit API
┌────────────────────▼────────────────────────────┐
│  agent.py — LiveKit Agent Worker                │
│  Gemini Live session, SIP dialing, tools        │
└────────────────────┬────────────────────────────┘
                     │
    ┌────────────────┼────────────────┐
    ▼                ▼                ▼
 Supabase       Vobiz SIP        Google Gemini
 (Postgres)     (Telephony)      (Live Audio AI)
```

## Tech Stack

| Component | Technology |
|-----------|-----------|
| AI Voice | Google Gemini Live (3.1/2.5 native audio) |
| Agent Framework | LiveKit Agents 1.x |
| Telephony | Vobiz SIP via LiveKit SIP |
| Backend | FastAPI + Uvicorn |
| Database | Supabase (PostgreSQL) |
| Scheduling | APScheduler |
| Calendar | Cal.com API |
| SMS | Twilio |
| Recording | S3-compatible storage |
| Dashboard | Vanilla HTML/CSS/JS + Chart.js |

## Files

| File | Purpose |
|------|---------|
| `agent.py` | LiveKit agent worker — Gemini Live session, SIP dialing |
| `server.py` | FastAPI backend — REST API, campaign engine |
| `db.py` | Supabase async operations |
| `tools.py` | 9 LLM function tools (booking, transfer, SMS, etc.) |
| `prompts.py` | System prompt template |
| `ui/index.html` | Single-file dashboard (12 tabs) |
| `supabase_schema.sql` | Database schema |
| `Dockerfile` | Production Docker image |
| `start.sh` | Entrypoint script |
| `requirements.txt` | Python dependencies |

## Deployment

### Environment Variables (Single Source of Truth)

Set these on your VPS (Coolify, Docker Compose, or systemd). The app reads directly from VPS env vars — no `.env` file needed in production.

**Required:**
```
LIVEKIT_URL=wss://your-livekit.example.com
LIVEKIT_API_KEY=APIxxxxxxxx
LIVEKIT_API_SECRET=xxxxxxxx
GOOGLE_API_KEY=AIza...
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_KEY=eyJ...
VOBIZ_SIP_DOMAIN=sip.vobiz.com
VOBIZ_USERNAME=your_user
VOBIZ_PASSWORD=your_pass
VOBIZ_OUTBOUND_NUMBER=+91XXXXXXXXXX
OUTBOUND_TRUNK_ID=ST_xxx (created via dashboard)
```

**Optional:**
```
GEMINI_MODEL=gemini-3.1-flash-live-preview
GEMINI_TTS_VOICE=Aoede
DEFAULT_TRANSFER_NUMBER=+91XXXXXXXXXX
TWILIO_ACCOUNT_SID=ACxxx
TWILIO_AUTH_TOKEN=xxx
TWILIO_FROM_NUMBER=+1xxx
CALCOM_API_KEY=cal_live_xxx
CALCOM_EVENT_TYPE_ID=12345
S3_ACCESS_KEY_ID=xxx
S3_SECRET_ACCESS_KEY=xxx
S3_BUCKET=recordings
S3_ENDPOINT_URL=https://s3.example.com
DEEPGRAM_API_KEY=xxx (for STT fallback)
```

### Steps

1. **Database** — Run `supabase_schema.sql` in Supabase Dashboard → SQL Editor

2. **Deploy via Docker:**
   ```bash
   docker build -t outbound-ai .
   docker run -d --name outbound-ai \
     -p 8000:8000 \
     -e LIVEKIT_URL=wss://... \
     -e LIVEKIT_API_KEY=... \
     -e LIVEKIT_API_SECRET=... \
     -e GOOGLE_API_KEY=... \
     -e SUPABASE_URL=... \
     -e SUPABASE_SERVICE_KEY=... \
     -e VOBIZ_SIP_DOMAIN=... \
     -e VOBIZ_USERNAME=... \
     -e VOBIZ_PASSWORD=... \
     -e VOBIZ_OUTBOUND_NUMBER=... \
     outbound-ai
   ```

3. **First-time setup via Dashboard:**
   - Open `http://your-vps:8000`
   - Go to Settings → create SIP trunk (auto-generates `OUTBOUND_TRUNK_ID`)
   - Test a single call from the Call tab

## Env Var Priority

```
VPS Environment Variables  →  Supabase settings table  →  Hardcoded defaults
    (WINS)                      (fallback only)             (last resort)
```

The dashboard Settings tab saves to Supabase as fallback. VPS env vars are **never** overwritten.

## Key Architecture Rules

- **Dial-first pattern** — SIP call is placed BEFORE AI session starts
- **Never use `close_on_disconnect=True`** — causes drops on audio blips
- **Never call `generate_reply()` for Gemini 3.1/2.5** — native audio models speak autonomously
- **VAD sensitivity** — Uses `END_SENSITIVITY_LOW` with 2s silence threshold
- **Session resumption** — Transparent reconnection prevents context timeouts
- **Context compression** — Sliding window at 25,600 tokens prevents freezes

## License

Private — All rights reserved.

# HeatShift: Architecture

## 1. Components

```
                       ┌───────────────────────────────┐
                       │ Supervisor web app (Amplify)  │
                       │ React + Vite + Tailwind       │
                       └──────────────┬────────────────┘
                                      │ HTTPS
                       ┌──────────────▼────────────────┐
  Telegram ───webhook──►   Amazon API Gateway (HTTP)   │
  (supervisor +        └──────────────┬────────────────┘
   workers)                           │
                       ┌──────────────▼────────────────┐
  EventBridge          │        AWS Lambda (Python)    │
  Scheduler 05:30 ────►│  api · planner · telegram     │
                       └───┬──────┬──────┬──────┬──────┘
                           │      │      │      │
              ┌────────────▼┐ ┌───▼────┐ │  ┌───▼──────────────┐
              │ engine/     │ │DynamoDB│ │  │ Strands agent    │
              │ forecast    │ │ table  │ │  │ on Amazon Bedrock│
              │ wbgt, rules │ └────────┘ │  └───┬──────────────┘
              │ planner     │            │      │ tools
              └─────┬───────┘   ┌────────▼──┐   ├─► Amazon Location Service
                    │           │ Polly → S3│   │   (clinics, water, routes)
              Open-Meteo API    └───────────┘   └─► engine functions
```

## 2. Request flows

### A. Morning plan (automatic)
1. EventBridge Scheduler triggers `planner` Lambda at 05:30 IST for each active site.
2. `engine.forecast.get_hourly(lat, lon, date)` → hourly temperature, RH, wind, shortwave radiation.
3. `engine.wbgt.compute(hourly)` → WBGT per hour.
4. `engine.planner.build_plan(wbgt, intensity, acclimatised, shift)` → per-hour work/rest minutes + risk level (deterministic).
5. Strands agent receives the plan and produces: supervisor summary, crew message (per language), reasons.
6. Plan saved as `DRAFT` in DynamoDB; supervisor notified (Telegram message with **Approve** button + web link).

### B. Approval and broadcast
1. Supervisor taps **Approve** (Telegram callback or web app) → `POST /plans/{siteId}/{date}/approve`.
2. Lambda sets status `APPROVED`, calls Polly per language → MP3 in S3 (pre-signed URL).
3. Sends voice note + text to every worker chat in that site. Status → `SENT`, delivery count stored.

### C. Worker check-in and escalation
1. Worker taps **Unwell** (or replies) → symptom checklist buttons.
2. `engine.triage.assess(symptoms)` (rule-based):
   - **Red flags** (confusion, fainting, no sweating with hot skin, seizure, vomiting repeatedly) → `EMERGENCY`: tell worker/supervisor to call **108**, send nearest hospital + route.
   - Otherwise → `REST`: shade, water/ORS, 30-min rest, re-check prompt.
3. Agent tool `find_nearby_help` uses Location Service Places (hospital / clinic) + Routes (drive time).
4. Supervisor alerted; event logged.

## 3. Data model (single DynamoDB table `heatshift`)

| Entity | PK | SK | Key attributes |
|---|---|---|---|
| Site | `SITE#<siteId>` | `META` | name, lat, lon, crewSize, intensity, shiftStart, shiftEnd, languages[], supervisorChatId, active |
| Worker | `SITE#<siteId>` | `WORKER#<workerId>` | name, chatId, language, joinedOn (acclimatisation), active |
| Plan | `SITE#<siteId>` | `PLAN#<yyyy-mm-dd>` | hourly[], status (DRAFT/APPROVED/SENT), summary, messages{lang}, audioKeys{lang}, unsafeHoursAvoided, approvedAt, approvedBy, delivered |
| Check-in | `SITE#<siteId>` | `CHECKIN#<yyyy-mm-dd>#<ts>#<workerId>` | status, symptoms[], triage, escalatedTo, nearestFacility |

GSI1 (optional): `GSI1PK = ACTIVE`, `GSI1SK = SITE#<siteId>` for the scheduler to list active sites.

`hourly[]` item: `{hour, tempC, rh, windMs, solarWm2, wbgtC, risk, workMin, restMin, note}`

## 4. API (HTTP API, JSON)

| Method | Path | Purpose |
|---|---|---|
| POST | `/sites` | Create site |
| GET | `/sites/{siteId}` | Site details |
| POST | `/sites/{siteId}/workers` | Add worker(s) |
| POST | `/plans/generate` | `{siteId, date, replay?: bool}` → build draft plan |
| GET | `/plans/{siteId}/{date}` | Get plan |
| POST | `/plans/{siteId}/{date}/approve` | Approve + broadcast |
| POST | `/plans/{siteId}/{date}/shift` | `{startTime}` → recompute |
| POST | `/checkins` | Worker check-in |
| GET | `/metrics/{siteId}?days=7` | Metric card + history |
| POST | `/telegram/webhook` | Telegram updates (messages, button callbacks) |

## 5. Agent design (Strands)

**System role:** heat-safety planning assistant for Indian outdoor work sites. Never changes safety limits; explains them.

| Tool | Type | Does |
|---|---|---|
| `get_forecast(lat, lon, date)` | deterministic | Hourly weather (live or archive for replay) |
| `compute_wbgt(hourly)` | deterministic | WBGT per hour |
| `plan_shift(wbgt, intensity, acclimatised, shift)` | deterministic | Work/rest minutes per hour + risk |
| `find_nearby_help(lat, lon, kind)` | AWS Location | Nearest hospital/clinic/water + drive time |
| `draft_messages(plan, languages)` | LLM | Short crew messages + supervisor summary |
| `triage(symptoms)` | deterministic | REST vs EMERGENCY |

Guardrails: the agent's output must quote the plan's numbers exactly; a validator checks every time and minute value in the message against the plan before sending.

## 6. Security and cost

- Telegram webhook secret token verified on every request.
- No personal data beyond first name + chat ID; phone numbers optional.
- IAM: least-privilege roles per Lambda (Bedrock invoke, Polly synthesize, DynamoDB table, S3 bucket, Location index).
- Cost: well inside Free Tier/credits (Lambda, DynamoDB on-demand, a few Polly/Bedrock calls per day).

## 7. Local dev

- `engine/` runs with plain Python (no AWS) + pytest.
- `sam local start-api` for the backend; LocalStack optional for DynamoDB/S3.
- `web/` with `npm run dev`, `.env` pointing at local or deployed API.

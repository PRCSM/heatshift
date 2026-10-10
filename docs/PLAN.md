# HeatShift: Build Plan

Event window: **Thu 8 Oct – Sun 11 Oct 2026**. Submission: Sunday (exact hour on the event schedule page). We aim to **submit by 5 PM Sunday**.

## 1. Goal for the weekend

Ship **one complete, working loop**, end to end, on AWS:

> Forecast → WBGT → rule-based work–rest plan → AI explanation → supervisor approves → workers receive a voice note → a worker reports "unwell" → supervisor is alerted with the nearest clinic → metric shows unsafe worker-hours avoided.

Anything outside this loop is a stretch goal.

## 2. Scope

### Must have (MVP)
- [ ] Site setup: name, location (lat/lon), crew size, work intensity (light / moderate / heavy), shift hours, language.
- [ ] Hourly forecast for the site (Open-Meteo) + **replay mode** for a past date (e.g. Delhi, 29 May 2024).
- [ ] WBGT per hour (documented formula, unit-tested).
- [ ] Deterministic work–rest plan per hour (ACGIH/ISO table encoded as data).
- [ ] Strands agent on Bedrock: explains the plan in plain language + drafts crew message (Hindi / English / Tamil text).
- [ ] Supervisor web app: today's plan as a timeline, **Approve** button, status.
- [ ] Broadcast on approval: Polly voice note (Hindi/English) + text to the crew via Telegram bot.
- [ ] Worker check-in ("OK" / "Unwell" + symptoms) → red-flag triage → supervisor alert with nearest clinic (Amazon Location Service).
- [ ] Metric card: unsafe worker-hours avoided, plan approved time, messages delivered.
- [ ] Daily 05:30 run via EventBridge Scheduler.
- [ ] Deployed: API on AWS (SAM), web app on Amplify, public URL.

### Should have
- [ ] Supervisor can shift the start time (e.g. 6:00 → 5:30) and the plan recomputes.
- [ ] New-worker flag: lower limits for workers in their first week (not yet acclimatised).
- [ ] Plan history page (last 7 days) as the compliance log.

### Stretch
- [ ] WhatsApp delivery (Twilio sandbox; production path: AWS End User Messaging Social).
- [ ] Liljegren physics-based WBGT instead of the simplified formula.
- [ ] Multi-site dashboard for a contractor with several sites.
- [ ] IMD heat warning overlay.

### Out of scope
- Wearables / body-temperature sensors.
- Medical diagnosis of any kind.

## 3. Timeline and phases

Each phase ends with a **`HANDOFF_PHASE_N.md`** in `docs/handoffs/` listing every file changed, function signatures, env vars and open issues, so the next phase starts without re-reading the codebase.

| Phase | When | Output | Done when |
|---|---|---|---|
| **P0 Setup** | Sat 10:30–11:30 | Repo, AWS account, Bedrock model access (ap-south-1), Telegram bot token, SAM project skeleton | `sam build` passes; Bedrock test call returns text |
| **P1 Core engine** | Sat 11:30–15:00 | `engine/`: forecast, WBGT, plan_shift, replay, CLI, tests | `python -m engine.cli --lat 28.61 --lon 77.21 --date 2024-05-29` prints a plan; tests green |
| **P2 Backend + agent** | Sat 15:00–20:00 | SAM stack: API, Lambdas, DynamoDB, Strands agent, Polly → S3, Location Service | `POST /plans/generate` returns plan + explanation + audio URL from AWS |
| **P3 App + messaging** | Sat 20:00–01:00 | Supervisor web app on Amplify; Telegram approve/broadcast/check-in; EventBridge schedule | Full loop works on a phone, deployed URL |
| **P4 Polish** | Sun 08:00–12:00 | Seed demo site, metric card, error states, README architecture image | A stranger can use it without help |
| **P5 Ship** | Sun 12:00–17:00 | 3-min video, writeup, Builder Center blog, submission | Form submitted; YouTube link opens signed-out |

Exact Claude Code prompts for each phase: [`PHASE_PROMPTS.md`](PHASE_PROMPTS.md).

## 4. Team roles (adjust to team size)

| Role | Owns |
|---|---|
| **Backend / agent** | SAM stack, Lambdas, DynamoDB, Strands agent, Bedrock, Polly, Location Service |
| **Frontend** | Supervisor web app (React + Vite + Tailwind), Amplify deploy |
| **Rules + data + QA** | WBGT + work–rest tables, tests, replay data, triage rules, end-to-end testing |
| **Story** | Field interviews (2–3 site workers/supervisors, with consent), video, writeup, blog |

Solo / small team: one person takes Backend + Rules, another Frontend + Story.

## 5. Repository layout (target)

```
heatshift/
├── engine/                 # pure Python, no AWS: forecast, wbgt, rules, planner, triage
│   ├── forecast.py
│   ├── wbgt.py
│   ├── rules.py            # work–rest tables as data
│   ├── planner.py
│   ├── triage.py
│   ├── metrics.py
│   └── cli.py
├── agent/                  # Strands agent + tool definitions
│   ├── heatshift_agent.py
│   └── tools.py
├── backend/                # AWS SAM app
│   ├── template.yaml
│   └── functions/
│       ├── api/            # REST handlers
│       ├── scheduler/      # 05:30 daily planner
│       └── telegram/       # webhook: approve, check-in
├── web/                    # supervisor app (React + Vite + Tailwind)
├── tests/
├── docs/
│   └── handoffs/
└── README.md
```

## 6. Key decisions

| Decision | Choice | Why |
|---|---|---|
| Safety logic | Deterministic rules, not LLM | Explainable, testable, defensible to judges |
| WBGT (MVP) | Simplified formula from temperature + humidity, documented | Works with free forecast data; Liljegren as stretch |
| Agent framework | Strands Agents SDK | AWS open source (prize eligibility) + clean tool use |
| Region | ap-south-1 (Mumbai) | Low latency for India; check Bedrock model availability, use a cross-region inference profile if needed |
| Messaging (MVP) | Telegram bot | Free, instant setup, supports voice notes and buttons |
| Infra as code | AWS SAM | AWS open-source CLI, fast Lambda + API deploys |
| Frontend | React + Vite + Tailwind on Amplify | Fast to build, one-command hosting |

## 7. Success metrics (shown in the demo)

- **Unsafe worker-hours avoided** per day = Σ over hours where baseline work exceeds the safe limit of (baseline work minutes − planned work minutes) × crew size ÷ 60.
- **Time to plan approval** (target: before shift start).
- **Messages delivered** / crew size.
- **Unwell reports escalated** and time to supervisor alert.

## 8. Risks and mitigations

| Risk | Mitigation |
|---|---|
| October is off-season for heat | Demo uses **replay of a real heatwave day**; pitch as "ready before April 2027" |
| Bedrock model not enabled in region | Request access in P0; fall back to cross-region inference profile |
| Polly has no voice for a language | Use text in that language + Hindi/English voice; note it honestly |
| WBGT approximation questioned | Document formula, assumptions and source; Liljegren as next step |
| Telegram not used by workers in reality | Show WhatsApp as production path; voice notes work on both |
| Running out of time | Strict MVP; stretch items only after P3 is fully working |

## 9. Rules compliance

- New repo, all commits inside the event window, small frequent commits.
- AWS usage **shown on screen** in the video.
- AI coding tools listed in the writeup.
- All third-party code, libraries and data credited with licences.

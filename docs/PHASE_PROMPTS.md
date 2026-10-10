# HeatShift: Phase Execution Prompts

Planning lives in `PLAN.md`. These prompts are handed to the coding agent one phase at a time.
**Every phase must end by writing `docs/handoffs/HANDOFF_PHASE_<N>.md`**: files created/changed, public function signatures, env vars, commands to run, test status, open issues. The next phase reads only that handoff + the docs, never re-derives context.

Commit rule for all phases: small commits with clear messages (`feat(engine): ...`), push after each working step.

---

## P0: Setup
```
Read README.md and docs/*.md. Set up the heatshift repo skeleton exactly as in docs/PLAN.md section 5 (engine/, agent/, backend/, web/, tests/, docs/handoffs/). Add: pyproject.toml (Python 3.12; deps: requests, strands-agents, boto3, pytest), .env.example (AWS_REGION=ap-south-1, BEDROCK_MODEL_ID, TELEGRAM_BOT_TOKEN, TELEGRAM_WEBHOOK_SECRET, TABLE_NAME, AUDIO_BUCKET, PLACE_INDEX_NAME, ROUTE_CALCULATOR_NAME), .gitignore (Python, Node, .env, .aws-sam). Initialise backend/ with an AWS SAM template (HTTP API, one hello Lambda in Python 3.12, DynamoDB table "heatshift" PK/SK strings, on-demand). Write scripts/check_bedrock.py that calls Bedrock Converse with BEDROCK_MODEL_ID and prints the reply. Do not deploy. Write docs/handoffs/HANDOFF_PHASE_0.md.
```

## P1: Core engine (pure Python, no AWS)
```
Read docs/handoffs/HANDOFF_PHASE_0.md, docs/HEAT_SAFETY_RULES.md, docs/DATA_SOURCES.md. Implement in engine/:
1) forecast.py: get_hourly(lat, lon, date, replay=False) -> list[dict(hour, tempC, rh, windMs, solarWm2)] using Open-Meteo forecast API, or the historical archive API when replay=True; timezone Asia/Kolkata; 10s timeout; raise ForecastError with message on failure.
2) wbgt.py: compute(hourly) -> list[float] using the simplified formula in HEAT_SAFETY_RULES.md section 2; add low_sun flag when solarWm2 < 50.
3) rules.py: the two ACGIH tables as data (acclimatised / not), intensities light|moderate|heavy, tiers 75-100|50-75|25-50|0-25; function allowed_tier(wbgt, intensity, acclimatised) -> tier|"STOP". Cite source in a docstring.
4) planner.py: build_plan(hourly, wbgt, intensity, acclimatised, shift_start="08:00", shift_end="17:00", lunch=("13:00","14:00"), crew_size=40, heatwave_flag=False) -> dict(hourly=[{hour,tempC,rh,windMs,solarWm2,wbgtC,risk,workMin,restMin,note}], suggestedStart, suggestedEnd, unsafeHoursAvoided, summaryNumbers). Follow the conservative tier mapping and shift-moving logic in HEAT_SAFETY_RULES.md section 3. Pure, deterministic.
5) triage.py: assess(symptoms: list[str]) -> {"outcome": "EMERGENCY"|"REST"|"OK", "actions": [...]} per section 4.
6) metrics.py: unsafe_hours_avoided(baseline, plan, crew_size).
7) cli.py: python -m engine.cli --lat 28.58 --lon 77.20 --date 2024-05-29 --replay --intensity heavy --crew 40 prints the plan as a table + JSON.
Tests in tests/: WBGT known values (e.g. 35C/50%RH), tier boundaries for every table cell, planner never schedules work in a STOP hour, triage red flags -> EMERGENCY, metric arithmetic. All tests pass. Write docs/handoffs/HANDOFF_PHASE_1.md including the CLI output for the replay day.
```

## P2: Backend + agent on AWS
```
Read docs/handoffs/HANDOFF_PHASE_1.md and docs/ARCHITECTURE.md. Build:
1) agent/tools.py: Strands @tool wrappers: get_forecast, compute_wbgt, plan_shift (wrap engine), find_nearby_help(lat, lon, kind) using Amazon Location Service search_place_index_for_position/text + calculate_route (drive time), draft_messages(plan, languages).
2) agent/heatshift_agent.py: Strands Agent on Bedrock (BEDROCK_MODEL_ID, region from env) with a system prompt: explain plans in plain language for Indian site supervisors, never change limits, quote numbers exactly. Function run_morning_plan(site, date, replay) -> {plan, supervisorSummary, messages{lang}}. Add validate_messages(plan, messages): every time/minute in messages must exist in plan, else regenerate once, then fall back to a template.
3) backend/template.yaml (SAM): HTTP API; Lambdas api (routes in docs/ARCHITECTURE.md section 4), planner (EventBridge Scheduler cron 05:30 Asia/Kolkata, iterates active sites), telegram (webhook); DynamoDB single table + GSI1; S3 audio bucket; Location place index + route calculator; least-privilege IAM (bedrock:InvokeModel, polly:SynthesizeSpeech, dynamodb, s3, geo).
4) Polly: synthesize per language (hi-IN and en-IN voices; verify available neural voices) -> MP3 in S3 -> presigned URL (24h). Languages without a Polly voice: text only.
5) Data access layer for Site, Worker, Plan, Check-in per ARCHITECTURE.md section 3.
6) Seed script scripts/seed_demo.py: one Delhi site (replay 2024-05-29, heavy, crew 40), one Thanjavur site (live), 3 demo workers.
Deploy with sam build && sam deploy --guided (ap-south-1). Verify POST /plans/generate end to end and record curl examples. Write docs/handoffs/HANDOFF_PHASE_2.md with API base URL, resources created, env vars, curl commands.
```

## P3: Supervisor app + Telegram flows
```
Read docs/handoffs/HANDOFF_PHASE_2.md and docs/DEMO_SCRIPT.md.
1) web/: React + Vite + TypeScript + Tailwind. Pages: Sites (list/create), Today (hourly WBGT bars coloured by risk, plan timeline with work/rest blocks, agent summary with language toggle, Approve button, status chip DRAFT/APPROVED/SENT, "Replay: <date>" badge when replay), Check-ins (live list, EMERGENCY highlighted with facility + drive time), Metrics (unsafe worker-hours avoided, approval time, delivered count, 7-day history). Mobile-first, large tap targets, plain English + Hindi labels. Footer shows data sources and freshness. VITE_API_BASE env.
2) Telegram (backend telegram Lambda): verify secret header; /start links a chat to a site + worker or supervisor via code; supervisor receives draft plan with inline buttons Approve / Shift start -30 min / Reject; on Approve -> broadcast voice note + text to all site workers; workers get buttons OK / Unwell -> symptom buttons -> triage -> reply + supervisor alert (EMERGENCY includes 108 instruction, facility name, drive time, map link).
3) Deploy web to AWS Amplify Hosting; set Telegram webhook to API URL.
End-to-end test on a real phone for the full loop in docs/PLAN.md section 1. Write docs/handoffs/HANDOFF_PHASE_3.md with URLs, bot username, test results, known bugs.
```

## P4: Polish
```
Read docs/handoffs/HANDOFF_PHASE_3.md. Fix listed bugs. Add: loading/empty/error states; plan history page (compliance log); new-worker (not acclimatised) toggle per worker affecting their messages; architecture diagram image in docs/ and README; README "Run locally" and "Deploy" sections; credits/licences section; app footer with data sources. Run all tests; run the full loop twice on the deployed stack; capture the real numbers from the Delhi replay day and update docs/DEMO_SCRIPT.md placeholders. Write docs/handoffs/HANDOFF_PHASE_4.md.
```

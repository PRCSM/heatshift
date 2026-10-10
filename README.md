# HeatShift

**Safe work–rest shifts for outdoor crews, planned every morning, approved in one tap, delivered as a voice note.**

Track: **Heat and Water** · Environmental Hacks 2026 (Bharat Builds Tour, WeMakeDevs × AWS)

---

## The problem

India's summers are getting hotter, and the people most at risk are the ones who cannot go indoors.

- In April–May 2026, **97 of the world's 100 hottest cities were in India**, with Balangir (Odisha) touching **48 °C**.
- Heat deaths recorded in the 2026 pre-monsoon heatwave were linked mainly to **outdoor activity**.
- Construction's share of working hours lost to heat stress is projected to **triple to 19% by 2030** (ILO).
- State heat action plans already tell employers to **shift work timings, add cooling breaks and provide water/ORS**, but site supervisors get no tool to do it. They see "45 °C" on their phone and have to guess.

So on a site of 30–50 daily-wage workers, work usually carries on through 12–4 PM, the most dangerous hours of the day, because nobody tells the supervisor when to stop, for how long, and how often to rest.

## Who it is for

| Person | What they need |
|---|---|
| **Site supervisor / contractor** (primary user) | A clear plan for today: when to start, when to stop, how long the breaks are. One tap to approve. |
| **Workers** (20–50 per site, many without smartphones) | A short voice message in their language: today's timings, water and rest reminders, and a way to say "I feel unwell". |
| **Site owner / safety officer** | A record showing the heat plan was followed. |

## What HeatShift does

Every morning, before the shift starts:

1. **Reads the weather** for the site: hourly temperature, humidity, wind and sunshine for the day.
2. **Calculates heat stress (WBGT)** for every hour, the standard used to set safe work–rest cycles.
3. **Builds an hour-by-hour plan** using **fixed safety rules** (ISO 7243 / ACGIH work–rest tables) for the crew's work intensity. The AI never decides the safety limits.
4. **Explains it in plain words** to the supervisor: why work stops at 11:30, why breaks are every 45 minutes.
5. **Supervisor approves** with one tap (web app or Telegram).
6. **Workers get a voice note** (Amazon Polly) and a text in their language with today's timings and reminders.
7. **During the day**, any worker can report "unwell". HeatShift checks for heatstroke red flags and alerts the supervisor, with the **nearest clinic** and route, or tells them to call **108** at once.
8. **After the day**, it logs what was planned, approved and sent, and shows **unsafe worker-hours avoided**.

## The flow

```
 05:30  EventBridge Scheduler wakes the planner for each site
   │
   ▼
 Forecast (Open-Meteo) ──► WBGT per hour ──► Work–rest rules (deterministic)
   │                                              │
   └──────────────► Strands agent on Amazon Bedrock ◄┘
                    (explains plan, drafts messages, finds nearby help)
   │
   ▼
 Supervisor gets draft plan ──► [Approve] / [Edit start time] / [Reject]
   │ approved
   ▼
 Broadcast to crew: Polly voice note + text (Telegram / WhatsApp)
   │
   ▼
 Worker check-ins: "OK" / "Unwell" ──► red-flag triage ──► supervisor alert
                                                    └──► nearest clinic (Amazon Location Service)
   │
   ▼
 Daily log + metric: unsafe worker-hours avoided
```

## Why it is different

| Most heat tools | HeatShift |
|---|---|
| Show a heat map or a "feels like" number | Gives a **decision**: today's start, stop and break times |
| Built for one person checking their phone | Built for the **supervisor of a whole crew** |
| AI guesses risk | **Rule-based safety limits**; AI only explains and translates |
| English text alerts | **Voice notes in local languages**, works for workers without smartphones |
| No accountability | **Approval + log** of every plan sent |

## Benefits

- **Fewer heat illnesses**: the heaviest work moves out of the deadliest hours.
- **Work still gets done**: hours are shifted, not just cancelled (early start, late finish).
- **Proof of compliance**: contractors can show they followed the heat action plan.
- **Inclusive**: voice-first for workers with basic phones and low literacy.

## Built on AWS

| Need | Service |
|---|---|
| Agent that plans, explains and drafts messages | **Strands Agents SDK** (AWS open source) + **Amazon Bedrock** |
| API and business logic | **AWS Lambda** + **Amazon API Gateway** (deployed with **AWS SAM**) |
| Sites, workers, plans, check-ins | **Amazon DynamoDB** |
| Daily 05:30 planning run | **Amazon EventBridge Scheduler** |
| Voice notes for workers | **Amazon Polly** (audio stored in **Amazon S3**) |
| Nearest clinic, hospital, drinking water | **Amazon Location Service** (Places + Routes) |
| Supervisor web app | **AWS Amplify Hosting** |
| Logs and errors | **Amazon CloudWatch** |

Full details: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)

## Project docs

| Doc | What's inside |
|---|---|
| [`docs/PLAN.md`](docs/PLAN.md) | Full build plan: phases, tasks, timeline, owners, done-criteria |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Components, data model, API, agent tools, sequence flows |
| [`docs/HEAT_SAFETY_RULES.md`](docs/HEAT_SAFETY_RULES.md) | How WBGT is calculated and how work–rest cycles are decided |
| [`docs/DATA_SOURCES.md`](docs/DATA_SOURCES.md) | Every data source, what it gives, how fresh it is |
| [`docs/DEMO_SCRIPT.md`](docs/DEMO_SCRIPT.md) | The 3-minute video, shot by shot |
| [`docs/PHASE_PROMPTS.md`](docs/PHASE_PROMPTS.md) | Exact execution prompts for each build phase |
| [`docs/SUBMISSION_CHECKLIST.md`](docs/SUBMISSION_CHECKLIST.md) | Everything needed before we hit submit |

## Status

Planning complete. Build in progress during the event window (8–11 Oct 2026).

## Disclaimer

HeatShift is a planning aid, not a medical device. It follows published occupational heat-stress guidance and official heat advisories. In an emergency, call **108**.

## License

MIT

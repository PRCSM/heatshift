# HeatShift: 3-Minute Demo Video

Rules: under 3:00, YouTube (public or unlisted), test the link in a signed-out browser. AWS must be **visible on screen**.

| Time | Shot | Voice-over (short, plain) |
|---|---|---|
| 0:00–0:20 | Field clip / photos of a construction site at noon; headline stats on screen | "In May 2026, 97 of the world's 100 hottest cities were in India. The people working outside in that heat have no say in their hours, and their supervisor has no tool." |
| 0:20–0:35 | Persona card: Ramesh, site supervisor, 40 workers | "HeatShift is for Ramesh. Every morning it tells him when his crew should work, and when they must stop." |
| 0:35–1:10 | Web app: site, "Replay: Delhi, 29 May 2024", hourly WBGT bars turning red, plan timeline | "It reads the forecast, calculates heat stress every hour, and applies fixed safety rules. Work starts at 5:30, stops from 11:30 to 4, with breaks every 45 minutes." |
| 1:10–1:30 | Agent explanation panel (plain language, Hindi toggle) | "The AI on Amazon Bedrock explains the plan in plain words. It never changes the safety limits." |
| 1:30–1:50 | Telegram on phone: plan → **Approve** tap → crew receives voice note (play 3 sec of Polly Hindi) | "Ramesh approves with one tap. Every worker gets a voice note in their language, even on a basic phone." |
| 1:50–2:15 | Worker taps **Unwell** → "dizzy, confused" → EMERGENCY → supervisor alert with nearest hospital + drive time | "If a worker reports heatstroke signs, Ramesh is alerted at once, with the nearest hospital and route from Amazon Location Service." |
| 2:15–2:40 | Architecture diagram + quick AWS console glimpses (Lambda, DynamoDB, EventBridge schedule, Bedrock) | "Built on AWS: Strands Agents and Bedrock, Lambda, DynamoDB, EventBridge for the 5:30 run, Polly for voice, Location Service, deployed with SAM and Amplify." |
| 2:40–2:55 | Metric card: "[N] unsafe worker-hours avoided today (replay)" | "On that one heatwave day, a 40-person crew avoided [N] unsafe worker-hours." |
| 2:55–3:00 | Logo + repo URL | "HeatShift. Work the safe hours." |

Numbers in the voice-over must match what the app shows on the replay day. Update them after P4.

## Recording tips
- Record screen at 1080p; phone via screen mirroring.
- Voice-over recorded separately, clean audio.
- Captions on (judges may watch muted).

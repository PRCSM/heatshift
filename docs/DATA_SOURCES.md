# HeatShift: Data Sources

| Source | What we use | Freshness | Access | Notes |
|---|---|---|---|---|
| **Open-Meteo Forecast API** | Hourly temperature_2m, relative_humidity_2m, wind_speed_10m, shortwave_radiation, surface_pressure | Updated hourly; 7–16 day forecast | Free, no key | Primary live source |
| **Open-Meteo Historical Weather API** | Same variables for past dates | Archive (ERA5-based) | Free, no key | **Replay mode** for demo: Delhi, 29 May 2024 |
| **Amazon Location Service: Places** | Nearest hospital / clinic / drinking water | Live | AWS (place index) | Used in triage escalation |
| **Amazon Location Service: Routes** | Drive time to facility | Live | AWS (route calculator) | Shown in supervisor alert |
| **IMD heatwave warnings** | District heatwave / severe heatwave flag | Daily bulletins | mausam.imd.gov.in | MVP: manual flag per site; stretch: automate |
| **NASA POWER (Registry of Open Data on AWS)** | Historical solar radiation / climatology for validation | Daily | S3 public dataset | Optional validation of WBGT inputs |
| **Field interviews** | 2–3 short interviews with site workers / supervisors (with consent) | Event weekend | In person | For the video and problem framing |

## Honesty rules (shown in demo and README)
- Say where every number comes from and how fresh it is.
- Replay mode is clearly labelled "Replay: 29 May 2024, Delhi" on screen.
- WBGT method and its limits are stated (see `HEAT_SAFETY_RULES.md`).

## Demo locations
| Site | Lat, Lon | Use |
|---|---|---|
| Delhi (Safdarjung area) | 28.58, 77.20 | Replay of 29 May 2024 heatwave |
| Thanjavur | 10.79, 79.14 | Live forecast (local team context) |
| Balangir, Odisha | 20.70, 83.48 | 2026 heatwave hotspot (48 °C) for replay |

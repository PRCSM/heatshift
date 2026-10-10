# HeatShift: Heat Safety Rules

> Principle: **safety limits are deterministic and documented.** The AI explains and translates the plan; it never sets or changes a limit.
> Before the demo, the Rules owner must verify every number below against the cited source and record the edition used.

## 1. Why WBGT

Air temperature alone misses humidity, sun and wind. **Wet Bulb Globe Temperature (WBGT)** combines all four and is the basis of ISO 7243 and the ACGIH work–rest guidance used worldwide for outdoor workers.

## 2. How we calculate WBGT

### MVP: simplified formula (temperature + humidity)
Used by the Australian Bureau of Meteorology for general outdoor WBGT estimates:

```
e    = (RH / 100) × 6.105 × exp( 17.27 × Ta / (237.7 + Ta) )      # water vapour pressure, hPa
WBGT = 0.567 × Ta + 0.393 × e + 3.94                               # °C
```

- `Ta` = air temperature (°C), `RH` = relative humidity (%), from the hourly forecast.
- Assumes moderately high sun and light wind, which is typical of an Indian summer midday. It can overestimate at night and in heavy cloud.
- Adjustment: for hours with shortwave radiation < 50 W/m² (night or heavy cloud), we flag the value as "low-sun, conservative".

### Stretch: Liljegren method
The physics-based outdoor WBGT (Liljegren et al., 2008) uses temperature, humidity, wind, solar radiation and pressure. It replaces the simplified formula if time allows. Same interface: `compute_wbgt(hourly) -> list[float]`.

## 3. Work–rest limits

Work intensity is set per site (supervisor picks the closest):

| Intensity | Examples |
|---|---|
| Light | Supervising, light assembly, machine operation sitting |
| Moderate | Walking with light loads, painting, plastering, scaffold work |
| Heavy | Carrying bricks/cement, digging, shovelling |

### Acclimatised workers: WBGT limits (°C), ACGIH TLV table
| Work in each hour | Light | Moderate | Heavy |
|---|---|---|---|
| 75–100% | 31.0 | 28.0 | – |
| 50–75% | 31.0 | 29.0 | 27.5 |
| 25–50% | 32.0 | 30.0 | 29.0 |
| 0–25% | 32.5 | 31.5 | 30.5 |

### Not acclimatised (first ~7 days on site): ACGIH Action Limit table
| Work in each hour | Light | Moderate | Heavy |
|---|---|---|---|
| 75–100% | 28.0 | 25.0 | – |
| 50–75% | 28.5 | 26.0 | 24.0 |
| 25–50% | 29.5 | 27.0 | 25.5 |
| 0–25% | 30.0 | 29.0 | 28.0 |

"–" means that allocation is not allowed at that intensity.

### How the planner turns limits into a schedule (conservative by design)
For each hour, pick the **highest work tier whose limit ≥ that hour's WBGT**, then use the **lower bound** of that tier:

| Tier allowed | Plan for the hour | Risk label |
|---|---|---|
| 75–100% | 45 min work / 15 min rest | Low |
| 50–75% | 30 min work / 30 min rest | Moderate |
| 25–50% | 15 min work / 45 min rest | High |
| 0–25% | Light tasks only, ≤ 10 min per hour | Very high |
| Above all limits | **Stop work** | Extreme |

Then:
1. If any hour is "High" or worse, try to **move work earlier** (start up to 1.5 h before normal) and **later** (finish after 4 PM), keeping total hours ≤ shift length + 1 h.
2. Always add: water reminder every 20 min, shade breaks, ORS access.
3. On IMD heatwave-warning days (manual flag in MVP): no heavy work 12:00–15:00 regardless of WBGT, matching state heat action plans.

## 4. Unwell check-in triage (rule-based)

| Signs reported | Outcome |
|---|---|
| Confusion, slurred speech, fainting, seizure, hot dry skin / stopped sweating, repeated vomiting | **EMERGENCY**: call **108** now, move to shade, cool with water, alert supervisor, send nearest hospital + route |
| Heavy sweating, dizziness, headache, cramps, weakness, nausea | **REST**: stop work, shade, water/ORS, rest 30 min, re-check prompt; escalate if not better |
| Feeling fine | Log OK |

## 5. Metric: unsafe worker-hours avoided

```
baseline_work_min[h] = 60 if h in normal shift (minus lunch) else 0
for each hour h where baseline exceeds the safe limit:
    avoided += (baseline_work_min[h] − planned_work_min[h]) / 60 × crew_size
```

## 6. Sources to cite (verify edition before the demo)
- ISO 7243:2017, *Ergonomics of the thermal environment: assessment of heat stress using the WBGT index*.
- ACGIH, *TLVs and BEIs: Heat Stress and Strain* (TLV and Action Limit tables).
- Australian Bureau of Meteorology, "About the WBGT and Apparent Temperature indices" (simplified WBGT formula).
- Liljegren J.C. et al. (2008), "Modeling the Wet Bulb Globe Temperature Using Standard Meteorological Measurements", *J. Occup. Environ. Hyg.*
- NDMA *Guidelines for Preparation of Action Plan: Prevention and Management of Heat Wave*, and state heat action plans (e.g. Telangana, Ahmedabad).
- NIOSH *Criteria for a Recommended Standard: Occupational Exposure to Heat and Hot Environments* (hydration guidance).

## 7. Disclaimer
Planning aid only, not a medical device. Official IMD warnings and state advisories take priority.

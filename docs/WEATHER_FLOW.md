# Weather Flow — Design & Architecture

## Overview

Node-RED utility flow providing weather awareness, precipitation sensing, and actionable notifications. Designed around a polling state machine that scales API call frequency to actual weather threat level, keeping costs predictable while maintaining responsiveness during active events.

---

## API Provider

**Selected:** Pirate Weather  
**Cost:** ~$5/month (personal tier)  
**Rationale:** Dark Sky-compatible API with strong HRRR model sourcing for near-term minutely precision. Weatherbit ($45/month) was evaluated and rejected on cost; Pirate Weather provides sufficient resolution for the use case.

**Base URL:** `https://api.pirateweather.net/forecast/[apikey]/[lat],[lon]`

**Standard parameters — always include:**
```
?units=us&version=2
```

- `units=us` — Fahrenheit, inches, mph/mph-adjacent. Confirmed from live response.
- `version=2` — Unlocks type-specific precipitation fields (`rainIntensity`, `snowIntensity`, `iceIntensity`, `liquidAccumulation`, `snowAccumulation`, `iceAccumulation`), `cape`, `currentDaySnow/Liquid/Ice`, and ensemble spread (`precipIntensityError`). Always include; no reason not to.

**Credentials:** Store in `secrets.json` as `weather_api_key`.

---

## Location Context

**Coordinates:** Configured in `secrets.json` as `location.lat` / `location.lon`  
**Elevation:** 367 ft  
**Nearest API city:** Gardnertown — confirms coordinate-based grid lookup, not population-center snap.

**Microclimate note:** Hudson Valley terrain creates significant hyperlocal variation. The property sits at elevation; the Hudson River to the east creates thermal and moisture dynamics that can shift the rain/snow line within a few miles. Forecasts snapped to a nearby city center would often be wrong for this location. The HRRR grid cell resolves to within ~1.5 mi, which is about as good as gridded models get. Systematic bias analysis during snow events should factor in elevation and river distance.

---

## Data Source Stack

From `flags.sources` in live response — model stack and approximate validity windows:

| Model | Role | Window |
|-------|------|--------|
| `hrrrsubh` | Sub-hourly HRRR (minutely precision) | 0–18h |
| `rtma_ru` | Real-time mesoscale analysis | Current |
| `hrrr_0-18` | HRRR hourly | 0–18h |
| `nbm` | National Blend of Models | Short-range |
| `ecmwf_ifs` | ECMWF IFS | Extended |
| `hrrr_18-48` | HRRR extended | 18–48h |
| `gfs` | Global Forecast System | Extended |
| `gefs` | GEFS ensemble | Extended (spread source) |

HRRR subhourly drives minutely data quality — this is the primary reason Pirate Weather was selected for near-term precipitation sensing.

---

## Known API Quirks

**`-999` sentinel value:** Several fields return `-999` when data is unavailable rather than null/absent. Must null-guard in processing. Confirmed fields:
- `smoke`, `fireIndex` — only available for ~first 36 hours; returns `-999` beyond that
- `smokeMax`, `fireIndexMax` in daily — returns `-999` when not available

**`sleetIntensity` in minutely:** Present in live response but undocumented. Treat as bonus data, not guaranteed. Fields documented for minutely: `rainIntensity`, `snowIntensity`. `sleetIntensity` also appears in practice.

**`units` confirmation:** Live response confirmed `units: "us"` in flags block. All thresholds and logic should be written against US units (°F, inches, in/hr).

**`processTime`:** ~41ms in live response. Not a bottleneck even at 1-minute polling intervals.

---

## Rate Limit Monitoring

Response headers expose real-time consumption:

| Header | Content |
|--------|---------|
| `Ratelimit-Limit` | Monthly quota |
| `Ratelimit-Remaining` | Calls remaining this month |
| `Ratelimit-Reset` | Seconds until reset |
| `X-Forecast-API-Calls` | Total calls this month |
| `X-Response-Time` | Processing time (ms) |

**Flow behavior:** Log `X-Forecast-API-Calls` on every response. Alert (normal priority) if >50% quota consumed before 50% through month. Alert (high priority) if >80% consumed before 80% through month. Emergency fallback: force POLL_DORMANT if approaching limit.

---

## Polling State Machine

The core cost-control mechanism. Polling frequency scales with weather threat level.

### States

| State | Poll Interval | Purpose |
|-------|---------------|---------|
| `POLL_DORMANT` | 15 min | No precipitation expected; baseline monitoring |
| `POLL_MONITOR` | 5 min | Precipitation possible within forecast window; elevated attention |
| `POLL_ACTIVE` | 1 min | Precipitation imminent or occurring; full resolution |
| `POLL_LIGHTNING` | 1 min | Thunderstorm active; same rate as ACTIVE, distinct state for notification logic |

### State Transitions

**DORMANT → MONITOR:**
- Any hourly block within next 6 hours: `precipProbability > 0.40`

**MONITOR → ACTIVE:**
- Any minutely block: `precipIntensity > 0.01 in/hr` AND `precipProbability > 0.70`
- OR: Any hourly block within next 2 hours: `precipProbability > 0.80`

**ACTIVE → MONITOR:**
- All minutely blocks: `precipIntensity < 0.005 in/hr` for 30 consecutive minutes
- AND: No hourly block within next 2 hours exceeds `precipProbability > 0.70`

**MONITOR → DORMANT:**
- All hourly blocks within next 6 hours: `precipProbability < 0.25`
- Hold minimum 15 minutes in MONITOR before allowing DORMANT transition (prevent rapid oscillation)

**Any → LIGHTNING:**
- `cape > 2500 J/kg` in current or next 2 hourly blocks AND `precipProbability > 0.50`

**LIGHTNING → ACTIVE:**
- `cape < 500 J/kg` sustained for 2 consecutive polls AND no lightning detection

### `precipIntensityError` as Confidence Gate

`precipIntensityError` is the standard deviation of `precipIntensity` across GEFS/ECMWF ensemble members. High spread = model disagreement = less trustworthy forecast.

Application: When evaluating escalation thresholds, weight high-spread forecasts conservatively. Specifically: if `precipIntensityError > precipIntensity * 0.75` (error is >75% of the predicted intensity), treat the forecast as lower confidence and require higher `precipProbability` to trigger escalation.

*Specific threshold values TBD during calibration.*

### `exclude` Parameter Strategy

Reduce response size and processing load by excluding unneeded blocks per state:

| State | `exclude` parameter |
|-------|---------------------|
| `POLL_DORMANT` | `minutely,alerts` |
| `POLL_MONITOR` | `alerts` |
| `POLL_ACTIVE` | *(none)* |
| `POLL_LIGHTNING` | *(none)* |

---

## Precipitation State Machine

Tracks the current precipitation event lifecycle. Separate from polling state.

### States

| State | Description |
|-------|-------------|
| `PRECIP_NONE` | No precipitation |
| `PRECIP_IMMINENT` | High probability within 30 minutes per minutely |
| `PRECIP_ACTIVE` | Currently precipitating |
| `PRECIP_TAPERING` | Intensity declining; event may be ending |
| `PRECIP_DONE` | Event ended; cooling-off before returning to NONE |

### Precipitation Type Tracking

Uses v2 type-specific intensity fields directly rather than inferring from generic `precipIntensity` + `precipType`:

| Field | Use |
|-------|-----|
| `rainIntensity` | Rain rate (in/hr) |
| `snowIntensity` | Snow rate (in/hr liquid equivalent) |
| `iceIntensity` | Freezing rain rate (in/hr) |
| `currentDaySnow` | Accumulated snow since midnight (in) |
| `currentDayLiquid` | Accumulated liquid since midnight (in) |
| `currentDayIce` | Accumulated ice since midnight (in) |

**Derived `precipitation_type` enum:** `rain | snow | ice | sleet | mixed | none`
- Determined from which type-specific intensity fields are non-zero
- `mixed` when multiple types simultaneously present

---

## Snow Depth Sensing

**Hardware:** Ultrasonic depth sensor (TBD model) on fixed mount at known height above ground.

**Purpose:** Ground truth for snow accumulation. Validates API snow predictions and provides actual depth data the API cannot provide.

**Derived metrics:**
- `snow_depth_inches` — raw sensor reading converted to depth
- `snow_event_active` — boolean, derived from depth change rate
- `snow_accumulation_rate` — in/hr, rolling window

**Calibration:** Sensor baseline established on bare ground. Requires re-zero after ground conditions change (leaves, etc.).

**Integration with API:** During snow events, compare `currentDaySnow` (API) against local depth sensor accumulation. Log discrepancies. Over time, build a calibration profile for this location's systematic bias. Hudson Valley elevation/temperature effects expected to cause occasional divergence.

---

## Notifications

### Principles

- Notify on event start, significant changes, and event end
- No spam: notification cooldown periods between same-type alerts
- Severity maps to notification framework severity levels

### Notification Triggers

| Trigger | Severity | Notes |
|---------|----------|-------|
| Rain event starting | `low` | During waking hours only unless heavy |
| Heavy rain starting (>0.3 in/hr) | `medium` | Any time |
| Snow event starting | `medium` | Always |
| Significant snow accumulation (>2") | `high` | Based on sensor |
| Ice/freezing rain detected | `high` | Type-specific field trigger |
| Thunderstorm imminent | `high` | CAPE threshold + precip |
| Severe weather alert | `high` or `critical` | From `alerts` block |
| Event ended after significant accumulation | `low` | Summary notification |

### Daily Digest Integration

Weather section in daily digest (see NODERED_PATTERNS.md Daily Digest):
- Today summary + high/low
- Tonight summary + low
- Significant events in next 48h

---

## MQTT Topics

**Published by Weather Flow:**

```
highland/event/weather/period_change       # day/evening/overnight period (from scheduler)
highland/event/weather/precipitation_start # precip event began
highland/event/weather/precipitation_end   # precip event ended
highland/event/weather/precipitation_type  # type changed during event
highland/event/weather/alert               # NWS alert received
highland/status/weather/health             # flow health heartbeat
```

**State retained topics:**
```
highland/event/weather/current             # retained, current conditions summary
highland/event/weather/forecast_summary    # retained, day summary
```

**Polling state (for diagnostics/dashboards):**
```
highland/status/weather/polling_state      # retained, current polling state
```

---

## Configuration

**In `thresholds.json`:**
```json
{
  "weather": {
    "poll_dormant_to_monitor_probability": 0.40,
    "poll_monitor_to_active_probability": 0.70,
    "poll_monitor_to_active_intensity": 0.01,
    "poll_active_to_monitor_intensity_clear": 0.005,
    "poll_active_to_monitor_sustained_minutes": 30,
    "poll_lightning_cape": 2500,
    "heavy_rain_intensity": 0.30,
    "snow_notification_accumulation_inches": 2.0,
    "ensemble_spread_confidence_ratio": 0.75
  }
}
```

All threshold values are initial estimates. Calibrate against observed events.

---

## Open Questions

- [ ] Ultrasonic sensor hardware selection and mounting
- [ ] Parallel run vs Weatherbit for snow prediction scoring — defer until sensor deployed
- [ ] Optimal `precipIntensityError` confidence gate threshold — calibrate from observed events
- [ ] Whether `day_night` block adds value over hourly parsing for notification content
- [ ] Alert polling cadence — likely DORMANT-equivalent (15 min)
- [ ] Whether `nearestStormDistance` / `nearestStormBearing` useful for lightning threat lead time
- [ ] Version monitoring — check for Pirate Weather API versions beyond v2 when beginning implementation

---

## Implementation Notes

- API key in `secrets.json` as `config.secrets.weather_api_key`
- All `-999` values must be null-guarded before use in logic or display
- `sleetIntensity` in minutely: use if present, don't depend on it
- Rate limit headers: log on every response, alert on consumption anomalies
- State machines: persist state in flow context (disk-backed) to survive restarts
- HRRR subhourly source freshness: check `flags.sourceTimes.hrrr_subh` to confirm data recency

---

*Last Updated: 2026-03-13*

# OpenTelemetry Hackathon - Welcome!

## Quick Start (4 Steps)

### 1. Create Your Config File
```bash
cp .env.override.template .env.override
```

### 2. Add Your API Key
```bash
code .env.override
```
- Change `TEAM_NAME=team-1` to your team name
- Replace `ASK_FACILITATOR_FOR_KEY` with the API key provided

### 3. Start the Demo
```bash
make start
```
Wait ~2 minutes for all services to start.

### 4. Open the Store
Click the popup or go to: **PORTS tab -> Port 8080 -> Globe icon**

---

## Your Config File

**Edit this file for challenges:**
```bash
code src/otel-collector/otelcol-config-extras.yml
```

**After ANY change:**
```bash
make restart service=otel-collector
```

---

## Challenges

| # | Challenge | Difficulty |
|---|-----------|------------|
| 6 | Business Context | Easy |
| 1 | Span Metrics | Easy |
| 4 | Head Sampling | Easy |
| 3 | Drop Fields | Medium |
| 2 | Redact UUIDs | Medium |
| 5 | Tail Sampling | Hard |

---

## Help

- **Logs:** `docker logs otel-collector --tail=50`
- **Status:** `docker compose ps`

**Good luck!**

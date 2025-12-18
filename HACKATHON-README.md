# OpenTelemetry Hackathon

## The Scenario

You're the on-call engineer at a financial services firm. Compliance just discovered **PII leaking into your observability platform** - credit card numbers, passwords, and customer data are visible in production traces.

Your mission: **Fix the data leaks before the audit.**

You'll use the OpenTelemetry Collector to filter, mask, and redact sensitive data - all without changing application code.

---

## Quick Start

### 1. Create Your Config File
```bash
cp .env.override.template .env.override
```

### 2. Configure Your Team
```bash
code .env.override
```
- Change `TEAM_NAME=team-1` to your actual team name (e.g., `team-alpha`)
- The facilitator will provide the API key

### 3. Start the Demo
```bash
make start
```
Wait ~2 minutes for all services to start.

### 4. Open the Store
Click the popup or go to: **PORTS tab -> Port 8080 -> Globe icon**

(Or http://localhost:8080 if running locally)

Browse the store to generate trace data!

---

## Your Mission Control: Coralogix

All verification happens in Coralogix. Get the URL from your facilitator:

| View | URL | Use For |
|------|-----|---------|
| **LiveTail** | `https://[FACILITATOR_PROVIDED].coralogix.com/#/livetail` | Real-time logs |
| **Tracing** | `https://[FACILITATOR_PROVIDED].coralogix.com/#/explore` | Span attributes (where PII lives) |

> **Note:** The facilitator will provide the actual Coralogix URL at the start of the hackathon.

### Finding Your Team's Data
1. Open LiveTail or Explore
2. Filter by **Subsystem** = `YOUR_TEAM_NAME`
3. Click **Prettify** to make JSON readable

> **Important:** Your `TEAM_NAME` in `.env.override` becomes the Subsystem filter in Coralogix. This isolates your team's data.

---

## Challenges

| # | Challenge | Points | Difficulty | Verify In |
|---|-----------|--------|------------|-----------|
| 0 | Hello Coralogix | 10 | Warmup | LiveTail - see your logs |
| 1 | Find the PII | 20 | Discovery | Screenshot of PII found |
| 2 | Delete Secrets | 30 | Easy | Tracing - `db.password` gone |
| 3 | Mask Credit Cards | 40 | Medium | Tracing - cards show `****` |
| 4 | Hash Emails | 40 (+10 bonus) | Medium | Tracing - see `email_hash` |
| 5 | Zero-Trust Mode | 50 | Hard | Tracing - only allowed keys |
| 6 | Tail Sampling | 50 | Hard | Tracing - errors kept, rest sampled |

**Total Points: 240** (250 with bonus)

---

## Challenge Details

### Challenge 0: Hello Coralogix (Warmup)
**Goal:** Verify your environment works and data flows to Coralogix.

1. Ensure your `TEAM_NAME` is set in `.env.override`
2. Run `make start` and wait for services to start
3. Browse the demo store to generate traffic
4. Open **LiveTail** in Coralogix (URL from facilitator)
5. Filter by **Subsystem** = your team name
6. You should see logs streaming!

**Screenshot:** LiveTail showing your team's logs

---

### Challenge 1: Find the PII (Discovery)
**Goal:** Use Coralogix to discover what sensitive data is leaking.

No code changes needed - this is pure investigation!

1. Go to **Explore > Tracing** in Coralogix
2. Filter by **Subsystem** = your team name
3. Look at traces from these services:
   - `oteldemo.CheckoutService`
   - `oteldemo.PaymentService`
   - `oteldemo.EmailService`
4. Click on spans and examine the **attributes**
5. Find and document:
   - Credit card numbers (hint: look for "card")
   - Database passwords (hint: look for "db.")
   - Email addresses (hint: look for "email")
   - Session tokens (hint: look for "token")

**Screenshot:** A trace span showing PII in the attributes

---

### Challenge 2: Delete Secrets (Easy)
**Goal:** Remove database credentials and API secrets from traces.

1. Open `src/otel-collector/otelcol-config-extras.yml`
2. Find and uncomment the `attributes/delete-secrets` processor
3. Add `attributes/delete-secrets` to the traces pipeline (see instructions in file)
4. Run: `make restart service=otel-collector`
5. Wait 1-2 minutes, then check Coralogix

**Before:** `db.password: "Pr0d_P@ssw0rd_2024!"`
**After:** Attribute is completely gone

**Screenshot:** Trace span from CheckoutService with NO `db.password` attribute

---

### Challenge 3: Mask Credit Cards (Medium)
**Goal:** Redact credit card numbers while keeping the attribute present.

1. Open `src/otel-collector/otelcol-config-extras.yml`
2. Find and uncomment the `redaction/credit-cards` processor
3. Add `redaction/credit-cards` to the traces pipeline
4. Run: `make restart service=otel-collector`
5. Wait 1-2 minutes, then check Coralogix

**Before:** `user.credit_card: "4532-8721-9012-3456"`
**After:** `user.credit_card: "****"`

**Screenshot:** Trace span showing masked credit card values

---

### Challenge 4: Hash Emails for Analytics (Medium)
**Goal:** Replace email addresses with SHA256 hashes for pseudonymized analytics.

1. Open `src/otel-collector/otelcol-config-extras.yml`
2. Find and uncomment the `transform/hash-pii` processor
3. Add `transform/hash-pii` to the traces pipeline
4. Run: `make restart service=otel-collector`
5. Wait 1-2 minutes, then check Coralogix

**Before:** `user.email: "john.smith@example.com"`
**After:** `user.email_hash: "a8f2d9e1c4b7..."` (64-character hex)

The original `user.email` should be gone!

**Screenshot:** Trace span showing `user.email_hash` (no `user.email`)

---

### Challenge 5: Zero-Trust Allowlist (Hard)
**Goal:** Implement fail-closed security - only explicitly allowed attributes pass through.

> **WARNING:** This is destructive! Most span attributes will be removed. Only enable this if you understand the implications.

1. Open `src/otel-collector/otelcol-config-extras.yml`
2. Find and uncomment the `redaction/allowlist` processor
3. Add `redaction/allowlist` to the traces pipeline
4. Run: `make restart service=otel-collector`
5. Wait 1-2 minutes, then check Coralogix

**Before:** 20+ attributes per span
**After:** Only 5-10 explicitly allowed attributes remain

**Screenshot:** Trace span showing minimal attributes (only allowed keys)

---

### Challenge 6: Intelligent Tail Sampling (Hard)
**Goal:** Keep 100% of errors, sample 10% of successful requests to reduce costs.

1. Open `src/otel-collector/otelcol-config-extras.yml`
2. Find and uncomment the `tail_sampling/intelligent` processor
3. Add `tail_sampling/intelligent` to the traces pipeline **BEFORE batch**
4. Run: `make restart service=otel-collector`
5. Generate some traffic, including errors if possible
6. Wait 1-2 minutes, then check Coralogix

**Verification:** All error traces should be present. Successful traces should be reduced.

**Screenshot:** Show an error trace that was kept (proves errors aren't dropped)

---

## Your Config File

**Edit this file for all challenges:**
```bash
code src/otel-collector/otelcol-config-extras.yml
```

**After EVERY change:**
```bash
make restart service=otel-collector
```

> **Important:** Data takes 1-2 minutes to appear in Coralogix after a restart. Be patient!

---

## Logs vs Traces: Know the Difference

| Data Type | Where to Find | What You'll See |
|-----------|---------------|-----------------|
| **Logs** | LiveTail | Real-time log messages |
| **Traces** | Explore > Tracing | Span attributes (where PII lives!) |

The injected PII appears in **trace span attributes**, not logs. For Challenges 2-6, use **Explore > Tracing** to verify your fixes.

---

## Submit Your Work

After completing each challenge:

1. Take a screenshot showing your success
2. Make sure your **team name** is visible in the Subsystem filter
3. Post to the Miro board (facilitator will provide link)
4. Label your screenshot: `Team-[NAME] - Challenge [N]`

### Screenshot Requirements

| Challenge | What to Screenshot | Must Show |
|-----------|-------------------|-----------|
| 0 | LiveTail with logs | Subsystem filter = your team, logs visible |
| 1 | Trace span with PII | Expanded span showing `user.credit_card`, `db.password` |
| 2 | Trace span WITHOUT secrets | `db.password` and `api.secret` are GONE |
| 3 | Trace span with masked cards | `user.credit_card` shows `****` |
| 4 | Trace span with hash | `user.email_hash` visible, `user.email` GONE |
| 5 | Trace span minimal | Only allowed keys remain |
| 6 | Error trace kept | Show an error trace that wasn't dropped |

### Screenshot Tips
- Use **Prettify** in LiveTail to make JSON readable
- In Explore > Tracing, click a span to expand its attributes
- Make sure your **Subsystem filter** (team name) is visible
- Highlight or circle the relevant attribute if helpful

---

## Troubleshooting

### Check collector logs
```bash
docker logs otel-collector --tail=50
```

### Check service status
```bash
docker compose ps
```

### Common Issues

| Problem | Solution |
|---------|----------|
| "unknown processor type" | Check processor name spelling |
| YAML indentation error | Use 2 spaces, not tabs |
| Processor not working | Did you add it to the pipeline array? |
| No data in Coralogix | Wait 1-2 minutes after restart |
| Connection refused | Check API key in `.env.override` |

See [CHEAT-SHEET.md](CHEAT-SHEET.md) for more help.

---

## Pipeline Order

When adding processors, order matters! The final pipeline should look like:

```yaml
processors: [
  resourcedetection,      # Base
  memory_limiter,         # Base
  transform,              # Base - URL cleanup
  transform/inject-pii,   # Base - Creates the problem
  attributes/delete-secrets,  # Challenge 2
  transform/hash-pii,     # Challenge 4 (before redaction!)
  redaction/credit-cards, # Challenge 3
  redaction/allowlist,    # Challenge 5
  tail_sampling/intelligent,  # Challenge 6 (before batch!)
  batch                   # Base - Always last
]
```

---

**Good luck! May your traces be clean and your compliance audits be boring.**

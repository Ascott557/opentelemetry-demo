# OpenTelemetry Hackathon - Cheat Sheet

Copy-paste processor configs and troubleshooting reference.

---

## Processor Configs (Copy-Paste)

Copy these into the `processors:` section of `otelcol-config-extras.yml`.
Then add the processor name to your pipeline.

---

### Delete Secrets (30 pts)

**Goal:** Remove database passwords and API secrets from traces.

```yaml
  attributes/delete-secrets:
    actions:
      - key: db.password
        action: delete
      - key: db.connection_string
        action: delete
      - key: api.secret
        action: delete
      - key: session.token
        action: delete
```

**Add to pipeline:** `attributes/delete-secrets`

**Verify:** Explore → Tracing → CheckoutService → `db.password` should be GONE

---

### Mask Credit Cards (40 pts)

**Goal:** Redact credit card numbers to `****` for PCI-DSS compliance.

```yaml
  redaction/credit-cards:
    allow_all_keys: true
    blocked_values:
      - "4[0-9]{3}[- ]?[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}"
      - "5[1-5][0-9]{2}[- ]?[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}"
      - "[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}"
```

**Add to pipeline:** `redaction/credit-cards`

**Verify:** Explore → Tracing → PaymentService → `user.credit_card` shows `****`

---

### Hash Emails (40 pts)

**Goal:** Replace emails with SHA256 hashes for pseudonymized analytics.

```yaml
  transform/hash-pii:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - set(attributes["user.email_hash"], SHA256(attributes["user.email"])) where attributes["user.email"] != nil
          - delete_key(attributes, "user.email")
```

**Add to pipeline:** `transform/hash-pii` (BEFORE any redaction processors!)

**Verify:** Explore → Tracing → CheckoutService → see `user.email_hash`, no `user.email`

**BONUS (+10 pts):** Also hash `user.ssn` and `customer.email` - add these statements:
```yaml
          - set(attributes["user.ssn_hash"], SHA256(attributes["user.ssn"])) where attributes["user.ssn"] != nil
          - delete_key(attributes, "user.ssn")
          - set(attributes["customer.email_hash"], SHA256(attributes["customer.email"])) where attributes["customer.email"] != nil
          - delete_key(attributes, "customer.email")
```

---

### Zero-Trust Allowlist (50 pts)

**Goal:** Only allow explicitly approved attributes. Everything else is dropped.

**WARNING:** This is destructive! Most span attributes will be removed.

```yaml
  redaction/allowlist:
    allow_all_keys: false
    allowed_keys:
      - http.method
      - http.status_code
      - http.route
      - http.url
      - rpc.service
      - rpc.method
      - service.name
      - span.kind
      - status.code
      - otel.status_code
    summary: debug
```

**Add to pipeline:** `redaction/allowlist`

**Verify:** Explore → Tracing → Only allowed keys remain

---

### Tail Sampling (50 pts)

**Goal:** Keep 100% of errors, sample 10% of successful requests.

```yaml
  tail_sampling/intelligent:
    decision_wait: 10s
    num_traces: 50000
    policies:
      - name: keep-all-errors
        type: status_code
        status_code:
          status_codes:
            - ERROR
      - name: keep-slow-traces
        type: latency
        latency:
          threshold_ms: 2000
      - name: sample-rest
        type: probabilistic
        probabilistic:
          sampling_percentage: 10
```

**Add to pipeline:** `tail_sampling/intelligent` (BEFORE `batch`!)

**Verify:** Error traces always appear, successful traces reduced to ~10%

---

## Pipeline Order

When adding multiple processors, order matters:

```yaml
processors: [
  resourcedetection,
  memory_limiter,
  transform,
  transform/inject-pii,
  attributes/delete-secrets,     # Delete Secrets
  transform/hash-pii,            # Hash Emails (BEFORE redaction!)
  redaction/credit-cards,        # Mask Credit Cards
  redaction/allowlist,           # Zero-Trust
  tail_sampling/intelligent,     # Tail Sampling (BEFORE batch!)
  batch,
]
```

---

## Where to Find Things in Coralogix

| What | Where | URL |
|------|-------|-----|
| Real-time logs | LiveTail | `https://[FACILITATOR_PROVIDED].coralogix.com/#/livetail` |
| Trace spans | Explore > Tracing | `https://[FACILITATOR_PROVIDED].coralogix.com/#/explore` |
| Your team's data | Filter by Subsystem | Use dropdown, select YOUR_TEAM_NAME |

> **Note:** Get the actual Coralogix URL from your facilitator.

### How to Filter by Your Team
1. Open LiveTail or Explore
2. Click the **"Select Subsystems"** dropdown
3. Choose your team name (matches `TEAM_NAME` in `.env.override`)
4. Only your team's data will appear

---

## PII Attribute Reference

| Attribute | Example Value | Service | Fix With |
|-----------|---------------|---------|----------|
| `user.credit_card` | `4532-8721-9012-3456` | PaymentService | Mask Credit Cards |
| `payment.card_number` | `5425-2334-3010-9903` | CheckoutService | Mask Credit Cards |
| `db.password` | `Pr0d_P@ssw0rd_2024!` | Any with db.system | Delete Secrets |
| `db.connection_string` | `postgresql://admin:...` | PostgreSQL spans | Delete Secrets |
| `api.secret` | `FAKE_API_KEY_do_not_use_12345` | PaymentService | Delete Secrets |
| `session.token` | `eyJhbGciOiJIUzI1Ni...` | CheckoutService | Delete Secrets |
| `user.email` | `john.smith@example.com` | CheckoutService | Hash Emails |
| `customer.email` | `jane.doe@example.com` | EmailService | Hash Emails |
| `user.ssn` | `287-65-4321` | CheckoutService | Hash Emails (bonus) |
| `customer.phone` | `+1-212-555-0142` | CheckoutService | Zero-Trust |
| `billing.address` | `123 Main Street...` | CheckoutService | Zero-Trust |
| `shipping.address` | `456 Commerce Blvd...` | ShippingService | Zero-Trust |

---

## Common Errors & Fixes

### "unknown processor type"
**Cause:** Typo in processor name
**Fix:** Check spelling exactly matches:
- `attributes/delete-secrets`
- `redaction/credit-cards`
- `transform/hash-pii`
- `redaction/allowlist`
- `tail_sampling/intelligent`

### YAML Indentation Error
**Cause:** Using tabs or wrong number of spaces
**Fix:** Use exactly 2 spaces for each indentation level. Never use tabs.

### Processor Not Working
**Cause:** Processor defined but not added to pipeline
**Fix:** You must do TWO things:
1. Add the processor config to the `processors:` section
2. Add the processor name to the pipeline array

### No Data in Coralogix
**Causes:**
1. Data takes 1-2 minutes to propagate after restart
2. Wrong team name filter
3. API key not set

**Fixes:**
1. Wait 1-2 minutes after `make restart service=otel-collector`
2. Check Subsystem filter matches your `TEAM_NAME` exactly
3. Check `.env.override` has the correct `CX_API_KEY`

### Connection Refused
**Cause:** Collector can't connect to Coralogix
**Fix:** 
1. Check `CX_API_KEY` is set in `.env.override`
2. Check `CX_DOMAIN` matches what facilitator provided
3. Run `docker logs otel-collector --tail=50` to see errors

---

## Useful Commands

```bash
# Restart collector after config change
make restart service=otel-collector

# Check collector logs for errors
docker logs otel-collector --tail=50

# Check all service status
docker compose ps

# Restart everything if broken
make stop && make start
```

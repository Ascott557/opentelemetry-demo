# OpenTelemetry Hackathon - Cheat Sheet

Quick reference for troubleshooting and verification.

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

Here's what PII is injected and where to find it:

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
- `attributes/delete-secrets` (not `attribute/...`)
- `redaction/credit-cards` (not `redact/...`)
- `transform/hash-pii` (not `transformer/...`)
- `tail_sampling/intelligent` (not `tailsampling/...`)

### YAML Indentation Error
**Cause:** Using tabs or wrong number of spaces
**Fix:** Use exactly 2 spaces for each indentation level. Never use tabs.

```yaml
# WRONG (tabs or 4 spaces)
processors:
    transform:
        error_mode: ignore

# CORRECT (2 spaces)
processors:
  transform:
    error_mode: ignore
```

### Processor Not Working
**Cause:** Processor defined but not added to pipeline
**Fix:** You must do TWO things:
1. Uncomment the processor definition
2. Add the processor name to the pipeline

```yaml
# Step 1: Uncomment the processor (you did this)
attributes/delete-secrets:
  actions:
    - key: db.password
      action: delete

# Step 2: Add to pipeline (you might have forgotten this!)
service:
  pipelines:
    traces:
      processors: [..., attributes/delete-secrets, batch]  # <-- ADD IT HERE
```

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
2. Check `CX_DOMAIN` matches what facilitator provided (e.g., `ingress.eu2.coralogix.com:443`)
3. Run `docker logs otel-collector --tail=50` to see errors

### Collector Won't Start
**Cause:** Invalid YAML syntax
**Fix:**
1. Check `docker logs otel-collector --tail=50` for error message
2. Look for line number in error
3. Common issues: missing colon, wrong indentation, unclosed quotes

---

## Verification Examples

### Challenge 2: Delete Secrets

**BEFORE (problem):**
```json
{
  "name": "CheckoutService/PlaceOrder",
  "attributes": {
    "db.password": "Pr0d_P@ssw0rd_2024!",
    "db.connection_string": "postgresql://admin:secretpass123@...",
    "api.secret": "FAKE_API_KEY_do_not_use_12345"
  }
}
```

**AFTER (success):**
```json
{
  "name": "CheckoutService/PlaceOrder",
  "attributes": {
    // db.password is GONE
    // db.connection_string is GONE
    // api.secret is GONE
  }
}
```

### Challenge 3: Mask Credit Cards

**BEFORE (problem):**
```json
{
  "name": "PaymentService/Charge",
  "attributes": {
    "user.credit_card": "4532-8721-9012-3456",
    "payment.card_number": "5425-2334-3010-9903"
  }
}
```

**AFTER (success):**
```json
{
  "name": "PaymentService/Charge",
  "attributes": {
    "user.credit_card": "****",
    "payment.card_number": "****"
  }
}
```

### Challenge 4: Hash Emails

**BEFORE (problem):**
```json
{
  "name": "CheckoutService/PlaceOrder",
  "attributes": {
    "user.email": "john.smith@example.com"
  }
}
```

**AFTER (success):**
```json
{
  "name": "CheckoutService/PlaceOrder",
  "attributes": {
    "user.email_hash": "a8f2d9e1c4b73f5a2e1d8c9b7a6f5e4d3c2b1a0..."
  }
}
```
Note: `user.email` is completely gone, replaced by `user.email_hash`

### Challenge 5: Zero-Trust Allowlist

**BEFORE (problem):**
```json
{
  "name": "CheckoutService/PlaceOrder",
  "attributes": {
    "http.method": "POST",
    "http.status_code": 200,
    "user.email": "john.smith@...",
    "db.password": "secret",
    "customer.phone": "+1-212-...",
    // ... 20+ more attributes
  }
}
```

**AFTER (success):**
```json
{
  "name": "CheckoutService/PlaceOrder",
  "attributes": {
    "http.method": "POST",
    "http.status_code": 200,
    "rpc.service": "oteldemo.CheckoutService",
    "rpc.method": "PlaceOrder",
    "service.name": "checkoutservice"
    // ONLY allowed keys remain - everything else is gone
  }
}
```

---

## Useful Commands

```bash
# Restart collector after config change
make restart service=otel-collector

# Check collector logs for errors
docker logs otel-collector --tail=50

# Check all service status
docker compose ps

# Follow collector logs in real-time
docker logs -f otel-collector

# Restart everything if things are broken
make stop && make start
```

---

## Pipeline Order Quick Reference

When all challenges are enabled, your pipeline should be:

```yaml
processors: [
  resourcedetection,        # 1. Base
  memory_limiter,           # 2. Base
  transform,                # 3. Base
  transform/inject-pii,     # 4. Base (the problem)
  attributes/delete-secrets,# 5. Challenge 2
  transform/hash-pii,       # 6. Challenge 4
  redaction/credit-cards,   # 7. Challenge 3
  redaction/allowlist,      # 8. Challenge 5
  tail_sampling/intelligent,# 9. Challenge 6
  batch                     # 10. ALWAYS LAST
]
```

**Remember:**
- `transform/hash-pii` must come BEFORE `redaction/` processors
- `tail_sampling/intelligent` must come BEFORE `batch`
- `batch` is ALWAYS last

---

## Still Stuck?

1. Check the error message in `docker logs otel-collector --tail=50`
2. Compare your YAML to the examples in `otelcol-config-extras.yml`
3. Make sure processor is both UNCOMMENTED and ADDED TO PIPELINE
4. Ask a facilitator for help!

**Good luck!**

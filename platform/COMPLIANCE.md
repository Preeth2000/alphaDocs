---
page: Compliance
tags:
  - compliance
  - legal
  - security
---

# Compliance Notes

[[README]] · [[platform/Key-Decisions]] · [[platform/Tech-Stack]]

Open compliance items requiring human review before commercial launch.

---

## T212 API Terms of Service

**Status: Unreviewed**

alphaTrade integrates with the Trading 212 REST API for order execution.

**Action required:** A human must read the [T212 API Terms of Service](https://t212public-api-docs.redoc.ly/) before any live-money trading begins. Key questions:

- Is fully automated order execution (no human confirmation per trade) permitted for personal accounts?
- Are there rate limits or usage restrictions that affect algorithmic strategies?
- If other users are ever added, does each user need their own T212 account, or does T212 permit acting on behalf of others?

---

## FCA Perimeter

**Status: Unreviewed — human legal review required**

Running automated trading for **your own account** is generally outside FCA regulation in the UK. However:

- Running it for **other users** (executing trades on their behalf) likely constitutes discretionary investment management — a regulated activity under the Financial Services and Markets Act 2000.
- Adding multi-user capability without FCA authorisation carries **criminal liability**.

**Action required:** Before adding any second user, obtain a legal opinion on whether the platform falls within the FCA regulatory perimeter.

---

## UK GDPR

**Status: Partially addressed**

alphaKey's audit logs store IP addresses and email addresses, which are personal data under UK GDPR.

**Outstanding items:**

- [ ] Define a data retention policy and configure audit log auto-deletion (e.g., purge logs older than 90 days)
- [ ] Document the lawful basis for processing (likely Legitimate Interests for fraud prevention)
- [ ] Add a privacy notice accessible to users

---

## T212 Credentials — Production Configuration

**Status: Action required for production deployment**

alphaTrade has two credential sources, controlled by the `SECRETS_SOURCE` environment variable:

| Value | Behaviour |
|---|---|
| `db` (default) | Reads T212 keys from `.env` / pydantic-settings — **local dev only** |
| `alphakey` | Reads T212 keys from alphaKey's vault — **use in production** |

**Action required:** Set `SECRETS_SOURCE=alphakey` in the production environment (docker-compose override, Kubernetes secret, etc.). Never deploy with `SECRETS_SOURCE=db` in production — it requires a `.env` file with plaintext credentials.

---

*Last reviewed: 2026-06-13. Update when any item is actioned.*

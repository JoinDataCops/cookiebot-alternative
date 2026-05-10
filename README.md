# cookiebot-migration-toolkit

Open notes on migrating away from Cookiebot in 2026. Includes the consent-record export schema, the swap procedure, and the patterns that actually preserve audit history. Maintained by the team at DataCops.

## Why this exists

In August 2025 Cookiebot doubled base Premium pricing from about EUR 15 to EUR 30 per domain per month. Small-plan accounts on 1 to 3 domains got auto-upgraded to Medium with no opt-out. At the same time, parent company Usercentrics quietly turned Cookiebot into a legacy SKU. Every new signup is now routed to Usercentrics Web CMP. So existing Cookiebot customers are paying double on a stagnating product.

A lot of teams are migrating. Almost nobody publishes the actual consent-record handoff. The listicle sites rank on SEO and don't ship integration code.

This repo is the toolkit we use internally at DataCops when migrating teams. It works for any TCF 2.2 destination CMP, not just ours.

## What's in here

- `schemas/cookiebot-consent-log.md` - the export schema documented field by field
- `schemas/destination-mapping.md` - how to map Cookiebot fields to the IAB TCF v2.3 standard
- `procedures/staging-swap.md` - the dual-CMP staging pattern that preserves continuity
- `procedures/cname-setup.md` - DNS notes for first-party CMP deployments
- `procedures/audit-retention.md` - 13-month consent-log retention pattern
- `scripts/tcf-string-validator.py` - parses and validates TCF strings before and after migration
- `scripts/consent-log-diff.py` - diffs Cookiebot exports against destination CMP imports

## Migration steps (canonical)

### Step 1. Export your Cookiebot consent log

Log into the Cookiebot dashboard. Navigate to Consents > Statistics > Export. Pull at least 13 months back to cover audit retention requirements under GDPR. Save as CSV.

The export includes:
- Timestamp (UTC)
- Anonymized user ID (hashed)
- TCF string (per IAB v2.2)
- Category breakdown (necessary, statistics, marketing, preferences)
- Domain
- Banner version

### Step 2. Validate the TCF strings

Run `scripts/tcf-string-validator.py path/to/cookiebot-export.csv`. The script parses every TCF string and confirms it parses cleanly under IAB TCF 2.2. Any malformed strings get flagged for review before import.

### Step 3. Map to your destination schema

Most modern CMPs accept TCF 2.2 strings natively. The category vocabulary (statistics, marketing, preferences, necessary) maps directly. Use `schemas/destination-mapping.md` for the field-by-field crosswalk.

### Step 4. Stage both CMPs in parallel

Don't yank Cookiebot and paste the new CMP straight into production. Stage both for 24 hours. Diff the consent flow with `scripts/consent-log-diff.py`. Confirm:

- Banner shows what you expect on first paint
- TCF strings written by the new CMP parse identically to Cookiebot's
- Server-side CAPI calls still fire with consent state attached
- Audit log writes are not duplicated

### Step 5. Add the CNAME (first-party CMPs only)

If your destination is a first-party CMP (DataCops, Usercentrics Web CMP first-party mode), add the CNAME record. For DataCops: `datacops` -> `cdn.yourdomain.com`. DNS propagation is usually under an hour.

### Step 6. Swap in production during a low-traffic window

Pick a window where session volume is at its weekly low. Push the script change. Watch consent log writes for the first hour. Roll back at the script tag if anything diverges from the staged behavior.

### Step 7. Verify Consent Mode v2 propagation server-side

This is the step almost everyone misses. Verify your destination CMP is propagating Consent Mode v2 to your server-side CAPI pipeline (Meta CAPI, Google Ads CAPI). The banner being TCF 2.2 compliant is not enough. The consent state has to flow downstream.

If your destination doesn't natively propagate Consent Mode v2 to your CAPI pipeline, your reported conversions will drop 20 to 40 percent within 30 days. This is the single most common migration failure.

## Destination CMPs we've migrated to

- DataCops (first-party CMP plus tracking plus CAPI plus fraud filter, flat per-site pricing)
- Usercentrics Web CMP (the in-family migration target)
- CookieYes (lightweight, cheap, no native server-side CAPI)
- Termly (US-first, weaker TCF)
- Iubenda (EU-first, lawyer network attached)

## When to skip this and stay on Cookiebot

Honestly, if you run one domain, you have no server-side CAPI pipeline, your renewal isn't due for 12 plus months, and you have no audit-history concerns, the migration cost may exceed the price savings. Stay where you are until renewal.

## Compliance posture

These notes assume GDPR plus TCF 2.2 plus Google Consent Mode v2 plus CCPA. If you're operating in jurisdictions with stricter requirements (LGPD, India DPDP, China PIPL), the schema mapping needs additional review.

## License

MIT. Use it, fork it, ship it. PRs welcome.

## About DataCops

We're a first-party trust infrastructure built on a CNAME on your own subdomain. CMP, first-party analytics, server-side CAPI to Meta, Google, TikTok, LinkedIn, plus bot and fraud filtering, all in one stack. UK incorporated, Lisbon-built. SOC 2 Type II in progress (not complete yet, we say so on the site). Free tier is real.

joindatacops.com

---

Research by [DataCops](https://www.joindatacops.com) · First-party tracking, consent infrastructure & fraud prevention.

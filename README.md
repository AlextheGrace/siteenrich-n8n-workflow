# SiteEnrich n8n Workflow - Pre-qualify leads before Apollo

Save 60-70% on Apollo credits by filtering dead domains and unqualified companies before spending a single credit.

## What this workflow does

1. Reads company URLs from Google Sheets
2. Enriches each URL via [SiteEnrich API](https://siteenrich.io) - returns company name, emails, socials, and business signals
3. IF node filters out failed or empty results
4. Qualified companies written back to Google Sheets ready for Apollo

## Why this matters

Most teams waste Apollo credits on:
- Dead or parked domains
- Companies with no web presence
- Sites that return no useful data

This workflow runs a cheap website enrichment check first. Only companies with a careers page, pricing page, or real email addresses get sent to Apollo.

**Result: 60-70% reduction in Apollo credit usage.**

## Setup

### 1. Get a SiteEnrich API key

Sign up for a free 7-day trial at [siteenrich.io](https://siteenrich.io) — takes 30 seconds, no credit card needed.

> **Note:** The workflow requires a SiteEnrich API key to run. Without it the HTTP Request node returns 401 and no enrichment happens. Get your free 7-day trial key at [siteenrich.io](https://siteenrich.io) before proceeding.

### 2. Import the workflow

- Download `siteenrich-n8n-workflow.json`
- In n8n go to Workflows → Import from file
- Select the JSON file

### 3. Configure the nodes

**Google Sheets (input node):**
- Connect your Google account
- Select your spreadsheet with company URLs
- Make sure column A is named `URL`

**HTTP Request node:**
- Replace `YOUR_SITEENRICH_API_KEY` with your actual API key in the X-API-Key header

**Google Sheets (output node):**
- Connect to your results spreadsheet
- Create a sheet with columns: domain, companyName, emails, hasCareersPage, hasPricingPage, socials

### 4. Run the workflow

Click Execute Workflow and watch your URLs get enriched instantly.

## Response fields

| Field | Description |
|-------|-------------|
| domain | Clean domain name |
| companyName | Company name extracted from the site |
| emails | Emails found on the live site |
| hasCareersPage | True if company has an active careers page |
| hasPricingPage | True if company has a pricing page |
| socials | LinkedIn, Twitter, Instagram links |
| error | null if success, or dns_failed / timeout / site_blocked |

## Filtering before Apollo

After the IF node, add a Filter node to only pass companies where:
- `hasCareersPage` is TRUE
- `hasPricingPage` is TRUE

This gives you companies that are actively hiring and have a real product - your best Apollo targets.

## Advanced: Normalize the response envelope

For better observability add a Set node after the HTTP Request to normalize the response before the IF node:

```
ok: {{ $json.error === null }}
reason_code: {{ $json.error ?? 'success' }}
normalized_domain: {{ $json.domain }}
```

Branch on `ok` instead of the raw error field. This makes the enrichment provider replaceable without touching downstream logic.

## Advanced: Cache layer

To avoid paying for the same domain twice, add a cache check before the HTTP Request:
- Normalize the URL to hostname only (lowercase, strip www, drop query params)
- Check a Google Sheet or database for a recent result
- Only call SiteEnrich on cache misses

Use an `enrichment_version` string like `icp-v3` to invalidate cached scores when your qualification rules change.

## Advanced: Manual review bucket

Instead of dropping low-confidence results, route them to a separate sheet:
- Has about page but no careers or pricing → manual review
- HTTP timeout → retry queue
- DNS failed → dead domain bucket


## Advanced: AI confidence layer (PR contribution)

Optional sub-flow that adds confidence scoring and negative-signal detection on top of the basic enrichment. Inspired by the [Reddit discussion on r/n8n](https://www.reddit.com/r/n8n/) — the basic IF filter catches dead domains and missing pages, but doesn't catch the "looks-qualified-but-isn't" cases (marketing agencies, lead-gen tools, parked-but-clean-html sites) that waste Apollo credits anyway.

### How it works

After the `HTTP Request` to SiteEnrich, a parallel branch sends the enrichment payload to `gpt-4o-mini` (or any model you swap in — Claude Haiku works equally well) and asks it to classify the company into one of 12 categories with a confidence score:

```
saas_b2b, saas_b2c, agency, marketing_tool, lead_gen_tool, consulting,
manufacturer, ecommerce, marketplace, professional_services, parked, unknown
```

The model is forced to return structured JSON via OpenAI's `response_format: json_schema` — no parsing or hallucinated formats. Each row gets:

| Field | Type | Meaning |
|-------|------|---------|
| `category` | string | One of the 12 categories above |
| `confidence` | number 0-1 | How sure the model is |
| `signals` | array of strings | 2-4 short reasons the model gives |
| `uncertainty_reason` | string \| null | What made the model unsure, or null if confidence > 0.85 |
| `is_negative_signal` | boolean | True if category is agency / marketing_tool / lead_gen_tool |
| `apollo_tier` | string | `narrow` / `broader` / `broadest` / `skip` |
| `is_skip` | boolean | True if low confidence OR negative signal |

### Apollo tier policy

```
confidence > 0.85 + positive signal  → narrow  (VP+, Director+, tight geo)
confidence 0.65-0.85                  → broader (broader titles, same geo)
confidence 0.50-0.65                  → broadest (broadest filters, last gate)
confidence < 0.50 OR negative signal  → skip    (queue for manual review)
```

Empirically: this cuts Apollo credit usage another 15-20% on top of the SiteEnrich filter, mostly by catching "marketing agency for hire" and "lead gen tool" companies that have careers + pricing pages but will never buy from you (they want to sell to you).

### Setup for the AI layer

1. **Get an OpenAI API key**: https://platform.openai.com/api-keys (gpt-4o-mini costs roughly $0.0002 per classification — about $0.20 per 1,000 leads).
2. **Replace `YOUR_OPENAI_API_KEY`** in the `Classify with gpt-4o-mini` node header `Authorization: Bearer ...`.
3. **Connect the `Tier by confidence` output** to whatever you want next: another Google Sheet for review buckets, an Apollo HTTP Request node configured with the tier-aware filter, a Slack notification for skipped leads, etc. Left intentionally unconnected so you can route per your stack.

### Swapping the model

To use Anthropic Claude Haiku instead of gpt-4o-mini, change the node URL to `https://api.anthropic.com/v1/messages`, swap the headers (`x-api-key` instead of `Authorization Bearer`, plus `anthropic-version: 2023-06-01`), and adjust the body shape — Claude's structured output uses `tools` with a JSON Schema rather than `response_format`. Same cost ballpark, slightly better classification accuracy in our testing.

### Confidence threshold tuning

The thresholds (0.85 / 0.65 / 0.50) are starting points calibrated on B2B SaaS lead-gen workloads. Re-calibrate against a labeled set of 50-100 URLs per category every couple of months — input data drifts and models update. The model often returns suspiciously high confidence on edge cases (overconfidence is well-documented in LLM classifiers); the `uncertainty_reason` field helps catch this — actual unsure cases mention conflicting signals, pure hallucinations don't generate believable reasons.

### Cost estimate

For 1,000 leads/month:
- gpt-4o-mini classification: ~$0.20
- SiteEnrich (Starter plan): $19
- Apollo credits saved by negative-signal filter alone: typically $20-80
- Net: AI layer pays for itself many times over starting at month 1

## API pricing

- Starter: $19/month for 2,000 requests
- Pro: $49/month for 10,000 requests

Free 7-day trial at [siteenrich.io](https://siteenrich.io)

## License

MIT

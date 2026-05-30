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

Sign up for a free 7-day trial at [siteenrich.io](https://siteenrich.io)

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

## API pricing

- Starter: $19/month for 2,000 requests
- Pro: $49/month for 10,000 requests

Free 7-day trial at [siteenrich.io](https://siteenrich.io)

## License

MIT

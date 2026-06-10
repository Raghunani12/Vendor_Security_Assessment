# VendorRisk Intelligence

A browser-based ** Risk Management ** tool that performs passive, CORS-safe reconnaissance on vendor domains and generates structured risk assessments powered by the Google Gemini AI API.

---

## Overview

VendorRisk Intelligence collects publicly available telemetry about a vendor domain — DNS records, WHOIS/RDAP registration data, and Certificate Transparency logs — then routes that evidence to a Gemini LLM which scores the vendor across five risk dimensions and produces a structured JSON report rendered across six dashboard tabs.

No backend server required. Everything runs in a single HTML file in the browser.

---

## Features

- **Real DNS resolution** via Cloudflare's public DoH (DNS-over-HTTPS) endpoint
- **RDAP/WHOIS lookup** via `rdap.org` for registrar, registration date, and expiration
- **Certificate Transparency enumeration** via `crt.sh` for subdomain discovery
- **AI-powered risk scoring** using Google Gemini 2.5 Flash with strict JSON schema output
- **Six analysis tabs**: Overview, DNS Security, TLS/Headers, Compliance, Findings, AI Report
- **Scan terminal** with live per-module status indicators during assessment
- **Recent domains cache** persisted to `localStorage`
- **Fully client-side** — no server, no dependencies, no build step

---

## Architecture

```
Browser
│
├── DNS Module         → Cloudflare DoH API (cloudflare-dns.com)
│                        Records: A, MX, TXT (SPF), NS, _dmarc TXT
│
├── WHOIS Module       → RDAP API (rdap.org)
│                        Fields: registrar, registration date, expiry, domain age
│
├── Cert Transparency  → crt.sh JSON API
│                        Fields: total cert count, discovered subdomains
│
└── Gemini AI Engine   → Google Generative Language API
                         Model: gemini-2.5-flash
                         Input: structured evidence JSON
                         Output: scored risk report (strict JSON schema)
```

All three data collection modules run in parallel via `Promise.allSettled`. Gemini receives the aggregated evidence object and returns a fully structured report in a single call.

---

## Getting Started

### Prerequisites

- A modern browser (Chrome, Firefox, Edge, Safari)
- A **Google Gemini API key** (free tier available at [aistudio.google.com](https://aistudio.google.com))

### Running Locally

1. Download or clone the single `index.html` file
2. Open it directly in your browser — no web server needed
3. On first assessment, you will be prompted to enter your Gemini API key
4. The key is saved to `localStorage` so you only enter it once

### Deploying

Since this is a single static HTML file, it can be hosted anywhere:

```
GitHub Pages / Netlify / Vercel / S3 static site / Any CDN
```

To pre-embed your API key and skip the prompt, set the constant at the top of the `<script>` block:

```js
const DEPLOYED_GEMINI_KEY = 'your-key-here';
```

> ⚠️ Pre-embedding keys in client-side code exposes them publicly. Only do this in private/internal deployments.

---

## Usage

1. Enter a vendor domain (e.g. `stripe.com`, `atlassian.com`) in the search bar
2. Click **Assess →** or press Enter
3. Watch the scan terminal as each module completes
4. Review results across the six tabs once the report renders

Quick-start chips on the home screen let you run example assessments immediately.

---

## Report Tabs

| Tab | Contents |
|---|---|
| **Overview** | Composite risk score (0–100), five sub-scores, high/critical finding count, domain age, procurement recommendation |
| **DNS Security** | SPF status, DMARC policy and enforcement level, MX providers, discovered subdomains |
| **TLS / Headers** | Inferred SSL grade, security response headers (HSTS, CSP, X-Frame-Options, etc.) |
| **Compliance** | Signal detection for SOC 2, ISO 27001, GDPR, HIPAA, PCI-DSS, Trust Center presence |
| **Findings** | Individual findings with severity, category, detail, and evidence trace |
| **AI Report** | Executive summary, key risks, key strengths, and numbered remediation action items |

---

## Scoring Model

Gemini scores the vendor on five dimensions (0–100 each). The composite `vendor_score` maps to a risk tier:

| Score Range | Risk Level |
|---|---|
| 81 – 100 | Very Low |
| 61 – 80 | Low |
| 41 – 60 | Medium |
| 21 – 40 | High |
| 0 – 20 | Critical |

**Automatic penalties applied by the scoring prompt:**
- Missing DMARC → minimum −20 to Email Security score
- Missing SPF → minimum −15 to Email Security score
- Domain under 1 year old → `risk_flag: true` on WHOIS summary

---

## Data Sources

| Source | Endpoint | Data |
|---|---|---|
| Cloudflare DoH | `cloudflare-dns.com/dns-query` | A, MX, TXT, NS, DMARC records |
| RDAP | `rdap.org/domain/{domain}` | Registrar, dates, domain age |
| Cert Transparency | `crt.sh/?q=%.{domain}&output=json` | Certificate count, subdomains |
| Google Gemini | `generativelanguage.googleapis.com/v1beta/...` | Risk scoring and report generation |

All external calls are CORS-safe from the browser. No proxies or backend required.

---

## Limitations

- **No active scanning.** TLS grade and HTTP security headers are *inferred* by Gemini from DNS/WHOIS signals, not fetched live. Direct header inspection would require a server-side proxy.
- **CT log subdomain coverage is capped** at 50 recent certificate entries to keep response times acceptable.
- **RDAP availability varies** by TLD and registrar. Privacy-protected or restricted domains fall back to placeholder values.
- **Gemini outputs are probabilistic.** The system prompt instructs the model not to fabricate findings, but all AI-generated assessments should be validated against primary sources before use in procurement decisions.
- **API key is stored in `localStorage`** in plaintext. Do not use shared or public machines for assessments involving sensitive vendor lists.

---

## Local Storage Keys

| Key | Purpose |
|---|---|
| `vendorrisk_gemini_token` | Persisted Gemini API key |
| `vendorrisk_cache_history` | Last 5 assessed domains (for the Recent Cache sidebar) |

Clear both via browser DevTools → Application → Local Storage to reset.

---

## Customisation

**Add more example domains** — edit the chip array in the home screen HTML:
```html
<div class="chip" onclick="shortcutAssessment('yourvendor.com')">yourvendor.com</div>
```

**Change the AI model** — update the model string in `routeToGeminiCoreService`:
```js
const callRouteEndpoint = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=...`
```

**Adjust scoring thresholds** — modify the rules in `operationalSystemDirectives` inside `routeToGeminiCoreService` to tighten or relax risk penalties.

**Extend the compliance framework list** — add new boolean fields to `extractionJsonSchemaBluePrint.properties.compliance_signals` and update the `frameworks` array in `paintComprehensiveReportToTabs`.

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI | Vanilla HTML/CSS/JS — zero frameworks |
| Fonts | System sans-serif + Courier New monospace |
| DNS resolution | Cloudflare DoH (RFC 8484) |
| WHOIS | RDAP (RFC 7483) |
| Certificate logs | crt.sh public API |
| AI | Google Gemini 2.5 Flash (controlled generation) |
| Persistence | Browser `localStorage` |

---

## License

MIT — free to use, modify, and deploy. Attribution appreciated.

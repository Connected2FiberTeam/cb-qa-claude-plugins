# QA Testing Context

Domain knowledge for QA skills. Skills detect which repo they're running in and use the relevant platform sections.

---

## CPQ Test Architecture

### How Zephyr Scale Tests Work

Zephyr Scale test cases are **front-end UI-driven tests** executed through the CPQ (Configure, Price, Quote) application. Key implications:

- **Quote submission is a UI action** — the tester configures and submits a quote through the CPQ interface
- **Backend processing is invisible to the tester** — CPQ handles all communication with PricingAPI and the pricing layer internally; there is no manual API polling in Zephyr steps
- **Results appear in the CPQ UI** — the test verifies that pricing rows with MRC values are displayed in the quote, not that specific API responses were received
- **Do not write steps that call pricing APIs directly** — that is Karate territory, not Zephyr territory

### Test Layer Responsibilities

#### RP2 (RemotePricing2)

| Layer | Tool | What it tests | Who drives it |
|-------|------|--------------|---------------|
| UI end-to-end | Zephyr Scale (manual) | Full CPQ → PricingAPI → RP2 → D&A stack, as the user experiences it | QA tester via CPQ browser UI |
| API integration | Karate | RP2 API directly — request/response structure, timing, status codes | Automated, runs in CI |
| Component | Jest/Vitest | Individual service logic in isolation | Automated, runs in CI |

#### RP1 (RemotePricingAPI)

| Layer | Tool | What it tests | Who drives it |
|-------|------|--------------|---------------|
| UI end-to-end | Zephyr Scale (manual) | Full CPQ → PricingAPI → RP1 stack, as the user experiences it | QA tester via CPQ browser UI |
| API integration | Karate | RP1 API directly — `POST /api/v2/internal/remotePrice` request/response structure, status codes, returned services | Automated, runs against stage/UAT/prod |
| Unit | JUnit | Individual service logic in isolation | Automated, runs in CI |

### RP1 vs RP2 Async Architecture

Both RP1 and RP2 are **async** — a POST request enqueues work and returns immediately. The completion signal differs:

| Platform | POST returns | Completion signal | Karate validation |
|----------|-------------|-------------------|-------------------|
| RP2 | `requestId` + immediate status | Poll Response Builder HTTP API OR wait for webhook callback | Validates formatted `locations[].services[]` response from Response Builder |
| RP1 | `requestId` + "In progress" | Poll MongoDB `rp1_request_response` collection | Validates raw supplier API response docs in MongoDB |

---

## Writing Zephyr Test Cases

### What Belongs in Steps vs. Notes

- **Zephyr steps** — actions taken in the CPQ UI and what the tester observes on screen
- **Datadog log verification** — always a numbered test step with an expected result, never a precondition. If a log must be checked before proceeding (e.g. confirming a service started with correct config), it is Step 1, not a precondition.
- **Post-test Datadog verification** — log checks done after the test completes to confirm backend behaviour. If the check covers a requirement, it must be a numbered test step with an expected result — not a note. Only truly supplementary checks with no requirement traceability may be noted separately.
- **MongoDB queries** — valid verification pathway; include as numbered test steps. Navigation and connection steps (e.g. "open MongoDB", "connect to environment") are valid steps even without a direct assertion — they are necessary prerequisites to the verification query that follows. Steps with actual queries must include a specific expected result.
- **Fault injection / error simulation** — developer scope only; never include as Zephyr steps

### Keeping Step Content Clean

Step descriptions and expected results must be clean, tester-facing content only. **Never embed analyst notes inside step fields.** The following must not appear inside a step:

- Coverage traceability (e.g. "NOT directly covers R1", "Covers R2")
- Draft commentary or rationale (e.g. "this verifies the fix", "added for completeness")
- Parenthetical reminders to the analyst

If any of these need to be communicated, place them in a separate "Notes" or "Issues" section after the test case block. The step content must read as a clean, actionable instruction a tester can follow without interpretation.

---

## Call-To-Tests (CTTs)

Before writing steps in any new test case, check whether any of the following can replace inline steps. Reference them as a Call-To-Test step rather than spelling out the steps manually.

### CTT Catalog

| Key | Name | Use when |
|-----|------|----------|
| QTO-T6 | [CPQ] Call To Test - Simple Quote Setup (new account and deal) | Use when a test requires navigating away from the quote, submitting multiple quotes, or revisiting results |
| QTO-T7 | [CPQ] Call To Test - Simple Quick Quote Setup | Use only when results are verified immediately on the same page with no navigation away |
| QTO-T8 | [CPQ] Call To Test - Quote Configuration | Required in every test that submits a quote — details the quote configuration step |

### QTO-T6 vs QTO-T7: Choosing the Right Setup CTT

- **QTO-T7 (Quick Quote)** — backend processing completes normally. Results are visible on the current quote page. Navigating to external tools (MongoDB, Datadog) after submission is fine — these are separate applications and do not affect quote page state.
- **QTO-T6 (New Account and Deal)** — creates a persistent account and deal in CPQ. Results remain accessible after navigating away within CPQ, allowing the tester to leave the quote page, submit additional quotes, and return to check results.

**Decision rule:** Use QTO-T7 unless the test requires navigating away within CPQ itself (e.g., submitting a second quote and returning to compare both, or accessing a different CPQ section). Navigating to MongoDB or Datadog does NOT require QTO-T6 — those are external apps.

### CTT Step Order

The correct order for every test that submits a quote is:

1. **Call QTO-T6 or QTO-T7** — sets up the quote environment (account/deal or quick quote)
2. **Call QTO-T8** — configures the quote details (supplier, product, speed, term, media type, address)
3. **Submit the quote** — a discrete step in the test case with its own expected result

Never call QTO-T8 before QTO-T6/QTO-T7. Always include an explicit submit step after QTO-T8 — do not assume submission is implied by configuration.

### Required Quote Configuration Fields (QTO-T8)

Every QTO-T8 call must specify all of the following fields in the test data:

| Field | RP2 Notes | RP1 Notes |
|-------|-----------|-----------|
| Supplier | RP2 supplier name (e.g. Lyntia Quotation, Cityfibre) | RP1 supplier name as it appears in MongoDB `provider_credentials` (e.g. AT&T Wireline, Lumen) |
| Product | CPQ product name (e.g. Dedicated Internet, Broadband) | CPQ product name (e.g. Dedicated Internet, Ethernet - Switched) |
| Speed | As it appears in CPQ (e.g. ETHERNET 100M, 100/100) | As it appears in CPQ (e.g. 100, 1000 for DI; 100/100 for Broadband) |
| Term | Contract term in months (e.g. 12, 24, 36). Multiple products can have per-product terms. | Contract term in months (e.g. 12, 24, 36) |
| Media Type | Access medium (e.g. Fiber, Coax) | Access medium (e.g. Fiber, Copper) |
| Address | Full address including city, state, country, zip | Full address including city, state, country, zip |

Never omit Media Type — it is a required configuration field in CPQ.

### Multi-Quote Tests (submitting more than one quote)

When a test requires submitting multiple quotes and returning to check results on all of them:

- **Call QTO-T6 once** — one account and one deal covers all quotes in the test
- **Multiple quotes can be run under the same deal** — no need to create a new deal for each quote
- **Do NOT repeat QTO-T8** for subsequent quotes with identical configuration — add a step stating "run the same quote again under the same deal" (or similar)
- **Call QTO-T8 again only** if the subsequent quote has meaningfully different configuration

---

## Test Data Reference

### Platform Labels (at least one required per test case)

| Label | Platform |
|-------|----------|
| `RP2` | RemotePricing 2 (current system, includes LMX/DTAG adapters) |
| `RP1` | RemotePricing 1 (legacy Java Spring Boot) |
| `LMX` | LMX platform (separate legacy platform, still supported) |
| `UAP` | Unified API Platform |

### Speed Formats by Product and Test Layer

| Product | CPQ UI (Zephyr) | Karate (RP2) | Karate (RP1) |
|---------|-----------------|--------------|--------------|
| Dedicated Internet (all non-Broadband) | Numeric only — `10`, `100`, `1000` | `ETHERNET {value}M` e.g. `ETHERNET 100M` | `ETHERNET {value}M` e.g. `ETHERNET 100M` |
| Broadband | Upload/download split — `100/100`, `50/10` | Upload/download split — `100/100` | Upload/download split — `100/100` |

When writing Zephyr test steps or QTO-T8 test data, always use the CPQ UI format column.

### Valid Products

- Dedicated Internet
- Broadband
- Ethernet - Switched
- Ethernet - Dedicated
- E-Line
- E-Access

### Valid Access Mediums

- Fiber
- Copper
- Wireless
- Coax

### Address Requirements

**Minimum required fields:** street address, city, country, postal code

**Recommended additional fields:** state/region, latitude, longitude, county

All fields must be explicitly stated in test data — never use placeholder language like "a valid address."

### Environment-Agnostic Data Standard

Tests should be written to run on any environment (Stage, UAT, Prod). When test data varies by environment, all values must be provided — not just one:

- **Good:** `Stage: instance 1609 | UAT: 6407 | Prod: 6415`
- **Bad:** `Use instance 1609` (Stage-only; breaks on UAT/Prod)

If a test only applies to specific environments, state that explicitly in the preconditions.

---

## RP2-Specific Reference

### Instance IDs (provider_credentials)

Instance IDs for the RP2 supplier entity are environment-specific:

| Environment | Instance ID |
|-------------|-------------|
| Stage | 1609 |
| UAT | 6407 |
| Prod | 6415 |

### LMX Adapter Instance IDs

| Environment | Instance ID |
|-------------|-------------|
| Stage | 1637 |
| UAT | 6125 |
| Prod | 6125 |

### RP2 Karate Test Paths

- Quote feature files: `tests/integration/karate/src/test/resources/features/quote/`
- Mapping feature files: `tests/integration/karate/src/test/resources/features/mappings/`
- Helpers: `tests/integration/karate/src/test/resources/helpers/`

---

## RP1-Specific Reference

### Instance IDs (provider_credentials)

| Environment | Instance ID |
|-------------|-------------|
| Stage | 1609 |
| UAT | 6407 |
| Prod | 6415 |

### RP1 Host IPs

| Environment | IP | Port |
|-------------|-----|------|
| Stage | 10.82.25.78 | 8085 |
| UAT | 192.168.189.9 | 8085 |
| Prod | 192.168.89.11 | 8085 |

### RP1 Karate Architecture Note

RP1 Karate tests POST to `/api/v2/internal/remotePrice` and poll MongoDB `rp1_request_response` for results. The collection stores raw supplier API response documents — one per supplier API call. Karate validates HTTP status codes and error fields from these documents, not the RP1-formatted response.

### RP1 Karate Test Paths

- Quote feature files: `tests/integration/karate/src/test/resources/features/quote/`
- Helpers: `tests/integration/karate/src/test/resources/helpers/`
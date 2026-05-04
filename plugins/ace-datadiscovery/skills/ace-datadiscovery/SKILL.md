---
name: ace-datadiscovery
description: Search historical pricing databases to find addresses and configurations that returned valid pricing results. Uses MongoDB for RP2/RP1 and PostgreSQL for UAP. Auto-detects platform from repo, then walks through provider/product/term/speed/medium config, searches the correct DB, presents the first valid result, and offers to add it as a permanent Karate test scenario.
allowed-tools: Read, Grep, Glob, Bash, Write, AskUserQuestion
---

# ACE Data Discovery

Finds a historically-validated address + configuration pair for a given provider by searching the appropriate database. Stops at the first result that returned real pricing (non-empty services or solutions).

## When to use

- Find a working address for a provider before writing a permanent test scenario
- Identify what address/config worked historically when a test starts failing
- Bootstrap new test coverage from real traffic data

---

## Step 0 — Repo Detection

**Run first.** Determine which platform this repo serves:

```bash
ls src/test/resources/helpers/wait_for_uap_result.js 2>/dev/null   # UAP
ls tests/integration/karate/src/test/resources/data/templates/common/rp2_base_template.json 2>/dev/null  # RP2
ls tests/integration/karate/src/test/resources/data/templates/common/rp1_base_template.json 2>/dev/null  # RP1
```

Set `MODE` = `UAP`, `RP2`, or `RP1`. If none match, ask the user.

---

## Step 1 — Collect Quote Configuration

Ask using AskUserQuestion:

**Question** (header: "Provider")
- Question: "Which provider are you looking for?"
- Let the user type the provider name.
  - RP2/RP1: matches `supplier` name in MongoDB (e.g., `Lumen`, `AT&T Wireline`, `Zayo`)
  - UAP: matches `solution_provider_id` in PostgreSQL (e.g., `lumen`, `zayo`, `windstream`)

**Question** (header: "Product")
- Question: "Which product?"
- Suggest common values based on mode:
  - UAP: `DIA (IpService)`, `Ethernet Switched (EAccess)`, `Ethernet Dedicated P2P (ELine)`, `Broadband`
  - RP2/RP1: `Dedicated Internet`, `Ethernet - Switched`, `Ethernet - Dedicated`, `Broadband`

**Question** (header: "Speed")
- Question: "Which bandwidth in Mbps? (leave blank to search any speed)"

**Question** (header: "Term")
- Question: "Which contract term in months? (leave blank to search any term)"

**Question** (header: "Medium")
- Question: "Which access medium? (leave blank to search any medium)"
- Common values: `Fiber`, `Copper`, `Coax`, `Wireless`

**Question** (header: "Environment")
- Question: "Which environment's database?"
- Options: `stage` (default), `uat`, `prod`

---

## Step 2 — API Attribute Lookup

Find the provider's existing feature file:

**UAP:** `src/test/resources/features/quote/<Provider>/<Provider>_Quote.feature`
**RP2/RP1:** `tests/integration/karate/src/test/resources/features/quote/<Provider>/<Provider>_Quote.feature`

Directory names use title case with underscores. If not at the obvious path, grep:
```bash
grep -rl "<provider name>" src/test/resources/features/quote/        # UAP
grep -rl "<provider name>" tests/integration/karate/src/test/resources/features/quote/   # RP2
```

Read the `Background:` section and check for `suppliersApiRequestAttributes`.

- **Not present** → no API attributes needed, skip this step.
- **Present** → extract the parameter names and ask using AskUserQuestion whether the user wants to filter by specific API attribute values (optional — leave blank to skip this filter).

For UAP, also note the `resultConfig.providerFilter` value — this is the exact SP name used in `solution_provider_id` searches.

---

## Step 3 — Database Search

### RP2 — MongoDB

**Connection string by environment:**
- Stage: use `database-operations` plugin or `mongosh` with stage MongoDB connection
- UAT: use `database-operations` plugin or `mongosh` with UAT MongoDB connection
- Prod: use `database-operations` plugin or `mongosh` with prod MongoDB connection

Use the `database-operations` plugin's MongoDB query capability if available. Otherwise run via:
```bash
mongosh "<CONNECTION_STRING>" --eval '<query>'
```

#### Step 3a — Resolve provider → provider_id

```javascript
db.getCollection('remote_price_supplier').findOne(
  { supplier: { $regex: '<PROVIDER>', $options: 'i' } },
  { _id: 0, supplier: 1, provider_id: 1, entity_id: 1 }
)
```

Note the `provider_id` value.

#### Step 3b — Search original_request for matching config

Build filter from the user's inputs. All fields except `provider_id` are optional — omit if the user left them blank:

```javascript
db.getCollection('original_request').find(
  {
    suppliers: { $elemMatch: { $regex: '<PROVIDER>', $options: 'i' } },
    // include if product specified:
    'pricingData.products': { $elemMatch: { $regex: '<PRODUCT>', $options: 'i' } },
    // include if speed specified:
    'pricingData.speeds': '<SPEED>M',   // e.g. 'ETHERNET 100M'
    // include if term specified:
    'pricingData.terms': <TERM>,
    // include if medium specified:
    'pricingData.accessMediums': { $elemMatch: { $regex: '<MEDIUM>', $options: 'i' } },
    'feedback.status': 200
  },
  {
    _id: 0,
    requestId: 1,
    'locations.address': 1,
    'locations.city': 1,
    'locations.state': 1,
    'locations.postal': 1,
    'locations.country': 1,
    'pricingData.products': 1,
    'pricingData.speeds': 1,
    'pricingData.terms': 1,
    'pricingData.accessMediums': 1,
    inserted_at: 1
  }
).sort({ inserted_at: -1 }).limit(50)
```

For RP2 speed format: DIA/Ethernet speeds are stored as `ETHERNET {value}M` (e.g., `ETHERNET 100M`). Broadband speeds use upload/download split format (e.g., `100/100`).

#### Step 3c — Check request_response for valid pricing

For each `requestId` returned in Step 3b, check if the response has non-empty services:

```javascript
db.getCollection('request_response').findOne(
  {
    request_id: '<REQUEST_ID>',
    provider_id: <PROVIDER_ID>,
    status: 200,
    'final_response.locations': {
      $elemMatch: {
        services: { $exists: true, $not: { $size: 0 } }
      }
    }
  },
  {
    _id: 0,
    request_id: 1,
    provider_id: 1,
    status: 1,
    'final_response.locations.address': 1,
    'final_response.locations.city': 1,
    'final_response.locations.services.supplierProductId': 1,
    'final_response.locations.services.mrc': 1,
    'final_response.locations.services.nrc': 1,
    'final_response.locations.services.accessMedium': 1,
    'final_response.locations.services.term': 1
  }
)
```

Stop at the first `requestId` that returns a non-null result with services. Combine address from `original_request` with pricing from `request_response`.

**"Has pricing" definition:**
- `final_response.locations[].services[]` is non-empty (at least one service returned)
- `status` = 200

**"No coverage" / skip:**
- `status` ≠ 200
- `services[]` is empty
- `message` = `"Quotes not found."`

#### RP2 — Efficient combined query (alternative)

If the DB tool supports multi-collection queries, use a lookup approach to find the first request that has both a matching config AND a valid response in a single pass. Start from `request_response` where status=200 and services non-empty, then join to `original_request` to validate the config filters. This avoids iterating one request at a time.

---

### RP1 — Note

RP1 does not have a separate MongoDB collection for pricing results. RP1 responses are stored in `da_request_response.rp1_response` or in Elasticsearch. Ask the user if they want to search `da_request_response` for RP1 results, or switch to RP2 mode.

---

### UAP — PostgreSQL

**Connection:** use the `database-operations` plugin's PostgreSQL capability, or `psql` with the UAP DB connection string for the target environment.

#### Step 3a — Find solution_provider_engagement_id for provider

```sql
SELECT id, solution_provider_id
FROM solution_provider_engagement
WHERE solution_provider_id ILIKE '%<PROVIDER>%'
LIMIT 5;
```

Confirm the correct `solution_provider_id` with the user if multiple match.

#### Step 3b — Find engagements with valid pricing

```sql
SELECT
  se.engagement_id,
  se.request_body,
  se.response_body,
  sper.created_at
FROM supplier_engagement se
JOIN solution_provider_engagement spe ON spe.id = se.solution_provider_engagement_id
JOIN solution_provider_engagement_result sper ON sper.solution_provider_engagement_id = spe.id
JOIN supplier_product_result spr ON spr.solution_provider_engagement_result_id = sper.id
JOIN solution s ON s.supplier_product_result_id = spr.id
WHERE spe.solution_provider_id ILIKE '%<PROVIDER>%'
  -- include if term specified:
  AND s.contract_term_months = <TERM>
  -- include if speed specified (bandwidth_in_mbps column):
  -- AND spr.bandwidth_in_mbps = <SPEED>
ORDER BY sper.created_at DESC
LIMIT 20;
```

#### Step 3c — Retrieve full engagement via HTTP

For each `engagement_id` returned, call the UAP query service to get the full request with address:

**Stage:** `https://uap-query.stage.apps.connectbase.dev/engagement/<engagement_id>`
**UAT:** `https://uap-query.uat.apps.connectbase.dev/engagement/<engagement_id>`
**Prod:** `https://uap-query.apps.connectbase.dev/engagement/<engagement_id>`

```bash
curl -s "https://uap-query.<env>.apps.connectbase.dev/engagement/<engagement_id>"
```

Parse the response to extract the address from `requirements[0].serviceLocation` (or `sites[0]` for ELine).

Stop at the first engagement that returns a valid result with at least one solution.

---

## Step 4 — Present Results

### If a valid result was found:

**RP2/RP1:**
```
✓ VALID RESULT FOUND (RP2 / Stage)

  Provider:    <PROVIDER>
  Product:     <PRODUCT>
  Speed:       <SPEED>
  Term:        <TERM>mo
  Medium:      <MEDIUM>

  Address:
    <address>, <city>, <state> <postal>, <country>

  Request ID:  <REQUEST_ID>
  Date:        <inserted_at>

  Pricing (first service):
    supplierProductId: <VALUE>
    MRC:  <VALUE>
    NRC:  <VALUE>
    Access Medium: <VALUE>
```

**UAP:**
```
✓ VALID RESULT FOUND (UAP / Stage)

  Provider:    <PROVIDER>
  Product:     <PRODUCT> (<@type>)
  Speed:       <SPEED>M
  Term:        <TERM>mo

  Address:
    streetNr:        <VALUE>
    streetName:      <VALUE>
    streetType:      <VALUE>
    city:            <VALUE>
    stateOrProvince: <VALUE>
    postcode:        <VALUE>
    countryIso3:     <VALUE>

  Engagement ID: <ENGAGEMENT_ID>
  Date:          <created_at>

  Pricing (first solution):
    supplierProductId: <VALUE>
    MRC:  <VALUE>
    NRC:  <VALUE>
    Currency: <VALUE>
```

Then ask using AskUserQuestion:

**Question** (header: "Next Step")
- Question: "Would you like to add this as a permanent test scenario?"
- Options:
  - "Yes — add as a permanent test scenario" (description: "Invokes qa-addquote with this address and config")
  - "No — just show me the data" (description: "Done")
  - "Show full raw response first" (description: "Display complete DB document before deciding")

If the user selects "Yes", hand off to `qa-addquote` with:
- The discovered address (formatted per platform)
- Provider, product, speed, term, medium
- The supplier product ID from the pricing result as the expected assertion value
- MRC assertion: `mrc!=0`

### If no valid result was found after searching:

```
✗ NO VALID RESULT FOUND

  Searched <N> requests for <PROVIDER> / <PRODUCT> / <SPEED> / <TERM>mo.
  All returned no-coverage or error status.

  Suggestions:
  - Try a different product or speed combination
  - Run ace-datavalidate to probe live addresses
  - Check if the provider is currently healthy
```

---

## Error Handling

| Symptom | Likely cause |
|---------|-------------|
| No documents in `original_request` | Provider name mismatch — try regex or partial match |
| `request_response` has no valid services | Provider had no coverage at tested addresses historically |
| `remote_price_supplier` returns null | Provider not registered in this environment's DB |
| PostgreSQL connection refused | VPN not connected or wrong host for this environment |
| UAP query service 404 on engagement ID | Engagement too old or purged from query service |
| RP1 mode selected | No direct collection — suggest searching `da_request_response` or switching to RP2 |

---

## User Request

$ARGUMENTS

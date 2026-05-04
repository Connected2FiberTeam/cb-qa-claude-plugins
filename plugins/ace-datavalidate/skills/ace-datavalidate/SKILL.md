---
name: ace-datavalidate
description: Validate which addresses and quote configurations return successful pricing results. Accepts multiple addresses and configurations, generates all combinations, batches them (max 10 per Gradle run), and stops as soon as a valid result is found. Auto-detects RP2/RP1 or UAP repo and adapts accordingly.
allowed-tools: Read, Grep, Glob, Bash, Write, AskUserQuestion
---

# ACE Data Validate

Finds the first address + configuration combination that returns a successful quote result. Runs live Karate requests in batches of up to 10, stopping immediately when a valid result is found.

## When to use

- Find a working address for a provider before writing a permanent test scenario
- Validate which of a set of addresses are serviceable for a given configuration
- Quickly identify what configuration (product/speed/term) works at a given address
- Diagnose why a known-good address stopped returning quotes

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

## Step 1 — Collect Addresses

Ask using AskUserQuestion:

**Question** (header: "Addresses")
- Question: "How many addresses do you want to validate?"
- Options:
  - "One address" (description: "Single location")
  - "Multiple addresses" (description: "I'll provide a list")

Collect address details for each. Format depends on mode:

**UAP address fields:**

| Field | Required | Notes |
|-------|----------|-------|
| `streetNr` | Yes | Street number only (e.g., `"50"`) |
| `streetName` | Yes | Name only, no type suffix (e.g., `"Fremont"`) |
| `streetType` | No | e.g., `"St"`, `"Ave"`, `"Rd"` |
| `postcode` | Yes | Postal/ZIP code |
| `stateOrProvince` | Yes | 2-letter for US/CA; region name for international |
| `countryIso3` | Yes | 3-letter ISO code (e.g., `"USA"`, `"GBR"`) |
| `city` | Yes | |
| `lat` / `lon` | No | Include if known |

**RP2/RP1 address fields:**

| Field | Required | Notes |
|-------|----------|-------|
| `address` | Yes | Full street address string |
| `city` | Yes | |
| `state` | Yes | 2-letter for US/CA; region name for international |
| `country` | Yes | 3-letter ISO code |
| `zipCode` | Yes | Postal/ZIP code |
| `latitude` / `longitude` | No | Include if known |

Assign each address a short key (e.g., `ADDR_1`, `ADDR_2`) for reference in output.

---

## Step 2 — Collect Quote Configuration

Ask using AskUserQuestion:

**Question** (header: "Provider")
- Question: "Which provider are you testing?"
- Let user type the provider name. For UAP, this is the `providerFilter` string; for RP2/RP1, it is the exact `provider_credentials.provider_name`.

**Question** (header: "Product")
- Question: "Which product(s)? You can specify one or multiple (separate with commas)."
- Suggest common values based on mode:
  - UAP: `DIA (IpService)`, `Ethernet Switched (EAccess)`, `Ethernet Dedicated P2P (ELine)`, `Broadband`
  - RP2/RP1: `Dedicated Internet`, `Ethernet - Switched`, `Ethernet - Dedicated`, `Broadband`

**Question** (header: "Speed")
- Question: "Which bandwidth(s) in Mbps? You can specify one or multiple (e.g., 100, 1000)."

**Question** (header: "Term")
- Question: "Which contract term(s) in months? You can specify one or multiple (e.g., 12, 24, 36)."

**Question** (header: "Environment")
- Question: "Which environment?"
- Options: `stage` (default), `uat`, `prod`

**Question** (header: "Assertions")
- Question: "Any specific assertions to validate beyond 'quote returned'?"
- Options:
  - "No — just confirm a quote is returned" (description: "Uses `mrc!=0; nrc!=0`")
  - "Yes — I want to check specific values" (description: "e.g., currencyIso3=USD; supplierProductId=LUMEN_DIA")

---

## Step 3 — API Attribute Lookup

Find the provider's existing feature file:

**UAP:** `src/test/resources/features/quote/<Provider>/<Provider>_Quote.feature`
**RP2/RP1:** `tests/integration/karate/src/test/resources/features/quote/<Provider>/<Provider>_Quote.feature`

Directory names use title case with underscores. If not at the obvious path, grep for the provider name:
```bash
grep -rl "<provider name>" src/test/resources/features/quote/   # UAP
grep -rl "<provider name>" tests/integration/karate/src/test/resources/features/quote/   # RP2
```

Read the `Background:` section and check for `suppliersApiRequestAttributes`.

- **Not present**: no API attributes needed — skip this step.
- **Present with fixed values** (e.g., Resolute Nexus): use as-is.
- **Present with configurable fields**: extract defaults, ask the user if they want to override.
- **Present as a helper function**: extract the parameter names and ask the user.
- **Product-dependent**: apply the same conditional logic as the feature file.

For UAP, also check the `resultConfig.providerFilter` value and note it — use this exact string in all generated scenarios.

If no feature file exists for this provider, proceed without API attributes and note it in the summary.

---

## Step 4 — Generate Combinations

Compute the cartesian product:

```
combinations = addresses × products × speeds × terms
```

Assign each combination a label: `COMBO-{N}` (e.g., `COMBO-1`, `COMBO-2`, …).

Log the full combination matrix in a compact table so the user can see what will be tested:

```
COMBO-1:  ADDR_1 | DIA | 100M | 12mo
COMBO-2:  ADDR_1 | DIA | 100M | 24mo
COMBO-3:  ADDR_2 | DIA | 100M | 12mo
...
```

If total combinations > 50: warn the user and ask whether to proceed or trim the list.

For **ELine (P2P) only** (UAP): a single combination requires two addresses (A-side + Z-side). Ask the user to confirm which pairs to test; only generate pairs explicitly specified.

---

## Step 5 — Map Products to UAP `@type` *(UAP mode only)*

Convert user-supplied product names to UAP request types:

| Product name (any variant) | `@type` | `productType` |
|---------------------------|---------|---------------|
| DIA / Dedicated Internet / IP Services / IpService | `IpService` | `IP_SERVICES` |
| Ethernet Switched / EAccess / Ethernet Access | `EAccess` | *(omit)* |
| Ethernet Dedicated / ELine / P2P / Point-to-Point | `ELine` | *(omit)* |
| Broadband | `Broadband` | `BROADBAND` |

If a product cannot be mapped unambiguously, ask the user.

---

## Step 6 — Batch Execution

Group combinations into batches of **maximum 10**. Process batches in order. **Stop immediately when any combination returns `success`** — do not start the next batch.

For each batch:

### 6a — Generate Run IDs

```bash
TIMESTAMP=$(date +%s)
BATCH_TAG="validate-${TIMESTAMP}-batch-${BATCH_NUM}"
FEATURE_DIR="<karate-root>/src/test/resources/features/quote/AdHoc"   # adjust path per mode
FEATURE_FILE="${FEATURE_DIR}/${BATCH_TAG}.feature"
mkdir -p "$FEATURE_DIR"
```

Where `<karate-root>` is:
- UAP: repo root (`src/test/resources/…`)
- RP2/RP1: `tests/integration/karate` (`tests/integration/karate/src/test/resources/…`)

### 6b — Write the Batch Feature File

Write one Karate feature file containing one `Scenario` per combination in this batch. Each scenario:
- Has a unique tag `@<BATCH_TAG>`
- Writes its result to `/tmp/${BATCH_TAG}-${COMBO_N}.txt`
- **Does NOT call `karate.fail()`** — always writes the result file so all scenarios in the batch run to completion

**UAP scenario template (per combination):**

```gherkin
@<BATCH_TAG>
Feature: Data Validation Batch <BATCH_NUM>

  Background:
    * url restApiUrl
    * def queryServiceUrl = queryServiceUrl
    * def customer = testCustomer

  Scenario: <COMBO_N> - <ADDR_KEY> - <TYPE> - <BANDWIDTH>M - <TERM>mo
    * def clientRequestRef = '<BATCH_TAG>-<COMBO_N>'

    Given path 'requirement'
    And request
      """
      {
        "clientRequestReference": "#(clientRequestRef)",
        "customer": { "customerId": "#(customer.customerId)", "customerName": "#(customer.customerName)" },
        "requirements": [<REQUEST_BODY_FOR_TYPE>]
      }
      """
    When method POST
    Then status 200
    And match response.requirements[0].systemRequirementId == '#notnull'

    * def engagementId = response.requirements[0].systemRequirementId
    * def resultConfig = { engagementId: '#(engagementId)', queryUrl: '#(queryServiceUrl)', timeoutSeconds: <TIMEOUT>, testType: 'success', assertions: '<ASSERTIONS>', providerFilter: '<PROVIDER_FILTER>' }
    * def pollResult = call read('classpath:helpers/wait_for_uap_result.js') resultConfig

    * def FileWriter = Java.type('java.io.FileWriter')
    * def writer = new FileWriter('/tmp/<BATCH_TAG>-<COMBO_N>.txt')
    * writer.write('COMBO: <COMBO_N>\nADDR: <ADDR_KEY>\nCONFIG: <TYPE> <BANDWIDTH>M <TERM>mo\nSTATUS: ' + pollResult.status + '\nSOURCE: ' + pollResult.source + '\nENGAGEMENT_ID: ' + engagementId + '\n\n' + (pollResult.response || ''))
    * writer.close()
```

Use the appropriate `<REQUEST_BODY_FOR_TYPE>` block from ace-runquote Step 3 (IpService, EAccess, ELine, or Broadband). Substitute all real values — no placeholders remain in the written file.

**RP2/RP1 scenario template (per combination):**

```gherkin
@<BATCH_TAG>
Feature: Data Validation Batch <BATCH_NUM>

  Background:
    * url requestBuilderUrl

  Scenario: <COMBO_N> - <ADDR_KEY> - <PROVIDER> - <SPEED> - <TERM>mo
    * def config = {}
    * config.suppliers = '<PROVIDER>'
    * config.speeds = '<SPEED>'
    * config.terms = '<TERM>'
    * config.products = '<PRODUCT>'
    * config.accessMediums = '<MEDIUM>'
    * config.useCallbackMock = false
    * config.location = { address: '<ADDRESS>', city: '<CITY>', state: '<STATE>', country: '<COUNTRY>', zipCode: '<ZIPCODE>' }
    <API_ATTRS_BLOCK_IF_NEEDED>
    * def req = call read('classpath:helpers/build_request.js') config
    * karate.log('REQUEST <COMBO_N>:', karate.toJson(req))

    Given request req
    When method POST
    * match responseStatus == 200
    * def requestId = response.requestId

    * configure retry = { count: <RETRY_COUNT>, interval: 5000 }
    Given url responseBuilderUrl
    And param requestId = requestId
    When method GET
    And retry until responseStatus == 200 && (response.status == 'Completed' || response.status == 'Failed')

    * def hasServices = karate.jsonPath(response, '$.locations[*].services[*]').size() > 0
    * def resultStatus = response.status == 'Completed' && hasServices ? 'success' : (response.status == 'Completed' ? 'no-coverage' : 'failed')

    * def FileWriter = Java.type('java.io.FileWriter')
    * def writer = new FileWriter('/tmp/<BATCH_TAG>-<COMBO_N>.txt')
    * writer.write('COMBO: <COMBO_N>\nADDR: <ADDR_KEY>\nCONFIG: <PROVIDER> | <PRODUCT> | <SPEED> | <TERM>mo\nSTATUS: ' + resultStatus + '\n\n' + karate.toJson(response))
    * writer.close()
```

Set `<RETRY_COUNT>` = `timeout / 5` (default: 300s → 60 retries).

### 6c — Run the Batch

**UAP — Stage:**
```bash
./gradlew test '-Dkarate.options=--tags @<BATCH_TAG> --tags ~@bug --tags ~@uat --tags ~@prod' -Dkarate.env=stage
```

**UAP — UAT:**
```bash
./gradlew test '-Dkarate.options=--tags @<BATCH_TAG> --tags ~@bug --tags ~@stage --tags ~@prod' -Dkarate.env=uat
```

**RP2 — Stage:**
```bash
cd tests/integration/karate
./gradlew test '-Dkarate.options=--tags @<BATCH_TAG>' -Dkarate.mock.callback=false
```

**RP2 — UAT:**
```bash
cd tests/integration/karate
MYSQL_HOST=$MYSQL_HOST_UAT MYSQL_PASSWORD=$MYSQL_PASSWORD_UAT \
./gradlew test '-Dkarate.options=--tags @<BATCH_TAG>' \
  -Dkarate.env=uat -Dkarate.scenario.env=uat -Dkarate.mock.callback=false
```

### 6d — Check Batch Results

After Gradle completes, read each output file for this batch:

```bash
for f in /tmp/<BATCH_TAG>-COMBO-*.txt; do
  echo "=== $f ==="; head -5 "$f"; echo
done
```

Parse the `STATUS:` line from each file:
- `STATUS: success` → **winner found** — record this combination and proceed to Step 7
- `STATUS: no-coverage` → no quote at this address/config — continue
- `STATUS: failed` / file missing → error — log and continue

If any combination returned `success`, **do not run further batches**.

### 6e — Clean Up Batch Files

```bash
rm -f "$FEATURE_FILE"
rm -f /tmp/<BATCH_TAG>-COMBO-*.txt
rmdir "$FEATURE_DIR" 2>/dev/null || true
```

---

## Step 7 — Present Results

### If a winning combination was found:

```
✓ VALID RESULT FOUND

  Address:   <ADDR_KEY> — <full address details>
  Provider:  <PROVIDER>
  Product:   <PRODUCT> (<@type if UAP>)
  Bandwidth: <BANDWIDTH>M
  Term:      <TERM>mo
  
  Engagement/Request ID: <ID>
  Source: <query / event-hub / response-builder>
  
  [Key response values: supplierProductId, MRC, NRC, currency, access medium]
  
  Batches run: <N of M>   Combinations tested: <X of Y total>
```

Offer to:
- Show the full raw response
- Write this as a permanent test scenario (invoke `qa-addquote` steps for the winning combination)

### If no winning combination was found after all batches:

```
✗ NO VALID RESULT

  Tested <N> combinations across <B> batches.
  
  | Combo | Address   | Config              | Status       |
  |-------|-----------|---------------------|--------------|
  | 1     | ADDR_1    | DIA 100M 12mo       | no-coverage  |
  | 2     | ADDR_1    | DIA 100M 24mo       | no-coverage  |
  | 3     | ADDR_2    | DIA 100M 12mo       | failed       |
  ...
```

Suggest next steps: different provider, different product, verify VPN, check provider health.

---

## Error Handling

| Symptom | Likely cause |
|---------|-------------|
| HTTP 400 on `POST /requirement` | Invalid `@type`/`productType` combo or missing required field |
| HTTP 400 from RP2 request builder | Invalid speed/product/medium for this provider |
| All combos `no-coverage` | Provider has no coverage at any of the tested addresses |
| All combos `failed` / output files missing | Gradle failure before write — check console for assertion/exception |
| VPN not connected | Stage/UAT/prod unreachable |
| Timeout on all combos | Increase timeout or check provider health |

---

## User Request

$ARGUMENTS

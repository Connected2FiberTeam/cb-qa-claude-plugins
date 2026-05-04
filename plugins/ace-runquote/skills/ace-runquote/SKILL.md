---
name: ace-runquote
description: Run a live one-off quote against any ACE platform provider and return the full raw response for investigation or validation. Auto-detects the repo (RP2, RP1, or UAP) and uses the appropriate Karate mechanism. No standing test file is created.
allowed-tools: Read, Grep, Glob, Bash, Write
---

# ACE Run Quote

Executes a disposable Karate-based quote request — no standing test file is created. Supports RP2 (standard + LMX), RP1, and UAP platforms.

## When to use

- Reproduce a pricing issue with specific inputs
- Validate live provider behaviour without adding a permanent test
- Explore what a provider returns for a given address/speed/product combination
- Investigate engagement or response builder results interactively

Other skills that need to execute a live quote to investigate an issue should invoke this skill's steps directly.

---

## Step 0 — Repo Detection

**Run this before any other step.** Check which platform this repo serves:

```bash
# UAP
ls src/test/resources/helpers/wait_for_uap_result.js 2>/dev/null

# RP2
ls tests/integration/karate/src/test/resources/data/templates/common/rp2_base_template.json 2>/dev/null

# RP1
ls tests/integration/karate/src/test/resources/data/templates/common/rp1_base_template.json 2>/dev/null
```

| File found | Mode |
|------------|------|
| `wait_for_uap_result.js` | **UAP mode** — POST `/requirement` → poll query service |
| `rp2_base_template.json` | **RP2 mode** — POST request builder → poll response builder |
| `rp1_base_template.json` | **RP1 mode** — POST RP1 → poll MongoDB |

Proceed to the matching section below. If none found, ask the user which platform they're working in.

---

## RP2 Mode

### Inputs

Parse all inputs from `$ARGUMENTS` or conversational context. Ask only when something required is genuinely ambiguous.

#### Required

| Parameter | Notes |
|-----------|-------|
| `provider` | Exact name as stored in MongoDB `provider_credentials.provider_name` (e.g., `"CityFibre TMF-Eth"`) |
| `speed` | e.g., `"ETHERNET 100M"`, `"100/100"`, `"ALL"` |

#### Location — Standard single address

| Parameter | Notes |
|-----------|-------|
| `address` | Street address |
| `city` | |
| `state` | 2-letter abbreviation for US/CA; local district/region for international |
| `country` | 3-letter ISO code (e.g., `"USA"`, `"GBR"`, `"CHN"`) |
| `zipCode` | Postal/ZIP code |
| `latitude` / `longitude` | Optional — include if known; some providers require them |

#### Location — P2P (two addresses)

Provide the same location fields twice, prefixed `A` and `Z`. Detected automatically when the user provides two addresses.

#### Optional

| Parameter | Default | Notes |
|-----------|---------|-------|
| `product` | `"Ethernet - Switched"` | e.g., `"Dedicated Internet"`, `"Broadband"` |
| `medium` | `"Fiber"` | e.g., `"Wireless - Fixed"`, `"COAX/HFC"`, `"Copper"` |
| `term` | `"12"` | Months; comma-separated for multiple |
| `env` | `stage` | `stage`, `uat`, or `prod` |
| `timeout` | `300` | Seconds to wait for response |

#### Alternatives (optional)

| Parameter | Default | Options |
|-----------|---------|---------|
| `speedPolicy` | `EXACT` | `EXACT`, `NEXT_HIGHER`, `NEXT_LOWER`, `NEXT_HIGHER_AND_LOWER` |
| `termPolicy` | `EXACT` | `EXACT`, `NEXT_HIGHER`, `NEXT_LOWER`, `NEXT_HIGHER_AND_LOWER` |
| `altSpeedAlways` | `false` | `true` returns exact AND alternatives side-by-side |
| `altTermAlways` | `false` | Same for terms |

> Alternatives require feature flag `rp2.alternatives_resolution_TEMP_toggle` enabled in the target environment.

---

### RP2 Execution Steps

#### RP2 Step 1 — Resolve API request attributes

Find the provider's feature file:
```
tests/integration/karate/src/test/resources/features/quote/<Provider>/<Provider>_Quote.feature
```

Read the `Background:` section. Check whether `suppliersApiRequestAttributes` appears:

- **Not present**: skip — no attributes needed
- **Fixed structure** (e.g., Resolute Nexus `Port Bandwidth: "least-cost"`): use as-is
- **Configurable fields** (e.g., ESUN: IP Blocks, Last Mile Provider): extract names and defaults, ask user to confirm or override
- **Helper function** (e.g., Blue Wireless `buildBlueWirelessApiRequestAttributes`): read the function signature and ask for those parameters
- **Product-dependent**: apply the same conditional logic as the feature file

#### RP2 Step 2 — Generate run ID and paths

```bash
TIMESTAMP=$(date +%s)
ADHOC_TAG="rp2-adhoc-${TIMESTAMP}"
OUTPUT_FILE="/tmp/${ADHOC_TAG}-response.json"
FEATURE_DIR="tests/integration/karate/src/test/resources/features/quote/AdHoc"
FEATURE_FILE="${FEATURE_DIR}/${ADHOC_TAG}.feature"
mkdir -p "$FEATURE_DIR"
```

#### RP2 Step 3 — Write the feature file

Replace all `<PLACEHOLDER>` tokens with actual values before writing.

**Standard quote:**
```gherkin
@<ADHOC_TAG>
Feature: Ad-hoc Quote - <PROVIDER>

  Background:
    * url requestBuilderUrl

  Scenario: One-off quote
    * def config = {}
    * config.suppliers = '<PROVIDER>'
    * config.speeds = '<SPEED>'
    * config.terms = '<TERM>'
    * config.products = '<PRODUCT>'
    * config.accessMediums = '<MEDIUM>'
    * config.useCallbackMock = false
    * config.location = { address: '<ADDRESS>', city: '<CITY>', state: '<STATE>', country: '<COUNTRY>', zipCode: '<ZIPCODE>' }
```

**P2P quote:** replace `config.location` with:
```gherkin
    * config.locationA = { address: '<ADDRESS_A>', city: '<CITY_A>', state: '<STATE_A>', country: '<COUNTRY_A>', zipCode: '<ZIPCODE_A>' }
    * config.locationZ = { address: '<ADDRESS_Z>', city: '<CITY_Z>', state: '<STATE_Z>', country: '<COUNTRY_Z>', zipCode: '<ZIPCODE_Z>' }
```

**If API request attributes are needed**, add after location:
```gherkin
    * def apiAttrs =
    """
    <API_ATTRS_JSON>
    """
    * config.suppliersApiRequestAttributesJson = karate.toJson(apiAttrs)
```

**If alternatives are requested**, add:
```gherkin
    * config.alternativesPolicy = { speedPolicy: '<SPEED_POLICY>', termPolicy: '<TERM_POLICY>', altSpeedAlways: <ALT_SPEED_ALWAYS>, altTermAlways: <ALT_TERM_ALWAYS> }
```

**Append this closing block for all quote types:**
```gherkin
    * def req = call read('classpath:helpers/build_request.js') config
    * karate.log('REQUEST:', karate.toJson(req))

    Given request req
    When method POST
    * match responseStatus == 200
    * def requestId = response.requestId
    * karate.log('Request ID:', requestId)

    * configure retry = { count: <RETRY_COUNT>, interval: 5000 }
    Given url responseBuilderUrl
    And param requestId = requestId
    When method GET
    And retry until responseStatus == 200 && (response.status == 'Completed' || response.status == 'Failed')
    * def finalResponse = response
    * karate.log('FINAL RESPONSE:', karate.toJson(finalResponse))

    * def FileWriter = Java.type('java.io.FileWriter')
    * def writer = new FileWriter('<OUTPUT_FILE>')
    * writer.write(karate.toJson(finalResponse))
    * writer.close()
```

Set `<RETRY_COUNT>` = `timeout / 5` (e.g., 300s → 60 retries).

#### RP2 Step 4 — Run the test

**Stage (default):**
```bash
cd tests/integration/karate
./gradlew test '-Dkarate.options=--tags @<ADHOC_TAG>' -Dkarate.mock.callback=false
```

**UAT:**
```bash
cd tests/integration/karate
MYSQL_HOST=$MYSQL_HOST_UAT MYSQL_PASSWORD=$MYSQL_PASSWORD_UAT \
./gradlew test '-Dkarate.options=--tags @<ADHOC_TAG>' \
  -Dkarate.env=uat -Dkarate.scenario.env=uat -Dkarate.mock.callback=false
```

**Prod:**
```bash
cd tests/integration/karate
MYSQL_HOST=$MYSQL_HOST_PROD MYSQL_PASSWORD=$MYSQL_PASSWORD_PROD \
MONGODB_HOST=$MONGODB_HOST_PROD MONGODB_PASSWORD=$MONGODB_PASSWORD_PROD \
./gradlew test '-Dkarate.options=--tags @<ADHOC_TAG>' \
  -Dkarate.env=prod -Dkarate.scenario.env=prod -Dkarate.mock.callback=false
```

#### RP2 Step 5 — Present the response

Read `$OUTPUT_FILE` and call out:
- `status` — `Completed` / `Failed`
- `locations[].services[]` — each returned service: provider, product, speed, term, MRC, NRC, access medium
- `apiStatuses[]` — per-provider execution status and any error messages
- For alternatives: services with `matchType` showing fallback/supplement results

#### RP2 Step 6 — Clean up

```bash
rm -f "$FEATURE_FILE" "$OUTPUT_FILE"
rmdir "$FEATURE_DIR" 2>/dev/null || true
```

---

### RP2 Error Handling

| Symptom | Likely cause |
|---------|-------------|
| Gradle fails with entity lookup error | Provider name doesn't match MongoDB — check exact casing |
| HTTP 400 from request builder | Invalid speed/product/medium combination for this provider |
| `status: Failed`, empty `services[]` | No coverage at address — normal, not an error |
| Alternatives not returned | Feature flag `rp2.alternatives_resolution_TEMP_toggle` likely disabled |
| Test times out | Provider slow or unreachable — check VPN, check `apiStatuses` |
| Output file missing | Test failed before write step — inspect Gradle console output |

---

## UAP Mode

### Inputs

Parse all inputs from `$ARGUMENTS` or conversational context. Ask only when something required is genuinely ambiguous.

#### Required

| Parameter | Notes |
|-----------|-------|
| `type` | `IpService` (DIA), `EAccess` (Ethernet Switched), `ELine` (Ethernet Dedicated P2P), or `Broadband` |
| `bandwidth` | Integer Mbps (e.g., `100`, `1000`) |
| `termMonths` | Contract term in months (e.g., `12`, `24`, `36`) |

#### Location — Single site (IpService, EAccess, Broadband)

| Parameter | Notes |
|-----------|-------|
| `streetNr` | Street number |
| `streetName` | Street name only — no type suffix (e.g., `Fremont`, not `Fremont St`) |
| `streetType` | Optional — e.g., `St`, `Ave`, `Rd` |
| `postcode` | Postal/ZIP code |
| `stateOrProvince` | 2-letter abbreviation for US/CA; region name for international |
| `countryIso3` | 3-letter ISO country code (e.g., `USA`, `GBR`, `CHN`) |
| `city` | |
| `lat` / `lon` | Optional — include if known; Lumen uses them for geocoded lookups |

#### Location — P2P ELine (two sites)

Provide the same address fields twice, prefixed `A` and `Z`. Detected automatically when `type` is `ELine` or user provides two addresses.

#### Optional

| Parameter | Default | Notes |
|-----------|---------|-------|
| `providerFilter` | (none) | Exact SP name in engagement results (e.g., `"Lumen Quote v4"`) — waits specifically for that provider's result |
| `assertions` | `mrc!=0; nrc!=0` | Semicolon-separated assertion string |
| `testType` | `success` | `success` or `no-coverage` |
| `accessMedia` | `FIBRE` | For EAccess/ELine/Broadband: `FIBRE`, `COPPER`, `WIRELESS` |
| `env` | `stage` | `stage`, `uat`, or `prod` |
| `timeout` | `180` | Seconds to wait for result |

#### Assertion syntax

| Syntax | Meaning |
|--------|---------|
| `supplierProductId=LUMEN_DIA` | Exact match on supplier product ID |
| `mrc=433.5` | Exact MRC amount |
| `mrc!=0` / `nrc!=0` | Non-zero cost check |
| `currencyIso3=USD` | Currency check |
| `accessMedium=FIBRE` | Access medium check |
| `chipset=FTTP` | Chipset value check |

---

### UAP Execution Steps

#### UAP Step 1 — Look up provider details (optional)

If the user mentions a provider name, scan `src/test/resources/features/quote/` for a matching feature file. Read the existing `resultConfig` lines to find:
- The exact `providerFilter` string (e.g., `'Lumen Quote v4'`)
- Example assertions already in use (e.g., `supplierProductId=LUMEN_DIA; currencyIso3=USD`)

Suggest these as defaults. If no `providerFilter` was found or given, use an empty string `''` — polling accepts any provider's result.

#### UAP Step 2 — Generate run ID and paths

```bash
TIMESTAMP=$(date +%s)
ADHOC_TAG="uap-adhoc-${TIMESTAMP}"
OUTPUT_FILE="/tmp/${ADHOC_TAG}-response.txt"
FEATURE_DIR="src/test/resources/features/quote/AdHoc"
FEATURE_FILE="${FEATURE_DIR}/${ADHOC_TAG}.feature"
mkdir -p "$FEATURE_DIR"
```

#### UAP Step 3 — Write the feature file

Replace every `<PLACEHOLDER>` with a real value. Omit optional address fields (`streetType`, geo point) when not provided.

**IpService (DIA):**
```gherkin
@<ADHOC_TAG>
Feature: Ad-hoc UAP Quote

  Background:
    * url restApiUrl
    * def queryServiceUrl = queryServiceUrl
    * def customer = testCustomer

  Scenario: Ad-hoc IpService quote
    * def clientRequestRef = 'adhoc-<TIMESTAMP>'

    Given path 'requirement'
    And request
      """
      {
        "clientRequestReference": "#(clientRequestRef)",
        "customer": { "customerId": "#(customer.customerId)", "customerName": "#(customer.customerName)" },
        "requirements": [{
          "@type": "IpService",
          "productType": "IP_SERVICES",
          "clientRequirementReference": "adhoc-ipservice-<TIMESTAMP>",
          "processingPriority": "MEDIUM",
          "contractTermMonths": <TERM_MONTHS>,
          "site": {
            "clientSiteReference": "adhoc-site",
            "location": {
              "address": {
                "@type": "FieldedAddress",
                "streetNr": "<STREET_NR>",
                "streetName": "<STREET_NAME>",
                "streetType": "<STREET_TYPE>",
                "postcode": "<POSTCODE>",
                "stateOrProvince": "<STATE>",
                "countryIso3": "<COUNTRY_ISO3>",
                "city": "<CITY>"
              }<GEO_POINT_BLOCK>
            },
            "serviceConfiguration": {
              "accessBandwidth": { "unit": "MBPS", "value": <BANDWIDTH> },
              "serviceBandwidth": { "unit": "MBPS", "value": <BANDWIDTH> }
            }
          }
        }]
      }
      """
    When method POST
    Then status 200
    And match response.requirements[0].systemRequirementId == '#notnull'

    * def engagementId = response.requirements[0].systemRequirementId
    * karate.log('Ad-hoc UAP engagement ID:', engagementId)
    * def resultConfig = { engagementId: '#(engagementId)', queryUrl: '#(queryServiceUrl)', timeoutSeconds: <TIMEOUT>, testType: '<TEST_TYPE>', assertions: '<ASSERTIONS>', providerFilter: '<PROVIDER_FILTER>' }
    * def pollResult = call read('classpath:helpers/wait_for_uap_result.js') resultConfig
    * karate.log('Ad-hoc UAP result:', pollResult.status, '/', pollResult.source)

    * def FileWriter = Java.type('java.io.FileWriter')
    * def writer = new FileWriter('<OUTPUT_FILE>')
    * writer.write('ENGAGEMENT_ID: ' + engagementId + '\nSTATUS: ' + pollResult.status + '\nSOURCE: ' + pollResult.source + '\n\n' + (pollResult.response || '(no response body)'))
    * writer.close()

    * if (pollResult.status != '<TEST_TYPE>') karate.fail('Expected <TEST_TYPE> but got: ' + pollResult.status)
```

**EAccess (Ethernet Switched) — key differences from IpService:**
- `"@type": "EAccess"` — no `productType` field
- site `serviceConfiguration`: `accessBandwidth` + `"accessMedia": ["<ACCESS_MEDIA>"]`
- Add `"serviceBandwidth": { "unit": "MBPS", "value": <BANDWIDTH> }` at requirement level (outside `site`)

**ELine (Ethernet Dedicated P2P):**
```json
"@type": "ELine",
"backhaul": { "backhaulBandwidth": { "unit": "MBPS", "value": <BANDWIDTH> } },
"sites": [
  {
    "clientSiteReference": "adhoc-site-a",
    "location": { "address": { "@type": "FieldedAddress", <SITE_A_FIELDS> } },
    "serviceConfiguration": {
      "accessBandwidth": { "unit": "MBPS", "value": <BANDWIDTH> },
      "accessMedia": ["<ACCESS_MEDIA>"]
    }
  },
  {
    "clientSiteReference": "adhoc-site-z",
    "location": { "address": { "@type": "FieldedAddress", <SITE_Z_FIELDS> } },
    "serviceConfiguration": {
      "accessBandwidth": { "unit": "MBPS", "value": <BANDWIDTH> },
      "accessMedia": ["<ACCESS_MEDIA>"]
    }
  }
]
```

**Broadband:**
```json
"@type": "Broadband",
"productType": "BROADBAND",
"site": {
  ...,
  "serviceConfiguration": {
    "accessMedia": ["<ACCESS_MEDIA>"],
    "serviceBandwidth": { "unit": "MBPS", "value": <BANDWIDTH> },
    "accessBandwidth": {
      "downloadBandwidth": { "unit": "MBPS", "value": <BANDWIDTH> },
      "uploadBandwidth": { "unit": "MBPS", "value": <UPLOAD_BW> }
    }
  }
}
```

**Geo point block** — insert inside `location`, after `address`, only when `lat`/`lon` are provided:
```json
,
"geographicPoint": {
  "spatialReferenceSystem": "WGS84",
  "x": "<LAT>",
  "y": "<LON>"
}
```

#### UAP Step 4 — Run the test

Run from the repo root.

**Stage (default):**
```bash
./gradlew test '-Dkarate.options=--tags @<ADHOC_TAG> --tags ~@bug --tags ~@uat --tags ~@prod' -Dkarate.env=stage
```

**UAT:**
```bash
./gradlew test '-Dkarate.options=--tags @<ADHOC_TAG> --tags ~@bug --tags ~@stage --tags ~@prod' -Dkarate.env=uat
```

**Prod:**
```bash
./gradlew test '-Dkarate.options=--tags @<ADHOC_TAG> --tags ~@bug --tags ~@stage --tags ~@uat' -Dkarate.env=prod
```

#### UAP Step 5 — Present the response

Read `$OUTPUT_FILE` and show:
- `ENGAGEMENT_ID` — useful for follow-up queries against the query service directly
- `STATUS` (`success` / `no-coverage`) and `SOURCE` (`query` / `redis-result-callback` / `event-hub`)
- Key values parsed from the response body: `supplierProductId`, MRC/NRC amounts, `currencyIso3`, `accessMedium`
- Full raw body (truncated if very large — offer to show more on request)

Also provide the direct query URL for manual follow-up:
```
GET <queryServiceUrl>/engagement/<ENGAGEMENT_ID>
```

Query service URLs by environment:
- Stage: `https://uap-query.stage.apps.connectbase.dev`
- UAT: `https://uap-query.uat.apps.connectbase.dev`
- Prod: `https://uap-query.apps.connectbase.dev`

#### UAP Step 6 — Clean up

```bash
rm -f "$FEATURE_FILE" "$OUTPUT_FILE"
rmdir "$FEATURE_DIR" 2>/dev/null || true
```

---

### UAP Error Handling

| Symptom | Likely cause |
|---------|-------------|
| HTTP 400 from `POST /requirement` | Invalid request shape — check `@type`/`productType` combination and required fields |
| `Expected success but got: no-coverage` | No coverage at this address for the provider |
| Test times out | Provider didn't respond — increase `timeout` or check provider health in the query service |
| Output file missing | Test failed before write — inspect Gradle console output |
| Assertion failure | Expected value not in response — check `assertions` string matches what this provider returns |
| VPN not connected | Stage/UAT/prod unreachable from local machine |

---

## User Request

$ARGUMENTS
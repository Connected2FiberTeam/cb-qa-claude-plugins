---
name: qa-addquote
description: Add a new quote test scenario to the Karate integration test suite. Works in both RemotePricing2 (RP2) and RemotePricingAPI (RP1). Auto-detects the repo and adapts templates, provider lists, and field options accordingly.
allowed-tools: Read, Grep, Glob, Bash, AskUserQuestion, Edit, Write
---

# Add Quote Test Scenario Generator

## Repo Detection

**Run this before any user interaction.** Check for the base template file to determine the repo:

```bash
ls tests/integration/karate/src/test/resources/data/templates/common/ 2>/dev/null
```

- `rp2_base_template.json` present → **RP2 mode** (RemotePricing2)
  - Async: POST to Request Builder → poll Response Builder HTTP API (or wait for callback)
  - Supports: LMX adapters, callback mode, polling config fields, `cache`, `quoteMasterId`
  - Feature Background uses: `requestBuilderUrl` + `responseBuilderUrl`
  - No auth header required

- `rp1_base_template.json` present → **RP1 mode** (RemotePricingAPI)
  - Async: POST to RP1 → poll MongoDB `rp1_request_response`
  - Does NOT support: LMX adapters, `useCallback`, `callbackTimeout`, `initialDelay`, `maxRetries`, `retryInterval`, `cache`
  - Feature Background uses: `remotePricingUrl`
  - Auth header required: `header Authorization = accessToken`
  - Read timeout defaults to 180s (covers RP1's ~120s internal supplier timeout)

If neither file found, ask the user which repo/platform they're working in.

---

## Reference: Available testData Fields

**Required Fields:**
- `suppliers` - Provider name(s) — must match MongoDB `provider_credentials.provider_name`
- `speed` - Speed specification(s) e.g. `'ETHERNET 100M'`, `'ETHERNET 1G'`, `'ALL'`
- `testType` - `'success'` | `'no-coverage'` | `'validation-error'`
- `environments` - Comma-separated envs e.g. `'stage,uat,prod'`

**Validation Fields:**
- `validateRequestedConfig` - `'true'` (all fields), `'false'` (disabled), or specific fields like `'speed,product,accessMedium,term'`
- `assertions` - Field-level assertions e.g. `'provider=AT&T Wireline; mrc!=0'`

**Product/Service Fields (with defaults):**
- `products` - Product type(s) (default: `'Ethernet - Switched'`)
- `terms` - Contract term(s) (default: `'12'`). Use `'ALL'` for 12,24,36,60
- `accessMediums` - Access medium(s) (default: `'Fiber'`)

**Location Fields (one required):**
- `location` - Single address object
- `locations` - Array of addresses (multi-address)
- `locationA` + `locationZ` - P2P addresses

**Location Object Properties:**
- Core: `address`, `city`, `state`, `state_abbv`, `country`, `zipCode`
- Extended: `zipPlus4`, `county`, `latitude`, `longitude`
- Metadata: `source`, `rdi`, `usps_match`, `time_zone`, `utc_offset`
- Complex: `secondaryComponents` (array with `favoriteLocationId`)

**Request Options:**
- `forceFetch` - Force fresh fetch, bypass ES cache (default: `false`)
- `quoteMasterId` - Override quote master ID
- `alternativesPolicy` - Alternatives policy override

**RP2-only Fields:**
- `useCallback` - Use webhook callback instead of polling (default: `false`)
- `callbackTimeout` - Callback timeout in ms (default: `60000`)
- `initialDelay` - Initial polling delay in ms (default: `8000`)
- `maxRetries` - Max polling retries (default: global retryCount)
- `retryInterval` - Polling interval in ms (default: global retryInterval)
- `cache` - Enable response caching (default: `false`)

**Provider-Specific:**
- `suppliersApiRequestAttributes` - Provider-specific API attributes

---

## Step 1: LMX Adapter Check *(RP2 only — skip entirely in RP1 mode)*

Ask the user using AskUserQuestion:

**Question 1** (header: "API Type")
- Question: "Is this an LMX Adapter API?"
- Options:
  - "Yes" (description: "LMX Adapter API — uses different entity IDs (Stage: 1637, UAT: 6125, Prod: 6125)")
  - "No" (description: "Standard API — uses default entity IDs")

**If LMX Adapter:**
- Feature files go in `features/quote/LMX/` folder
- Feature files must include `* def entityId = lmxEntityId` in Background
- testData must include `entityId: '#(entityId)'`
- Tags must include `@lmx`

---

## Step 2: Existing or New Provider

Ask the user using AskUserQuestion:

**Question** (header: "Provider")
- Question: "Is this for an existing provider?"
- Options:
  - "Yes" (description: "Add scenario to an existing provider's feature file")
  - "No" (description: "Create a new provider feature file")

### If Existing Provider (Yes):

List available providers by reading the folders in `src/test/resources/features/quote/`.

**RP2 Standard Providers** (when in RP2 mode):
- AT&T, Beyond_Reach, Blue_Wireless, Cityfibre, ESUN, Earthlink, Eurofiber
- Fastweb, Frontier_ESB, IELO-LIAZO, Lyntia, Momentum_Cypress, Orange, Orchest
- PCCW, Qoolize, Resolute_Nexus, Retelit, Space_Hellas, Spectrum_Charter
- TI_Sparkle, Verizon
- Retail_Platform (contains: ACC, Nitel, Spectrum_Business, Spectrum_Enterprise, ViaSat)

**RP2 LMX Adapter Providers** (in `features/quote/LMX/`) *(RP2 only)*:
- A1_Telekom_Austria_NDA_DTAG, BT_Wholesale_DTAG, BT_Wholesale_Availability_DTAG
- Colt_eBonding_DTAG, KPN_Wholesale_Prequalification_DTAG, Odido_Partner_DTAG
- ESUN_DTAG, Lyntia_Quotation_DTAG, NOS_Wholesale_Pricing_DTAG, Optus_PODS_DTAG
- Phibee_VOP_Eligibility_DTAG, SFR_E_Access_DTAG, SFR_ISMA_DTAG, Space_Hellas_DTAG
- Sunrise_OneHub_Portal_DTAG, Vodafone_UK_DTAG

**RP1 Providers** (when in RP1 mode):
- AT&T: AT&T Wireline, AT&T ASE, AT&T AIA, AT&T HSIA-E, AT&T Wholesale
- CenturyLink / Lumen
- GTT XLink
- Nitel, Rogers, Shaw, Spectrum SMB, Telus, Verizon FTTI, Windstream Wholesale, Zayo, Entelegent

Ask which provider/feature file to add the scenario to, then skip to Step 5.

### If New Provider (No):

**RP2 mode + LMX Adapter (from Step 1):**
- New LMX providers go in `features/quote/LMX/` folder
- File naming: `{Provider_Name}_DTAG_Quote.feature`
- Skip folder question

**RP2 Standard / RP1 — Folder question:**

**Question** (header: "Folder")
- Question: "Should this new provider be in an existing folder or a new one?"
- Options:
  - "Existing folder" (description: "Add to an existing folder like Retail_Platform")
  - "New folder" (description: "Create a new folder under features/quote/")

If "Existing folder": Ask which folder to use.
If "New folder": Ask for the new folder name.

**Question** (header: "File Name")
- Question: "What should the new feature file be named?"
- Let user provide a name (will append .feature if not included)

Then create the folder (if new) and feature file using the appropriate template below.

---

### Feature File Templates

#### RP2 Standard API Template:
```gherkin
Feature: {Provider Name} Quote Tests
  Testing {Provider Name} integration through RP2

  Background:
    * url requestBuilderUrl
    * def responseBuilderUrl = responseBuilderUrl

  # ================================================================================
  # Single Address Test Suite
  # Use /qa-addquote to add scenarios to this feature file
  # ================================================================================
  @{provider_tag} @quote
  Scenario Outline: {Provider} <scenario> - <speed>
    * def suppliers = getValueOrDefault('<suppliers>', '{Default Supplier Name}')
    * def products = getValueOrDefault('<products>', '{Default Product}')
    * def terms = getValueOrDefault('<terms>', '12')
    * def accessMediums = getValueOrDefault('<accessMediums>', 'Fiber')
    * def validateRequestedConfig = getValueOrDefault('<validateRequestedConfig>', 'true')
    * def location = { address: "<address>", city: "<city>", state: "<state>", country: "<country>", zipCode: "<zipCode>", zipPlus4: "<zipPlus4>", latitude: <latitude>, longitude: <longitude>, county: "<county>" }
    * def testData = { suppliers: '#(suppliers)', speed: '<speed>', location: '#(location)', testType: '<testType>', environments: '<environments>', products: '#(products)', terms: '#(terms)', accessMediums: '#(accessMediums)', validateRequestedConfig: '#(validateRequestedConfig)' }
    * call read('classpath:helpers/quote_scenario_template.feature') testData

    Examples:
      | scenario | suppliers | speed | address | city | state | country | zipCode | zipPlus4 | latitude | longitude | county | testType | environments | products | terms | accessMediums | validateRequestedConfig |
```

#### RP2 LMX Adapter Template *(RP2 only)*:
```gherkin
Feature: {Provider Name} - DTAG Quote Tests
  Testing {Provider Name} - DTAG LMX Adapter integration through RP2

  Background:
    * url requestBuilderUrl
    * def responseBuilderUrl = responseBuilderUrl
    * def entityId = lmxEntityId

  # ================================================================================
  # LMX Adapter Test Suite
  # Entity IDs: Stage: 1637, UAT: 6125, Prod: 6125
  # Use /qa-addquote to add scenarios to this feature file
  # ================================================================================
  @lmx @{provider_tag} @quote
  Scenario Outline: <scenario>
    * def suppliers = getValueOrDefault('<suppliers>', '{Provider Name} - DTAG')
    * def products = getValueOrDefault('<products>', 'Dedicated Internet')
    * def terms = getValueOrDefault('<terms>', '12')
    * def accessMediums = getValueOrDefault('<accessMediums>', 'Fiber')
    * def validateRequestedConfig = getValueOrDefault('<validateRequestedConfig>', 'true')
    * def location = { address: "<address>", city: "<city>", state: "<state>", country: "<country>", zipCode: "<zipCode>", zipPlus4: "<zipPlus4>", latitude: <latitude>, longitude: <longitude>, county: "<county>" }
    * def testData = { suppliers: '#(suppliers)', speed: '<speed>', location: '#(location)', testType: '<testType>', environments: '<environments>', products: '#(products)', terms: '#(terms)', accessMediums: '#(accessMediums)', validateRequestedConfig: '#(validateRequestedConfig)', entityId: '#(entityId)' }
    * call read('classpath:helpers/quote_scenario_template.feature') testData

    Examples:
      | scenario | suppliers | speed | address | city | state | country | zipCode | zipPlus4 | latitude | longitude | county | testType | environments | products | terms | accessMediums | validateRequestedConfig |
```

#### RP1 Template:
```gherkin
Feature: {Provider Name} Quote Tests

  Background:
    * url remotePricingUrl

  # ================================================================================
  # Single Address Test Suite
  # Use /qa-addquote to add scenarios to this feature file
  # ================================================================================
  @{provider_tag} @quote
  Scenario Outline: {Provider} <scenario> - <speed>
    * def suppliers = getValueOrDefault('<suppliers>', '{Default Supplier Name}')
    * def products = getValueOrDefault('<products>', 'Ethernet - Switched')
    * def terms = getValueOrDefault('<terms>', '12')
    * def accessMediums = getValueOrDefault('<accessMediums>', 'Fiber')
    * def validateRequestedConfig = getValueOrDefault('<validateRequestedConfig>', 'true')
    * def location = { address: "<address>", city: "<city>", state: "<state>", country: "<country>", zipCode: "<zipCode>", zipPlus4: "<zipPlus4>", latitude: <latitude>, longitude: <longitude>, county: "<county>" }
    * def testData = { suppliers: '#(suppliers)', speed: '<speed>', location: '#(location)', testType: '<testType>', environments: '<environments>', products: '#(products)', terms: '#(terms)', accessMediums: '#(accessMediums)', validateRequestedConfig: '#(validateRequestedConfig)' }
    * call read('classpath:helpers/quote_scenario_template.feature') testData

    Examples:
      | scenario | suppliers | speed | address | city | state | country | zipCode | zipPlus4 | latitude | longitude | county | testType | environments | products | terms | accessMediums | validateRequestedConfig |
```

---

## Step 3: Run Command Reference

```bash
# RP2 — Stage (default)
cd tests/integration/karate
./gradlew test '-Dkarate.options=--tags @{provider_tag} --tags ~@bug --tags ~@uat --tags ~@prod' -Dkarate.mock.callback=false

# RP2 — UAT
./gradlew test '-Dkarate.options=--tags @{provider_tag} --tags ~@bug --tags ~@stage --tags ~@prod' \
  -Dkarate.env=uat -Dkarate.scenario.env=uat -Dkarate.mock.callback=false

# RP1 — Stage
cd tests/integration/karate
./gradlew test '-Dkarate.options=--tags @{provider_tag} --tags ~@bug --tags ~@uat --tags ~@prod'

# RP1 — UAT
MYSQL_HOST=$MYSQL_HOST_UAT MYSQL_PASSWORD=$MYSQL_PASSWORD_UAT \
./gradlew test '-Dkarate.options=--tags @{provider_tag} --tags ~@bug --tags ~@stage --tags ~@prod' \
  -Dkarate.env=uat -Dkarate.scenario.env=uat
```

No runner updates needed — the `@provider_tag` in the feature file determines which tests run.

---

## Step 4: Advanced Configuration (Optional)

For providers that need custom polling intervals or other advanced options, add them to testData:

```gherkin
# RP2 — custom polling (for slower APIs)
* def testData = { suppliers: '#(suppliers)', speed: '<speed>', location: '#(location)', testType: '<testType>', environments: '<environments>', retryInterval: 4000, maxRetries: 30 }

# Extended location fields (for providers requiring metadata)
* def location = { address: "<address>", city: "<city>", state: "<state>", state_abbv: "<state_abbv>", country: "<country>", zipCode: "<zipCode>", source: "source_2", rdi: "Commercial", secondaryComponents: [{ secondaryDesignator: '', secondaryNumber: '', favoriteLocationId: 0 }] }
```

---

## Step 5: Single, Multiple, or P2P Addresses

**Question** (header: "Addresses")
- Question: "What type of address configuration is this for?"
- Options:
  - "One address" (description: "Single location test scenario")
  - "Multiple addresses" (description: "Multi-location test scenario")
  - "P2P (Point-to-Point)" (description: "A-side and Z-side addresses for P2P products")

### If One Address:
Ask for the address details. User can provide in any format:
- Full address string: "123 Main St, City, State, Country, ZIP"
- Or individual components

### If Multiple Addresses:
Ask how many addresses, then collect them. For multi-address scenarios:
1. Add a `locationGroups` definition in the Background section (if not already present)
2. Create a Multi-Address Scenario Outline section (see Cityfibre.feature for reference)

### If P2P (Point-to-Point):
Ask for both A-side and Z-side address details. For P2P scenarios:
1. Add a `p2pLocationPairs` definition in the Background section (if not already present)
2. Create a P2P Scenario Outline section with `@p2p` tag

Example P2P Background setup:
```gherkin
* def p2pLocationPairs =
"""
{
  "LOCATION_PAIR_NAME": {
    "a": { "address": "...", "city": "...", "state": "...", "country": "...", "zipCode": "...", "county": "...", "latitude": 0.0, "longitude": 0.0 },
    "z": { "address": "...", "city": "...", "state": "...", "country": "...", "zipCode": "...", "county": "...", "latitude": 0.0, "longitude": 0.0 }
  }
}
"""
```

---

## Step 6: API Request Attributes

Some suppliers require API request attributes to be passed with the quote request.

**Question** (header: "API Attributes")
- Question: "Does this supplier require API request attributes?"
- Options:
  - "Yes" (description: "Supplier needs apiRequestAttributes (like PCCW, Resolute Nexus)")
  - "No" (description: "No special API attributes needed (like Cityfibre, Nitel)")

### If Yes:

Collect API request attributes for each product that needs them. Add to Background:

```gherkin
# {Supplier} API request attributes for {Product} product
* def {productVar}ApiRequestAttributes =
"""
[
  {
    "supplier": {
      "name": "{Supplier Name}"
    },
    "products": [
      {
        "id": {product_id},
        "name": "{Product Name}",
        "apiRequestAttributes": {
          "{Attribute Name}": {
            "selectedValue": "{value}"
          }
        }
      }
    ]
  }
]
"""
```

Also add `suppliersApiRequestAttributes: '#(apiRequestAttrs)'` to the testData object.

---

## Step 7: Quote Configuration

**IMPORTANT: Ask for EVERY scenario — each can have different product/term/access medium.**

Read the existing feature file to determine current defaults.

**Question** (header: "Defaults")
- Question: "Use the default quote setup? (Product: {product}, Term: {term}, Access Medium: {accessMedium})"
- Options:
  - "Yes" (description: "Use provider defaults — leave fields empty in test")
  - "No" (description: "Specify custom product, term, or access medium")

---

## Step 8: Speed Selection

**Question** (header: "Speed")
- Question: "What speed(s) will you be quoting for?"
- Options:
  - "Single speed" (description: "e.g., ETHERNET 100M, ETHERNET 1000M")
  - "Multiple speeds" (description: "Comma-separated list of speeds")
  - "ALL speeds" (description: "Test all available transmission rates")

---

## Step 9: Test Type

**Question** (header: "Test Type")
- Question: "What is the expected test outcome?"
- Options:
  - "success" (description: "Expects services to be returned")
  - "no-coverage" (description: "Expects no services (out of coverage area)")
  - "validation-error" (description: "Expects HTTP 400 validation error")

---

## Step 10: Strict Verification (Optional)

**Question** (header: "Validation")
- Question: "Enable strict verification that returned services match your request?"
- Options:
  - "Yes - All fields (Recommended)" (description: "Verify speed, product, term, and access medium all match")
  - "Yes - Custom fields" (description: "Choose which fields to verify")
  - "No" (description: "Skip strict verification")

- "Yes - All fields": set `validateRequestedConfig` to `'true'`
- "Yes - Custom fields": set to comma-separated list e.g. `'speed,product'`
- "No": omit from testData

---

## Step 11: Environment Selection

**Question** (header: "Environments")
- Question: "Which environments should this test run in?"
- Options:
  - "All environments" (description: "stage,uat,prod")
  - "Stage only" (description: "stage")
  - "Stage and UAT" (description: "stage,uat")
  - "Custom" (description: "Specify custom environment list")

---

## Step 12: Address Geocoding

For each address provided:
1. Parse address components (street, city, state, country, zip)
2. If latitude/longitude not provided, leave as `null`
3. If county not provided, leave blank
4. If zipPlus4 not provided, leave blank

If any address information cannot be determined exactly, leave those fields blank/null and note in the summary.

---

## Step 13: Generate Scenario

Add the new scenario row(s) to the appropriate Examples table:

```
| {scenario_name} | | {speed} | {address} | {city} | {state} | {country} | {zipCode} | {zipPlus4} | {latitude} | {longitude} | {county} | {testType} | {environments} | {products} | {terms} | {accessMediums} | {validateRequestedConfig} |
```

- Leave `validateRequestedConfig` empty to use default (`'true'`)
- Set to `'false'` for no-coverage tests
- Set to specific fields (e.g., `'speed,product'`) for custom validation

---

## Step 14: Add Another Scenario

**Question** (header: "Add More")
- Question: "Would you like to add another quote scenario to this provider?"
- Options:
  - "Yes" (description: "Add another scenario to the same feature file")
  - "No" (description: "Proceed to summary")

If "Yes": Return to Step 5.

---

## Step 15: Summary

Provide a summary of ALL scenarios added during this session:

- **File**: {feature_file} (NEW if created)
- **Repo Mode**: RP1 or RP2
- For each scenario: address, supplier, speed, product, term, access medium, testType, environments, strict verification, any blank fields

---

## Step 16: Run Tests Prompt

**Question** (header: "Run Tests")
- Question: "Would you like to run the quote tests for this provider now?"
- Options:
  - "Yes" (description: "Run tests with the appropriate command for this repo")
  - "No" (description: "Skip test execution")

If "Yes": Execute the appropriate run command and report results.

---

## Important Notes

- Always use the Read tool before editing any files
- Preserve existing scenarios when adding new ones
- Match the formatting and column alignment of existing Examples tables
- Use `null` for latitude/longitude when not provided (not empty string)
- Use empty string for optional text fields (zipPlus4, county) when not provided
- The supplier field in Examples should usually be left empty to use the default
- Products, terms, and accessMediums fields should be left empty when using defaults
- Generate meaningful scenario names that describe the test (e.g., "New York NY 100M", "London UK No Coverage")
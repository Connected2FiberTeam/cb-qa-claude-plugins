---
name: qa-addquote
description: Add a new quote test scenario to the Karate integration test suite. Works in RemotePricing2 (RP2), RemotePricingAPI (RP1), and UAP. Auto-detects the repo and adapts templates, provider lists, and field options accordingly.
allowed-tools: Read, Grep, Glob, Bash, AskUserQuestion, Edit, Write
---

# Add Quote Test Scenario Generator

## Repo Detection

**Run this before any user interaction.** Check for marker files to determine the repo:

```bash
ls src/test/resources/helpers/wait_for_uap_result.js 2>/dev/null
ls tests/integration/karate/src/test/resources/data/templates/common/ 2>/dev/null
```

- `wait_for_uap_result.js` present → **UAP mode** — skip to [UAP Mode](#uap-mode) section below
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

If no file found, ask the user which repo/platform they're working in.

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

## Important Notes (RP2/RP1)

- Always use the Read tool before editing any files
- Preserve existing scenarios when adding new ones
- Match the formatting and column alignment of existing Examples tables
- Use `null` for latitude/longitude when not provided (not empty string)
- Use empty string for optional text fields (zipPlus4, county) when not provided
- The supplier field in Examples should usually be left empty to use the default
- Products, terms, and accessMediums fields should be left empty when using defaults
- Generate meaningful scenario names that describe the test (e.g., "New York NY 100M", "London UK No Coverage")

---

## UAP Mode

Feature files live at: `src/test/resources/features/quote/{ProviderFolder}/{Provider}_Quote.feature`

Run commands execute from the **repo root** (not a subdirectory).

### Existing UAP Providers

- `Lumen` → `Lumen/Lumen_Quote.feature` — DIA (IpService); uses address dictionary + `addressKey` column
- `Windstream` → `Windstream/Windstream_Quote.feature` — Broadband; uses inline address columns
- `ESUN` → `ESUN/Esun_Quote.feature` — DIA/EAccess/ELine; uses address dictionary + unified `buildRequest` function

---

### UAP Step 1: Product Type

Ask the user using AskUserQuestion:

**Question** (header: "Product Type")
- Question: "Which UAP product type is this for?"
- Options:
  - "DIA / Dedicated Internet (IpService)" — single site, `@type: IpService`
  - "Ethernet Switched (EAccess)" — single site, `@type: EAccess`
  - "Ethernet Dedicated P2P (ELine)" — two sites, `@type: ELine`
  - "Broadband" — single site, `@type: Broadband`

---

### UAP Step 2: Existing or New Provider

Ask using AskUserQuestion:

**Question** (header: "Provider")
- Question: "Is this for an existing provider?"
- Options:
  - "Yes" (description: "Add scenario to an existing provider's feature file")
  - "No" (description: "Create a new provider feature file")

**If existing provider:** list the providers above, ask which one. Read the existing feature file before proceeding — note the address style (dictionary vs inline), the Scenario Outline column layout, and the `providerFilter` from existing `resultConfig` lines. Skip to UAP Step 4.

**If new provider:** ask for the provider name and proceed through all steps.

---

### UAP Step 3: Provider Filter (new providers only)

The `providerFilter` is the exact solution provider name as it appears in the UAP query service results (e.g., `'Lumen Quote v4'`, `'Windstream Wholesale Broadband Quoting'`, `'ESUN LMP Inquiry'`). It is used in `resultConfig` to wait for that specific provider's result.

Ask using AskUserQuestion:

**Question** (header: "Provider Filter")
- Question: "What is the exact provider filter string for the query service? (Check existing feature files or UAP solution provider config for the exact name)"
- Let the user type the exact string.

---

### UAP Step 4: Address(es)

Collect address information. For ELine (P2P), collect both A-side and Z-side.

**Single site (IpService, EAccess, Broadband):**

Ask for:
- `streetNr` — street number only (e.g., `"50"`)
- `streetName` — name only, no type (e.g., `"Fremont"`, not `"Fremont St"`)
- `streetType` — optional (e.g., `"St"`, `"Ave"`, `"Rd"`)
- `postcode` — postal/ZIP code
- `stateOrProvince` — 2-letter for US/CA; full region name for international
- `countryIso3` — 3-letter ISO code (e.g., `USA`, `GBR`, `CHN`)
- `city`
- `lat` / `lon` — optional; include if known (Lumen uses these for geocoded lookups)
- `locality` — optional; used for international addresses (e.g., district name in China)

Generate an address key in SCREAMING_SNAKE_CASE: `{CITY}_{DESCRIPTOR}` (e.g., `SAN_FRANCISCO_FREMONT`, `BEIJING_DIA`). Use a descriptor that distinguishes addresses for the same city.

**ELine (P2P) — two sites:**

Collect the same fields twice, labeled A-side and Z-side. Generate two keys: `siteAKey` and `siteZKey`.

---

### UAP Step 5: Quote Configuration

Ask using AskUserQuestion for each relevant field:

**Question** (header: "Bandwidth")
- Question: "What bandwidth in Mbps?" (integer, e.g., `100`, `1000`)

**Question** (header: "Term")
- Question: "What contract term in months?" (e.g., `12`, `24`, `36`)

**For EAccess, ELine, Broadband only:**

**Question** (header: "Access Media")
- Question: "What access media?"
- Options: `FIBRE` (default), `COPPER`, `WIRELESS`

**For Broadband only — also ask:**

**Question** (header: "Upload Bandwidth")
- Question: "What upload bandwidth in Mbps?" (e.g., `4`, `10`, `50` — often asymmetric)

---

### UAP Step 6: Test Type and Assertions

**Question** (header: "Test Type")
- Question: "What is the expected outcome?"
- Options:
  - "success" — expects pricing result returned
  - "no-coverage" — expects no result within timeout

**Question** (header: "Assertions")
- Question: "What assertions should be verified? (semicolon-separated, or leave blank for none)"
- Suggest defaults based on product type:
  - DIA: `supplierProductId=PROVIDER_DIA; mrc!=0; nrc!=0`
  - EAccess: `supplierProductId=PROVIDER_ETHERNET_SWITCHED; mrc!=0; nrc!=0`
  - ELine: `supplierProductId=PROVIDER_ETHERNET_DEDICATED; mrc!=0; nrc!=0`
  - Broadband: `supplierProductId=PROVIDER_BROADBAND; mrc!=0; nrc!=0`
- Replace `PROVIDER_*` with the actual supplier product ID if known.

**Supported assertion syntax:**

| Syntax | Meaning |
|--------|---------|
| `supplierProductId=LUMEN_DIA` | Exact supplier product ID match |
| `mrc!=0` | MRC is non-zero |
| `nrc!=0` | NRC is non-zero |
| `mrc=87.24` | Exact MRC amount |
| `nrc=150` | Exact NRC amount |
| `currencyIso3=USD` | Currency check |
| `chipset=FTTP` | Chipset value (Broadband) |
| `accessMedium=FIBRE` | Access medium check |

For `no-coverage` tests, leave assertions blank.

---

### UAP Step 7: Generate / Update the Feature File

#### If adding to an existing provider:

1. Read the feature file
2. **If the file uses an address dictionary** (has `* def addresses = """` in Background):
   - Check if the address key already exists; if not, add the new address entry to the `addresses` object
   - Add a new row to the appropriate Examples table
3. **If the file uses inline address columns** (Windstream style):
   - Add a new row to the Examples table with inline address fields
4. Match the existing column order exactly

#### If creating a new provider:

Create the folder and feature file:
```
src/test/resources/features/quote/{ProviderFolder}/{Provider}_Quote.feature
```

Use the appropriate template below based on the product type(s) needed.

**For a single product type**, use a product-specific template.
**For multiple product types**, use the unified buildRequest template (ESUN style).

---

### UAP Feature File Templates

#### Single Product — IpService (DIA):

```gherkin
Feature: {Provider Name} quote integration tests
  End-to-end tests for {Provider Name} solution provider.

  Background:
    * url restApiUrl
    * def queryServiceUrl = queryServiceUrl
    * def customer = testCustomer
    * def generateRef = function() { return '{provider-prefix}-' + java.lang.System.currentTimeMillis() + '-' + Math.floor(Math.random() * 10000) }
    * def addresses =
      """
      {
        "{ADDRESS_KEY}": {
          "streetNr": "{streetNr}",
          "streetName": "{streetName}",
          "streetType": "{streetType}",
          "postcode": "{postcode}",
          "stateOrProvince": "{stateOrProvince}",
          "countryIso3": "{countryIso3}",
          "city": "{city}"
        }
      }
      """

  # ============================================================================
  # Single Site - IP Services (DIA)
  # ============================================================================

  @{provider_tag} @quote @{provider_tag}-dia
  Scenario Outline: {Provider} DIA - <scenario>
    * def clientRequestRef = generateRef()
    * def address = addresses['<addressKey>']

    Given path 'requirement'
    And request
      """
      {
        "clientRequestReference": "#(clientRequestRef)",
        "customer": {
          "customerId": "#(customer.customerId)",
          "customerName": "#(customer.customerName)"
        },
        "requirements": [{
          "@type": "IpService",
          "productType": "IP_SERVICES",
          "clientRequirementReference": "<scenario>",
          "processingPriority": "MEDIUM",
          "contractTermMonths": <termMonths>,
          "site": {
            "clientSiteReference": "<scenario>-site",
            "location": {
              "address": {
                "@type": "FieldedAddress",
                "streetNr": "#(address.streetNr)",
                "streetName": "#(address.streetName)",
                "streetType": "#(address.streetType)",
                "postcode": "#(address.postcode)",
                "stateOrProvince": "#(address.stateOrProvince)",
                "countryIso3": "#(address.countryIso3)",
                "city": "#(address.city)"
              }
            },
            "serviceConfiguration": {
              "accessBandwidth": { "unit": "MBPS", "value": <bandwidth> },
              "serviceBandwidth": { "unit": "MBPS", "value": <bandwidth> }
            }
          }
        }]
      }
      """
    When method POST
    Then status 200
    And match response.systemRequestId == '#notnull'
    And match response.requirements[0].systemRequirementId == '#notnull'

    * def engagementId = response.requirements[0].systemRequirementId
    * karate.log('{Provider} DIA engagement ID:', engagementId)
    * def resultConfig = { engagementId: '#(engagementId)', queryUrl: '#(queryServiceUrl)', timeoutSeconds: '#(resultTimeoutSeconds)', testType: '<testType>', assertions: '<assertions>', providerFilter: '{Provider Filter String}' }
    * def pollResult = call read('classpath:helpers/wait_for_uap_result.js') resultConfig
    * karate.log('{Provider} DIA result status:', pollResult.status, 'source:', pollResult.source)
    * def isSuccessTest = '<testType>' == 'success'
    * if (isSuccessTest && pollResult.status != 'success') karate.fail('Expected success but got: ' + pollResult.status)

    Examples:
      | scenario | addressKey | termMonths | bandwidth | testType | assertions |
```

> **Note:** If `lat`/`lon` are available, add a `"geographicPoint"` block after `"address"`:
> ```json
> "geographicPoint": { "spatialReferenceSystem": "WGS84", "x": "#(address.lat)", "y": "#(address.lon)" }
> ```
> Add `"lat"` and `"lon"` fields to the address dictionary entry.

---

#### Single Product — EAccess (Ethernet Switched):

Key differences from IpService:
- `"@type": "EAccess"` — **no `productType` field**
- `site.serviceConfiguration`: `accessBandwidth` + `"accessMedia": ["<accessMedia>"]`
- `serviceBandwidth` is at the **requirement level** (outside `site`):

```gherkin
  @{provider_tag} @quote @{provider_tag}-eaccess
  Scenario Outline: {Provider} Ethernet Switched - <scenario>
    * def clientRequestRef = generateRef()
    * def address = addresses['<addressKey>']

    Given path 'requirement'
    And request
      """
      {
        "clientRequestReference": "#(clientRequestRef)",
        "customer": {
          "customerId": "#(customer.customerId)",
          "customerName": "#(customer.customerName)"
        },
        "requirements": [{
          "@type": "EAccess",
          "clientRequirementReference": "<scenario>",
          "processingPriority": "MEDIUM",
          "contractTermMonths": <termMonths>,
          "site": {
            "clientSiteReference": "<scenario>-site",
            "location": {
              "address": {
                "@type": "FieldedAddress",
                "streetNr": "#(address.streetNr)",
                "streetName": "#(address.streetName)",
                "streetType": "#(address.streetType)",
                "postcode": "#(address.postcode)",
                "stateOrProvince": "#(address.stateOrProvince)",
                "countryIso3": "#(address.countryIso3)",
                "city": "#(address.city)"
              }
            },
            "serviceConfiguration": {
              "accessBandwidth": { "unit": "MBPS", "value": <bandwidth> },
              "accessMedia": ["<accessMedia>"]
            }
          },
          "serviceBandwidth": { "unit": "MBPS", "value": <bandwidth> }
        }]
      }
      """
    When method POST
    Then status 200
    And match response.systemRequestId == '#notnull'
    And match response.requirements[0].systemRequirementId == '#notnull'

    * def engagementId = response.requirements[0].systemRequirementId
    * karate.log('{Provider} EAccess engagement ID:', engagementId)
    * def resultConfig = { engagementId: '#(engagementId)', queryUrl: '#(queryServiceUrl)', timeoutSeconds: '#(resultTimeoutSeconds)', testType: '<testType>', assertions: '<assertions>', providerFilter: '{Provider Filter String}' }
    * def pollResult = call read('classpath:helpers/wait_for_uap_result.js') resultConfig
    * karate.log('{Provider} EAccess result status:', pollResult.status, 'source:', pollResult.source)
    * def isSuccessTest = '<testType>' == 'success'
    * if (isSuccessTest && pollResult.status != 'success') karate.fail('Expected success but got: ' + pollResult.status)

    Examples:
      | scenario | addressKey | termMonths | bandwidth | accessMedia | testType | assertions |
```

---

#### Single Product — ELine (Ethernet Dedicated P2P):

```gherkin
  @{provider_tag} @quote @{provider_tag}-eline
  Scenario Outline: {Provider} Ethernet Dedicated - <scenario>
    * def clientRequestRef = generateRef()
    * def addressA = addresses['<siteAKey>']
    * def addressZ = addresses['<siteZKey>']

    Given path 'requirement'
    And request
      """
      {
        "clientRequestReference": "#(clientRequestRef)",
        "customer": {
          "customerId": "#(customer.customerId)",
          "customerName": "#(customer.customerName)"
        },
        "requirements": [{
          "@type": "ELine",
          "clientRequirementReference": "<scenario>",
          "processingPriority": "MEDIUM",
          "contractTermMonths": <termMonths>,
          "backhaul": {
            "backhaulBandwidth": { "unit": "MBPS", "value": <bandwidth> }
          },
          "sites": [
            {
              "clientSiteReference": "<scenario>-site-a",
              "location": {
                "address": {
                  "@type": "FieldedAddress",
                  "streetNr": "#(addressA.streetNr)",
                  "streetName": "#(addressA.streetName)",
                  "streetType": "#(addressA.streetType)",
                  "postcode": "#(addressA.postcode)",
                  "stateOrProvince": "#(addressA.stateOrProvince)",
                  "countryIso3": "#(addressA.countryIso3)",
                  "city": "#(addressA.city)"
                }
              },
              "serviceConfiguration": {
                "accessBandwidth": { "unit": "MBPS", "value": <bandwidth> },
                "accessMedia": ["<accessMedia>"]
              }
            },
            {
              "clientSiteReference": "<scenario>-site-z",
              "location": {
                "address": {
                  "@type": "FieldedAddress",
                  "streetNr": "#(addressZ.streetNr)",
                  "streetName": "#(addressZ.streetName)",
                  "streetType": "#(addressZ.streetType)",
                  "postcode": "#(addressZ.postcode)",
                  "stateOrProvince": "#(addressZ.stateOrProvince)",
                  "countryIso3": "#(addressZ.countryIso3)",
                  "city": "#(addressZ.city)"
                }
              },
              "serviceConfiguration": {
                "accessBandwidth": { "unit": "MBPS", "value": <bandwidth> },
                "accessMedia": ["<accessMedia>"]
              }
            }
          ]
        }]
      }
      """
    When method POST
    Then status 200
    And match response.systemRequestId == '#notnull'
    And match response.requirements[0].systemRequirementId == '#notnull'

    * def engagementId = response.requirements[0].systemRequirementId
    * karate.log('{Provider} ELine engagement ID:', engagementId)
    * def resultConfig = { engagementId: '#(engagementId)', queryUrl: '#(queryServiceUrl)', timeoutSeconds: '#(resultTimeoutSeconds)', testType: '<testType>', assertions: '<assertions>', providerFilter: '{Provider Filter String}' }
    * def pollResult = call read('classpath:helpers/wait_for_uap_result.js') resultConfig
    * karate.log('{Provider} ELine result status:', pollResult.status, 'source:', pollResult.source)
    * def isSuccessTest = '<testType>' == 'success'
    * if (isSuccessTest && pollResult.status != 'success') karate.fail('Expected success but got: ' + pollResult.status)

    Examples:
      | scenario | siteAKey | siteZKey | termMonths | bandwidth | accessMedia | testType | assertions |
```

---

#### Single Product — Broadband:

```gherkin
  @{provider_tag} @quote @{provider_tag}-broadband
  Scenario Outline: {Provider} Broadband - <scenario>
    * def clientRequestRef = generateRef()

    Given path 'requirement'
    And request
      """
      {
        "clientRequestReference": "#(clientRequestRef)",
        "customer": {
          "customerId": "#(customer.customerId)",
          "customerName": "#(customer.customerName)"
        },
        "requirements": [{
          "@type": "Broadband",
          "productType": "BROADBAND",
          "clientRequirementReference": "<scenario>",
          "processingPriority": "MEDIUM",
          "contractTermMonths": 12,
          "site": {
            "clientSiteReference": "<scenario>-site",
            "location": {
              "address": {
                "@type": "FieldedAddress",
                "streetNr": "<streetNr>",
                "streetName": "<streetName>",
                "streetType": "<streetType>",
                "postcode": "<postcode>",
                "stateOrProvince": "<state>",
                "countryIso3": "USA",
                "city": "<city>"
              }
            },
            "serviceConfiguration": {
              "accessMedia": ["<accessMedia>"],
              "serviceBandwidth": { "unit": "MBPS", "value": <downloadBandwidth> },
              "accessBandwidth": {
                "downloadBandwidth": { "unit": "MBPS", "value": <downloadBandwidth> },
                "uploadBandwidth": { "unit": "MBPS", "value": <uploadBandwidth> }
              }
            }
          }
        }]
      }
      """
    When method POST
    Then status 200
    And match response.systemRequestId == '#notnull'
    And match response.requirements[0].systemRequirementId == '#notnull'

    * def engagementId = response.requirements[0].systemRequirementId
    * karate.log('{Provider} Broadband engagement ID:', engagementId)
    * def resultConfig = { engagementId: '#(engagementId)', queryUrl: '#(queryServiceUrl)', timeoutSeconds: '#(resultTimeoutSeconds)', testType: '<testType>', assertions: '<assertions>', providerFilter: '{Provider Filter String}' }
    * def pollResult = call read('classpath:helpers/wait_for_uap_result.js') resultConfig
    * karate.log('{Provider} Broadband result status:', pollResult.status, 'source:', pollResult.source)
    * def isSuccessTest = '<testType>' == 'success'
    * if (isSuccessTest && pollResult.status != 'success') karate.fail('Expected success but got: ' + pollResult.status)

    Examples:
      | scenario | streetNr | streetName | streetType | postcode | state | city | accessMedia | downloadBandwidth | uploadBandwidth | testType | assertions |
```

---

#### Multi-Product — Unified buildRequest template (IpService + EAccess + ELine):

Use this when a provider supports multiple product types (like ESUN). The `buildRequest` function handles all three types from a single Scenario Outline.

```gherkin
Feature: {Provider Name} quote integration tests
  End-to-end tests for {Provider Name} solution provider.

  Background:
    * url restApiUrl
    * def queryServiceUrl = queryServiceUrl
    * def customer = testCustomer
    * def generateRef = function() { return '{provider-prefix}-' + java.lang.System.currentTimeMillis() + '-' + Math.floor(Math.random() * 10000) }
    * def addresses =
      """
      {
        "{ADDRESS_KEY}": {
          "streetNr": "{streetNr}",
          "streetName": "{streetName}",
          "streetType": "{streetType}",
          "postcode": "{postcode}",
          "stateOrProvince": "{stateOrProvince}",
          "countryIso3": "{countryIso3}",
          "city": "{city}"
        }
      }
      """
    * def buildRequest =
      """
      function(p) {
        var toAddr = function(a) {
          var o = { '@type': 'FieldedAddress', streetNr: a.streetNr, streetName: a.streetName,
                    postcode: a.postcode, stateOrProvince: a.stateOrProvince, countryIso3: a.countryIso3, city: a.city };
          if (a.streetType) o.streetType = a.streetType;
          if (a.locality) o.locality = a.locality;
          return o;
        };
        var toSite = function(ref, addr, bw) {
          return { clientSiteReference: ref, location: { address: toAddr(addr) },
                   serviceConfiguration: { accessBandwidth: { unit: 'MBPS', value: bw }, accessMedia: ['FIBRE'] } };
        };
        var r = { '@type': p.type, clientRequirementReference: p.scenario,
                  processingPriority: 'MEDIUM', contractTermMonths: p.termMonths };
        if (p.type === 'IpService') {
          r.productType = 'IP_SERVICES';
          var s = toSite(p.scenario + '-site', p.addressA, p.bandwidth);
          s.serviceConfiguration.serviceBandwidth = { unit: 'MBPS', value: p.bandwidth };
          r.site = s;
        } else if (p.type === 'EAccess') {
          r.site = toSite(p.scenario + '-site', p.addressA, p.bandwidth);
          r.serviceBandwidth = { unit: 'MBPS', value: p.bandwidth };
        } else if (p.type === 'ELine') {
          r.backhaul = { backhaulBandwidth: { unit: 'MBPS', value: p.bandwidth } };
          r.sites = [ toSite(p.scenario + '-site-a', p.addressA, p.bandwidth),
                      toSite(p.scenario + '-site-z', p.addressZ, p.bandwidth) ];
        }
        return { clientRequestReference: p.ref, customer: p.customer, requirements: [r] };
      }
      """

  # ============================================================================
  # All Products: DIA (IpService) | Ethernet Switched (EAccess) | Ethernet Dedicated (ELine)
  # ============================================================================

  @{provider_tag} @quote
  Scenario Outline: {Provider} - <scenario>
    * def clientRequestRef = generateRef()
    * def addressA = addresses['<siteAKey>']
    * def addressZ = '<siteZKey>' != '' ? addresses['<siteZKey>'] : null
    * def reqBody = buildRequest({ type: '<reqType>', scenario: '<scenario>', ref: clientRequestRef, customer: customer, addressA: addressA, addressZ: addressZ, bandwidth: <bandwidth>, termMonths: <termMonths> })

    Given path 'requirement'
    And request reqBody
    When method POST
    Then status 200
    And match response.systemRequestId == '#notnull'
    And match response.requirements[0].systemRequirementId == '#notnull'

    * def engagementId = response.requirements[0].systemRequirementId
    * karate.log('{Provider} engagement ID:', engagementId)
    * def resultConfig = { engagementId: '#(engagementId)', queryUrl: '#(queryServiceUrl)', timeoutSeconds: '#(resultTimeoutSeconds)', testType: '<testType>', assertions: '<assertions>', providerFilter: '{Provider Filter String}' }
    * def pollResult = call read('classpath:helpers/wait_for_uap_result.js') resultConfig
    * karate.log('{Provider} result status:', pollResult.status, 'source:', pollResult.source)
    * def isSuccessTest = '<testType>' == 'success'
    * if (isSuccessTest && pollResult.status != 'success') karate.fail('Expected success but got: ' + pollResult.status)

    Examples:
      | scenario | reqType | siteAKey | siteZKey | termMonths | bandwidth | testType | assertions |
```

> For ELine rows: set `siteZKey` to the Z-side address key. For single-site rows (IpService/EAccess): leave `siteZKey` as empty string `''`.

---

### UAP Step 8: Add Another Scenario

**Question** (header: "Add More")
- Question: "Would you like to add another scenario to this provider?"
- Options:
  - "Yes" (description: "Add another scenario")
  - "No" (description: "Proceed to summary")

If "Yes": Return to UAP Step 1.

---

### UAP Step 9: Summary

Provide a summary of all scenarios added:

- **File**: `src/test/resources/features/quote/{Provider}/{Provider}_Quote.feature` (NEW if created)
- **Mode**: UAP
- For each scenario: provider, product type, address, bandwidth, term, testType, assertions

---

### UAP Step 10: Run Tests Prompt

**Question** (header: "Run Tests")
- Question: "Would you like to run these tests now?"
- Options:
  - "Yes" (description: "Run from repo root with the appropriate Gradle command")
  - "No" (description: "Skip test execution")

**Run commands (from repo root):**

```bash
# Stage (default)
./gradlew test '-Dkarate.options=--tags @{provider_tag} --tags ~@bug --tags ~@uat --tags ~@prod' -Dkarate.env=stage

# UAT
./gradlew test '-Dkarate.options=--tags @{provider_tag} --tags ~@bug --tags ~@stage --tags ~@prod' -Dkarate.env=uat

# Prod
./gradlew test '-Dkarate.options=--tags @{provider_tag} --tags ~@bug --tags ~@stage --tags ~@uat' -Dkarate.env=prod
```

---

### UAP Important Notes

- Always use the Read tool before editing any feature file
- Preserve existing scenarios and address dictionary entries when adding new ones
- `@type` and `productType` must match exactly: IpService uses `productType: IP_SERVICES`; EAccess/ELine have no `productType`; Broadband uses `productType: BROADBAND`
- `serviceBandwidth` placement differs: inside `site.serviceConfiguration` for IpService, at requirement level for EAccess, not present for ELine (uses `backhaulBandwidth` instead)
- `providerFilter` must be the exact string the query service returns — check existing `resultConfig` lines in the feature file
- Scenario names follow the pattern: `{Provider}-{ProductAbbrev}-{City}-{Bandwidth}M-{Term}mo` (e.g., `Lumen-DIA-SanFrancisco-100M-12mo`)
- Leave `streetType` out of the address object entirely when not provided (don't set it to empty string)
- Run from repo root — there is no `tests/integration/karate/` subdirectory in UAP
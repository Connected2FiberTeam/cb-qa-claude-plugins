---
name: uap-apiingest
description: Ingest a UAP provider API from two Confluence pages (BA spec + test data). Generates catalogue and quote feature files. Run as /uap-apiingest <spec-url> <test-data-url>.
allowed-tools: Read, Grep, Glob, Bash, Write, Edit, AskUserQuestion, WebFetch, mcp__plugin_atlassian_atlassian__getConfluencePage
---

# UAP API Ingest

Generates both the catalogue and quote Karate feature files for a new (or existing) UAP provider from two Confluence pages.

**Usage:** `/uap-apiingest <ba-spec-url> <test-data-url>`

Both URLs must be Confluence pages. If not provided as arguments, ask the user for them.

---

## Step 0: Get URLs

If both URLs were passed as arguments, use them directly.
If either is missing, ask:

**Question** (header: "Confluence Pages")
- "Paste the BA spec page URL (products, terms, speeds, countries):"
- "Paste the Test Data page URL (test addresses and scenarios):"

---

## Step 1: Fetch Both Confluence Pages

Fetch both pages. Prefer the Confluence MCP tool (`mcp__plugin_atlassian_atlassian__getConfluencePage`) when available; fall back to `WebFetch` otherwise.

Extract from each page:
- **Provider name** — usually in the page title or a prominent heading (e.g., "NOS Comunicações S.A. UAP Implementation")
- **Page type** — spec page vs test data page (confirm which is which from content)

**Validate provider match:** Extract the provider name from both pages. If they clearly refer to different providers, stop and tell the user:
> "These pages appear to be for different providers: '{spec-provider}' and '{test-data-provider}'. Please check the URLs and try again."

If the names are a close match (e.g., abbreviated vs full name), proceed but note the discrepancy.

---

## Step 2: Check Existing Files

Scan the feature file tree:

```bash
ls src/test/resources/features/provider/
```

Check whether a folder and/or feature files already exist for this provider.

- If **no folder**: will create `src/test/resources/features/provider/{ProviderFolder}/`
- If **folder exists but no catalogue file**: will create `{Provider}_Catalogue.feature`
- If **folder exists but no quote file**: will create `{Provider}_Quote.feature`
- If **both files already exist**: tell the user and ask:

**Question** (header: "Files Exist")
- "Both catalogue and quote feature files already exist for {Provider}. What would you like to do?"
- Options:
  - "Update catalogue only" 
  - "Update quote only"
  - "Update both"
  - "Cancel"

Determine:
- `providerFolder` — folder name (PascalCase, e.g., `NOS`, `Lumen`, `Eurofiber`)
- `providerTag` — Karate tag (lowercase, e.g., `@nos`, `@lumen`, `@eurofiber`)
- `providerName` — display name (e.g., `NOS Comunicações S.A.`)
- `supplierName` — registered name in UAP catalogue (often ASCII-safe, e.g., `NOS Comunicacoes S.A.`)

Ask the user to confirm these if unsure:

**Question** (header: "Provider Details")
- "Confirm provider details extracted from the spec:"
  - Folder name: `{providerFolder}`
  - Tag: `{providerTag}`
  - Supplier name (as registered in UAP): `{supplierName}`
- Options:
  - "Looks correct — proceed"
  - "Let me correct them" (then ask for each value)

---

## Step 3: Parse the BA Spec Page (Catalogue Data)

From the spec page, extract the following. Each section may be a table, bullet list, or heading+body in Confluence.

### 3a. Products

Extract each supplier product:
- `supplierProductId` — the UAP product ID (often SCREAMING_SNAKE_CASE, e.g., `NOS_COMUNICACOES_E2NET_DEDICATED_INTERNET_ACCESS`)
- `supplierProductName` — display name
- `requirementType` — one of: `IP_SERVICE`, `E_ACCESS`, `BROADBAND`, `E_LINE`

### 3b. Countries

Extract supported countries as ISO3 codes (e.g., `PRT`, `USA`, `GBR`). If only ISO2 is listed, convert.

### 3c. Contract Terms

Extract supported term months (e.g., `[12, 24, 36]`).

### 3d. Access Media

Extract supported media per product (e.g., `FIBRE`, `COPPER`, `WIRELESS`).

### 3e. Speeds

For each product, extract the full speed list as download/upload kbps pairs. Common pattern:
- Symmetric: `10M` → `dlKbps=10000, ulKbps=10000`
- Asymmetric (Broadband): `100/10M` → `dlKbps=100000, ulKbps=10000`
- Convert Mbps to kbps (multiply by 1000)

### 3f. Rate Limits

If the spec page mentions rate limiting (requests per second/minute, throttling), note the values. These will be added as a comment in the catalogue feature file.

---

## Step 4: Generate the Catalogue Feature File

Use the parsed data to generate `{ProviderFolder}/{Provider}_Catalogue.feature`.

Follow the pattern from `src/test/resources/features/provider/_Provider_Catalogue.feature` exactly.

```gherkin
Feature: {providerName} catalogue validation
  Validates the {providerName} supplier product catalogue against the expected specification.
  See Confluence "{page-title}" for expected configuration.
  Catalogue endpoint: GET /catalogue/supplier/{supplierName}
  Supplier ID: {supplierName}
  {# If rate limits found: Rate limit: {N} requests per {unit} #}

  Background:
    * url catalogueServiceUrl

    Given path 'catalogue/supplier', '{supplierName}'
    When method GET
    Then status 200
    * match response.supplierName == '{supplierName}'
    * def providerProducts = response.products

  # ============================================================================
  # Exact product list — no missing, no unexpected extras
  # ============================================================================

  @{providerTag} @catalogue
  Scenario: {providerName} - registered products match specification
    * def actualIds = providerProducts.map(function(p) { return p.supplierProductId })
    * match actualIds contains only
      """
      [
        "{PRODUCT_ID_1}",
        "{PRODUCT_ID_2}"
      ]
      """

  # ============================================================================
  # Per-product metadata — name, requirement type, countries, terms, media
  # All checks bidirectional (no missing, no unexpected extras)
  # ============================================================================

  @{providerTag} @catalogue
  Scenario Outline: {providerName} - <productName> - metadata
    * def product = providerProducts.filter(function(p) { return p.supplierProductId === '<productId>' })[0]
    * match product.supplierProductName == '<productName>'
    * match product.requirementType == '<requirementType>'
    * def actualCountries = product.coverages.map(function(c) { return c.countryCode })
    * match actualCountries contains only <expectedCountries>
    * def coverage = product.coverages.filter(function(c) { return c.countryCode === '<primaryCountry>' })[0]
    * def activeTerms = coverage.contractTerms.filter(function(t) { return t.active }).map(function(t) { return t.termMonths })
    * match activeTerms contains only <expectedTerms>
    * def activeMedia = coverage.accessMedia.filter(function(m) { return m.active }).map(function(m) { return m.name })
    * def expectedMedia = <expectedMedia>
    * match activeMedia contains only expectedMedia

    Examples:
      | productId | productName | requirementType | expectedCountries | primaryCountry | expectedTerms | expectedMedia |
      {# one row per product #}

  # ============================================================================
  # Per-speed validation — one scenario per product / speed
  # ============================================================================

  @{providerTag} @catalogue
  Scenario Outline: {providerName} - <scenario>
    * def product = providerProducts.filter(function(p) { return p.supplierProductId === '<productId>' })[0]
    * def coverage = product.coverages.filter(function(c) { return c.countryCode === '{primaryCountry}' })[0]
    * def speedMatch = coverage.speeds.filter(function(s) { return s.active && s.downloadBandwidth.kbps == <dlKbps> && s.uploadBandwidth.kbps == <ulKbps> })
    * match speedMatch == '#[1]'

    Examples:
      | scenario | productId | dlKbps | ulKbps |
      {# one row per product+speed combination, grouped by product with a comment header #}
```

**Formatting rules:**
- Group speed rows by product with a `# {productName} — {media}` comment before each group
- Use `['ISO3']` format for single-country `expectedCountries`; `['ISO3_1', 'ISO3_2']` for multi
- Use `[12, 24, 36]` format for `expectedTerms`
- Use `['FIBRE']` or `['FIBRE', 'COPPER']` format for `expectedMedia`
- Scenario name format: `{Product Short Name} - {Media} - {Speed}` (e.g., `E2NET DIA - Fibre - 100M`)

---

## Step 5: Parse the Test Data Page (Quote Scenarios)

From the test data page, extract all test scenarios. Each scenario typically includes:
- Address (street, city, postcode, country, state/province)
- Product type (DIA, EAccess, Broadband, ELine)
- Bandwidth / speed
- Contract term
- Access media
- Expected outcome (success or noted as "no coverage")

Count the extracted scenarios. If more than **10 success scenarios**, ask:

**Question** (header: "Scenario Count")
- "The test data page lists {N} success scenarios. That's a lot — would you like to:"
- Options:
  - "Include all {N} scenarios" (description: "Generates a comprehensive but large feature file")
  - "I'll specify which to include" (description: "Tell me which products/locations to prioritise")
  - "Use one scenario per product type" (description: "Pick the best representative address for each product")

---

## Step 6: Determine Feature File Style

Based on the product types found in the test data:

- **Single product type only** (e.g., only DIA) → use a single Scenario Outline with inline columns (NOS style)
- **Multiple product types** (DIA + EAccess, or DIA + EAccess + ELine) → use the unified `buildRequest` function template (ESUN style)
- **Broadband only** → use the Broadband inline template

Ask the user to confirm the approach if mixed:

**Question** (header: "Feature File Style")
- "This provider has multiple product types ({list}). Use the unified buildRequest template?"
- Options:
  - "Yes — unified buildRequest (handles all types in one Scenario Outline)"
  - "No — separate Scenario Outlines per product type"

Also ask for the `providerFilter` string (the exact name as returned by the UAP query service):

**Question** (header: "Provider Filter")
- "What is the exact provider filter string? (This is the name the UAP query service returns for this SP — check the solution provider config or ask the SP dev)"
- Let the user type the value.

---

## Step 7: Generate the Quote Feature File

Generate `{ProviderFolder}/{Provider}_Quote.feature` using the appropriate template from the UAP skill templates (see qa-addquote UAP Feature File Templates section for the full template code).

For each success scenario from the test data, add one row to the Examples table.

**Scenario naming convention:** `{Provider}-{ProductAbbrev}-{City}-{Speed}M-{Term}mo`
- DIA → `DIA`
- EAccess → `EthSwitch`  
- ELine → `EthDedicated`
- Broadband → `BB`

**Assertions per product type:**
- IpService (DIA): `mrc!=0; currencyIso3={ISO3}`
- EAccess: `mrc!=0; currencyIso3={ISO3}`
- ELine: `mrc!=0; currencyIso3={ISO3}`
- Broadband: `mrc!=0; currencyIso3={ISO3}`

Leave `nrc!=0` out unless the spec explicitly states NRC is charged.

---

## Step 8: No-Coverage Scenarios

After the success scenarios, generate a no-coverage section. These use `testType: no-coverage` and empty assertions.

### 8a. Out-of-country scenario

Ask the user:

**Question** (header: "Out-of-Country Address")
- "What address should we use for the out-of-country no-coverage test? Provide a real address in a country {providerName} does NOT serve."
- Let the user provide the full address (or components).

Create one scenario:
- Product type: same as the first/primary success product
- Bandwidth: same as a passing scenario
- Term: same as a passing scenario
- Address: the out-of-country address provided

### 8b. Unsupported dimension scenarios

For each dimension the provider does NOT support, create one no-coverage scenario using an IN-COVERAGE address but with the unsupported value. Use the first/best passing address for all of these.

Generate one scenario per unsupported dimension found:

| Dimension | How to identify | Scenario |
|-----------|----------------|----------|
| **Unsupported product type** | A UAP `@type` not in the spec (e.g., if provider only does DIA, test EAccess) | Use in-coverage address, valid bandwidth/term/media, but wrong `@type` |
| **Unsupported media** | A media type not listed for the product (e.g., if only FIBRE, test COPPER) | Valid address/product/speed/term, wrong media |
| **Unsupported speed** | A bandwidth well outside the spec range (e.g., if max is 1G, test 10G) | Valid address/product/media/term, unsupported bandwidth |
| **Unsupported term** | A contract term not in the supported list (e.g., if only 12/24/36mo, test 60mo) | Valid address/product/media/speed, unsupported term |

Only generate scenarios for dimensions where there IS a clear unsupported value. Skip if every possible value is supported (e.g., if a provider supports all access media).

Ask the user to confirm the unsupported values before generating if any are ambiguous.

**Question** (header: "Unsupported Dimensions")
- "Based on the spec, I identified these unsupported values for no-coverage tests:
  - Unsupported product: {type} (provider only supports {supported-types})
  - Unsupported media: {media}
  - Unsupported speed: {speed}M
  - Unsupported term: {N}mo
  Does this look right, or should I adjust any?"
- Options:
  - "Correct — generate these scenarios"
  - "Let me adjust" (ask for corrections)

All no-coverage scenarios go in a separate `@{providerTag} @quote @no-coverage` Scenario Outline section at the bottom of the quote feature file.

---

## Step 9: Write the Files

Write the generated catalogue and quote files using the Write tool.

If either file already exists, use Edit to add/replace content rather than overwriting entirely, unless the user chose "Update both" and the content is being fully regenerated.

After writing, show the user a summary:

```
✅ Files written:

📁 src/test/resources/features/provider/{ProviderFolder}/
   ├── {Provider}_Catalogue.feature  ({N} products, {N} speed rows)
   └── {Provider}_Quote.feature      ({N} success scenarios, {N} no-coverage scenarios)

Tags: @{providerTag} @catalogue | @{providerTag} @quote
```

---

## Step 10: Ask to Run Tests

**Question** (header: "Run Tests")
- "Would you like to run the catalogue tests against stage now to verify the spec data is correct?"
- Options:
  - "Yes — run catalogue tests" (description: "Runs @{providerTag} @catalogue against stage")
  - "Yes — run quote tests" (description: "Runs @{providerTag} @quote against stage")
  - "Yes — run both"
  - "No — I'll run them manually"

If yes, execute from repo root:

```bash
# Catalogue
./gradlew test '-Dkarate.options=--tags @{providerTag} --tags @catalogue --tags ~@bug --tags ~@uat --tags ~@prod' -Dkarate.env=stage

# Quote
./gradlew test '-Dkarate.options=--tags @{providerTag} --tags @quote --tags ~@bug --tags ~@uat --tags ~@prod' -Dkarate.env=stage
```

---

## Important Notes

- Always fetch both Confluence pages before writing any files
- Validate provider name match before generating anything
- The `supplierName` in the catalogue Background must match exactly what is registered in the UAP catalogue service (ASCII safe, no special characters unless confirmed)
- Speed rows in the catalogue can be large — include ALL speeds from the spec, grouped by product with comment headers
- No-coverage scenarios use `testType: no-coverage` and empty assertions column
- If the test data page is incomplete or ambiguous, ask the user to clarify before generating
- Scenario names must be unique within the file
- Do not add `@bug` tags to newly generated scenarios — they are assumed correct until proven otherwise
- The catalogue feature file uses `@ignore` in `catalogue_validation_template.feature` — do NOT copy that; provider catalogue files are real tests
---
name: qa-apimapadd
description: Generate Karate API mapping feature files from PSAM CSV files. Handles product mappings, country mappings, and transmission rate lookups across Stage/UAT/Prod environments. Primarily used in RemotePricing2 (RP2).
allowed-tools: Read, Grep, Glob, Bash, AskUserQuestion, Edit, Write
---

# API Mapping Feature Generator

You are creating a new Karate API mapping feature file for testing provider product and country mappings.

## Step 1: Gather Information

Ask the user the following questions (use the AskUserQuestion tool with multiple questions):

1. **Folder Location** (header: "Folder")
   - Question: "Which folder should the feature file be placed in?"
   - Options:
     - "Use existing folder" (description: "Select from existing mapping folders")
     - "Create new folder" (description: "Create a new provider folder in features/mappings/")

2. **Feature File Name** (header: "File Name")
   - Question: "What should the feature file be named?"
   - Options:
     - "Auto-generate" (description: "Generate name from provider (e.g., Provider_Name_Mapping.feature)")
     - "Custom name" (description: "Specify a custom file name")

3. **PSAM File** (header: "PSAM File")
   - Question: "What is the PSAM CSV file name in Downloads?"
   - Options:
     - "Search Downloads" (description: "I'll search for CSV files in Downloads folder")
     - "Specify exact name" (description: "Provide the exact filename (e.g., PROVIDERPSAM.csv)")

4. **Transmission Rate Source** (header: "TR Source")
   - Question: "How should transmission rate IDs be populated?"
   - Options:
     - "Database lookup" (description: "Query MySQL database for existing transmission rates (Recommended)")
     - "CSV files" (description: "Use downloaded TR export CSV files from Downloads folder")
     - "No TR data" (description: "Leave transmission_rate_ids blank")

5. **Country Mapping** (header: "Countries")
   - Question: "How should country mappings be configured?"
   - Options:
     - "Database lookup" (description: "Query MySQL database using external_api_id (Recommended)")
     - "From CSV file" (description: "Use a country mapping CSV file from Downloads")
     - "USA only" (description: "Add only United States (223)")
     - "Skip for now" (description: "Leave country mappings empty")

## Step 2: Folder Setup

Based on the folder response:
- If "Create new folder": Ask for the folder name and create it under `src/test/resources/features/mappings/`
- If "Use existing folder": List available folders and ask which one to use

## Step 3: Provider Name Detection

From the PSAM file, read the `provider_name` column to determine the provider name for the feature file.

## Step 4: Create Feature File Structure

Using `features/mappings/Retail_Platform/ACC_Mapping.feature` as the template:
1. Create the feature file with proper structure
2. Set up Background with MySQLQueryHelper and provider name
3. Create Product Mapping scenario outline with:
   - Environment filtering
   - Transmission rate ID selection logic
   - Product mapping query and validation
   - Empty Examples table (will be populated in next step)
4. Create Country Mapping scenario outline with empty Examples table

## Step 5: Process PSAM File

1. Find the PSAM CSV file in `~/Downloads`
2. Read and parse the CSV data
3. For each row, generate a product mapping test case with all required fields

### CRITICAL: Scenario Naming Convention

**BEFORE generating any scenarios, read an existing mapping feature file** (e.g., `features/mappings/AT&T/AT&T_HSIA_E_Mapping.feature`) to see the exact format used.

The scenario name format is: `{Provider} - {sub_product_name} - {speed}`

**Speed format rules by product type:**

1. **Broadband products**: Always use `{download}/{upload}` format (for both symmetric AND asymmetric speeds)
   - Example: `AT&T HSIA-E - Broadband - 768k/384k` (asymmetric)
   - Example: `AT&T HSIA-E - Broadband - 100M/100M` (symmetric)

2. **All other products**: Use single speed format (symmetric format)
   - Example: `Comcast - Ethernet Fiber - 100M`
   - Example: `Comcast - Internet Fiber - 1G` (DIA where upload is null)

**Speed unit format:**
- Use the exact format from the PSAM file when available (e.g., "768k", "1.5M", "100M")
- If PSAM has raw numbers, convert to readable format:
  - < 1000: `{speed}M` (e.g., 100 -> "100M")
  - >= 1000 && < 10000: `{speed}M` or `{speed/1000}G` (e.g., 1000 -> "1000M" or "1G")
  - >= 10000: `{speed/1000}G` (e.g., 10000 -> "10G")

**What NOT to include in scenario names:**
- Access medium suffix (Fiber, COAX, Copper, etc.) — the sub_product_name already contains this
- The word "null" for DIA products

**Example correct scenarios:**
```
| AT&T HSIA-E - Broadband - 768k/384k                   | 7  | Broadband              | ...
| Comcast - Ethernet Fiber - 100M                       | 11 | Comcast Ethernet Fiber | ...
| Comcast - Internet Fiber - 1G                         | 6  | Comcast Internet Fiber | ...
```

**Example INCORRECT scenarios (do NOT use):**
```
| Comcast - Ethernet Fiber - 100/100 - Fiber            | ...  ← Wrong: access medium at end, no unit
| Comcast - Internet Fiber - 100/null                   | ...  ← Wrong: shows "null"
| Comcast - Ethernet Fiber - 100M/100M                  | ...  ← Wrong: non-Broadband using download/upload format
```

## Step 6: Process Transmission Rates

Based on the user's TR Source selection:

### Option A: Database Lookup (Recommended)

**Required:** MySQL environment variables configured in shell:
- `MYSQL_USER` — MySQL username
- `MYSQL_HOST_STAGE`, `MYSQL_PASSWORD_STAGE` — Stage connection
- `MYSQL_HOST_UAT`, `MYSQL_PASSWORD_UAT` — UAT connection
- `MYSQL_HOST_PROD`, `MYSQL_PASSWORD_PROD` — Production connection

**IMPORTANT:** UAT/Prod passwords contain special characters (`*`, `,`). Use single-quoted literal password with `-p'<password>'` syntax instead of `MYSQL_PWD` environment variable. Stage password has no special chars so env var works. See `~/.zshrc` for actual credentials.

**Process:**
1. **Query all three environments** since transmission_rate_ids can differ between environments
2. For each environment, run the MySQL query:
```bash
# Stage (no special chars in password - env var works)
MYSQL_PWD=$MYSQL_PASSWORD_STAGE mysql -h $MYSQL_HOST_STAGE -u $MYSQL_USER -D connected2fiber -N -e "
SELECT psam.sub_product_name, psam.access_medium_id,
       psam.provider_download_speed, psam.provider_upload_speed,
       psam.transmission_rate_id, fs.transmissionrate
FROM product_service_api_mapping psam
LEFT JOIN fiberspeed fs ON psam.transmission_rate_id = fs.transmissionrateid
WHERE psam.provider_name = 'PROVIDER_NAME' AND psam.is_active = 1"

# UAT (special chars - use single-quoted literal password from ~/.zshrc)
mysql -h mysqlmaster-uat.connectbase.dev -u u_dond -p'<UAT_PASSWORD>' -D connected2fiber -N -e "..."

# Prod (special chars - use single-quoted literal password from ~/.zshrc)
mysql -h mysqlmaster-prod.connectbase.dev -u u_dond -p'<PROD_PASSWORD>' -D connected2fiber -N -e "..."
```

3. **For each PSAM entry**, find the matching transmission_rate_id from each environment by matching on:
   - sub_product_name, access_medium_id, provider_download_speed, provider_upload_speed

4. **If transmission_rate_ids differ** between environments, use environment-specific columns in the Examples table:
   - `transmission_rate_id_stage`, `transmission_rate_id_uat`, `transmission_rate_id_prod`

5. **If all environments have the same transmission_rate_id**, use a single `transmission_rate_ids` column

**Fallback:** If environment variables are not configured, inform the user and offer to use the CSV file option instead.

### Option B: CSV Files

When "CSV files" is selected:
1. Ask user: "Which environments have TR files?" — Stage only, or Stage/UAT/Prod
2. Find files in Downloads (e.g., PROVIDERTRSTAGE.csv, PROVIDERTRUAT.csv, PROVIDERTRPROD.csv)
3. Create lookup maps for each environment, match each PSAM row to its transmission_rate_id

### Option C: No TR Data

When "No TR data" is selected:
- Leave transmission_rate_id columns blank
- Tests will need TR data added manually before they can pass

### IMPORTANT: PSAM is the Source of Truth

The PSAM document is the authoritative source for product mappings. When a PSAM entry does NOT have a matching transmission rate entry:

1. **Alert the user** with a clear warning listing the scenario name, sub_product_name, access_medium_id, and speeds that had no TR match
2. **Still create the test** with blank transmission_rate_ids

3. **Ask the user** (using AskUserQuestion tool):
   - Question: "The following PSAM entries have no matching Transmission Rate entries: [list entries]. Should these tests be commented out?"
   - Header: "No TR Match"
   - Options:
     - "Comment out" (description: "Comment out these test rows — they won't run until TR data is added")
     - "Keep active" (description: "Keep tests active with blank TR IDs — tests will fail until TR data exists")

4. **If "Comment out"**: Prefix rows with `#` and add `# TODO: Missing TR entry - add transmission_rate_id when available`

## Step 7: Process Country Mappings

Based on country mapping option:

### Option A: Database Lookup (Recommended)

**IMPORTANT:** The `external_api_id` (supplier ID) differs between environments. Query each environment separately.

**Same credential handling as Step 6** — Stage uses env var, UAT/Prod use single-quoted literal passwords from `~/.zshrc`.

**Process:**
1. **Get the external_api_id** for the provider in each environment:
```bash
# Stage
MYSQL_PWD=$MYSQL_PASSWORD_STAGE mysql -h $MYSQL_HOST_STAGE -u $MYSQL_USER -D connected2fiber -N -e "
SELECT * FROM remote_price_supplier WHERE supplier LIKE 'PROVIDER_NAME'"
```

2. **Query the country mappings** for each environment using its external_api_id:
```bash
# Stage (e.g., external_api_id = 444)
MYSQL_PWD=$MYSQL_PASSWORD_STAGE mysql -h $MYSQL_HOST_STAGE -u $MYSQL_USER -D connected2fiber -N -e "
SELECT c.countries_id, c.countries_name, c.countries_iso_code_2, c.countries_iso_code_3
FROM remote_price_api_country rpac
INNER JOIN country c ON rpac.country_id = c.countries_id
WHERE rpac.external_api_id = 444
ORDER BY c.countries_name"
```

3. **Compare results across environments** and identify differences:
   - Countries that exist in all environments: set `environments` to `stage,uat,prod`
   - Countries with name differences (e.g., "Libyan Arab Jamahiriya" vs "Libya"): create separate rows for each variant with appropriate environment restrictions
   - Countries only in specific environments: set `environments` accordingly

   **Example of handling differences:**
   ```
   | Space Hellas - 121 - Libyan Arab Jamahiriya | 121 | Libyan Arab Jamahiriya | LY | LBY | stage     |
   | Space Hellas - 121 - Libya                  | 121 | Libya                  | LY | LBY | uat,prod  |
   ```

### Option B: From CSV File

1. Ask for the CSV filename in Downloads
2. Read the CSV which should have columns: countries_id, countries_name, countries_iso_code_2, countries_iso_code_3
3. Populate the Examples table with all countries from the CSV

### Option C: USA Only

Add only USA (223, United States, US, USA)

### Option D: Skip

Leave Examples table empty

## Step 8: Format Tables

Apply consistent spacing/alignment to both product and country mapping tables:
- **Read an existing mapping feature file** (e.g., `features/mappings/Retail_Platform/ACC_Mapping.feature`) to ensure consistent formatting
- Match the exact structure and formatting style of existing mapping tests
- Align all columns properly

## Step 9: Running the Tests

```bash
# Stage
cd tests/integration/karate
./gradlew test '-Dkarate.options=--tags @{provider_tag} --tags @mapping --tags ~@bug --tags ~@uat --tags ~@prod' -Dkarate.mock.callback=false

# UAT
./gradlew test '-Dkarate.options=--tags @{provider_tag} --tags @mapping --tags ~@bug --tags ~@stage --tags ~@prod' \
  -Dkarate.env=uat -Dkarate.scenario.env=uat -Dkarate.mock.callback=false
```

No runner updates needed — the `@provider_tag` in the feature file determines which tests run.

## Step 10: Summary

Provide a summary showing:
- Feature file location
- Number of product mappings created
- Number of country mappings created
- Breakdown by sub-product and access medium
- Command to run the tests

### Environment Differences Summary

If any differences were found between environments, include a detailed summary:

**Transmission Rate Differences:**
```
| Product                          | Speed    | Stage TR | UAT TR | Prod TR |
|----------------------------------|----------|----------|--------|---------|
| Broadband - Fiber                | 180/10   | 2070     | 2127   | 2163    |
```

**Country Mapping Differences:**
```
| Country ID | Stage Name              | UAT/Prod Name | Environments Affected |
|------------|-------------------------|---------------|----------------------|
| 121        | Libyan Arab Jamahiriya  | Libya         | Stage vs UAT/Prod    |
```

If no differences were found, state: "No environment differences found — all transmission rates and country mappings are consistent across Stage, UAT, and Prod."

## Important Notes

- Always use the Read tool before editing files
- Preserve exact column structure from ACC_Mapping.feature
- Handle CSV encoding properly (utf-8-sig)
- Validate that all required CSV columns exist before processing
- **PSAM is the source of truth** — all PSAM entries must generate tests, even without TR matches
- Create well-formatted, readable feature files with proper Gherkin syntax
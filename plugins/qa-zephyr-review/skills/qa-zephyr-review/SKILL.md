---
name: qa-zephyr-review
description: Review Zephyr Scale test cases for completeness, clarity, and platform standards. Supports RP1, RP2, LMX, and UAP platforms. Accepts XML file export or fetches via API.
allowed-tools: Bash(curl:*), Read, WebFetch, Grep, Glob
---

# Zephyr Scale Test Case Review

## Overview

Before beginning any review, read `.claude/qa-testing-context.md` for QA domain knowledge including CPQ test architecture, what belongs in Zephyr steps vs. notes, clean step content rules, and the Call-To-Test catalog.

Review Zephyr Scale test cases to ensure they meet quality standards for:
- Completeness of test information
- Clarity of test data
- Proper test steps and expected results
- No assumed information

## Prerequisites

**For API Access (Optional):**

Zephyr Scale API token enables fetching test cases directly. Check for:
```bash
echo $ZEPHYR_SCALE_TOKEN
```

If not set and you want API access:
1. Go to Zephyr Scale > Settings > API Keys
2. Generate a new API token
3. Add to `~/.zshrc`: `export ZEPHYR_SCALE_TOKEN="your-token-here"`

**For XML Import (No API Required):**

Export test cases from Zephyr Scale as XML and provide the file path.

---

## Phase 1: Get Test Case Input

Ask the user using AskUserQuestion:

**Question 1** (header: "Input Method")
- Question: "How would you like to provide the test case?"
- Options:
  - "XML File" (description: "Read from exported Zephyr Scale XML file (no API token needed)")
  - "Test Case Key" (description: "Fetch via API using test case key (e.g., ACE-T1234)")
  - "Multiple Keys" (description: "Fetch multiple test cases via API")
  - "Folder/Cycle" (description: "Fetch all test cases in a folder or test cycle via API")

### If XML File:
Ask for the path to the XML file. The file can be:
- An absolute path (e.g., `$HOME/Downloads/testcase-export.xml`)
- A relative path from the current directory

**How to export from Zephyr Scale:**
1. Open the test case in Zephyr Scale
2. Click the "..." menu (more actions)
3. Select "Export" > "Export to XML"
4. Save the file locally

### If Test Case Key:
Ask for the test case key (e.g., ACE-T1234).

### If Multiple Keys:
Ask for comma-separated keys (e.g., ACE-T1234, ACE-T1235, ACE-T1236).

### If Folder/Cycle:
Ask for the folder name or test cycle key.

---

## Phase 2: Load Test Case Data

### Method A: XML File Import (Recommended - No API Required)

Read the XML file using the Read tool and parse the Zephyr Scale XML structure.

**Zephyr Scale XML Structure:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<testCases>
  <testCase>
    <key>ACE-T1234</key>
    <name>Test Case Title</name>
    <objective>Test objective/description</objective>
    <precondition>Prerequisites and setup</precondition>
    <status>Draft|Approved|Deprecated</status>
    <priority>High|Medium|Low</priority>
    <folder>/Folder/Path</folder>
    <labels>
      <label>label1</label>
    </labels>
    <testScript>
      <steps>
        <step>
          <index>1</index>
          <description>Step description</description>
          <testData>Test data for this step</testData>
          <expectedResult>Expected outcome</expectedResult>
        </step>
      </steps>
    </testScript>
  </testCase>
</testCases>
```

**Note:** Zephyr Scale XML exports may vary slightly. Look for these common element names:
- `testCase`, `test-case`, or `TestCase`
- `testSteps`, `test-steps`, `steps`, or `testScript`
- `expectedResult`, `expected-result`, or `expectedOutcome`

---

### Method B: API Fetch (Requires Token)

**API Base URL:** `https://api.zephyrscale.smartbear.com/v2`

```bash
# Fetch single test case
curl -s -H "Authorization: Bearer $ZEPHYR_SCALE_TOKEN" \
  "https://api.zephyrscale.smartbear.com/v2/testcases/{testCaseKey}" | jq '.'

# Search test cases by folder
curl -s -H "Authorization: Bearer $ZEPHYR_SCALE_TOKEN" \
  "https://api.zephyrscale.smartbear.com/v2/testcases?projectKey=ACE&folderId={folderId}" | jq '.'

# Fetch test steps
curl -s -H "Authorization: Bearer $ZEPHYR_SCALE_TOKEN" \
  "https://api.zephyrscale.smartbear.com/v2/testcases/{testCaseKey}/teststeps" | jq '.'
```

---

## Review Philosophy

**Avoid being overly pedantic.** The goal is to identify issues that genuinely impact test usability, not to nitpick formatting or grammar.

### Key Principles:

1. **Environment listing = environment-agnostic**: When a test lists multiple environments (e.g., "Stage/UAT/Prod"), this means the test is designed to run on ANY of those environments. This is the standard practice — do NOT flag as ambiguous.

2. **Intent over perfection in expected results**: If the intent of an expected result is clear, do NOT flag minor grammar or wording issues as critical. Only flag if the expected result is genuinely unclear or unverifiable.

3. **Ticket reference provides context**: When a title includes a ticket reference like "[ACE-XXXX] Description", the Jira ticket provides the business context. Do NOT require a separate objective field in these cases.

4. **Don't require obvious operational details**: Preconditions should document what's non-obvious. Do NOT require DB host URLs when environment is specified, VPN details beyond "VPN required", or standard tool access that any tester would have.

5. **Critical issues = genuinely blocking**: Only mark something as "Critical" if it makes the test unusable or unreliable. Grammar issues, missing optional fields, and formatting problems are NOT critical.

---

## Phase 3: Review Checklist

### 1. Test Case Identification (REQUIRED)

- [ ] **Meaningful Title** — Title clearly describes what is being tested
  - Bad: "Test 1", "Quote Test", "API Test"
  - Good: "Verify Cityfibre returns 100M pricing for London address"
  - Acceptable: Title with ticket reference if ticket provides context
- [ ] **Platform Label (REQUIRED)** — Must have at least one platform label:
  - `RP1` — RemotePricing 1 (legacy)
  - `RP2` — RemotePricing 2 (includes LMX/DTAG adapters)
  - `LMX` — LMX platform (separate legacy platform)
  - `UAP` — Unified API Platform
  - Missing platform label = deduct 1 point from Identification score
- [ ] **Additional Labels** — Recommended: provider name, test type (quote, mapping, data-integrity)
- [ ] **Correct Folder** — Located in appropriate test folder structure

### 2. Preconditions (RECOMMENDED)

- [ ] **Authentication/Access** — Any required credentials or access documented?
- [ ] **Data Dependencies** — Required test data or database state documented?
- [ ] **External Dependencies** — Third-party services or APIs that must be available?

**Note on Environment:**
- Tests should be written to run on any environment (Stage, UAT, Prod) — this is the standard
- If environment-specific data IS required, verify it's provided for ALL environments:
  - **Instance IDs** — e.g., "Stage: 1609, UAT: 6407, Prod: 6415"
  - **URLs** — e.g., environment-specific endpoints if not using config variables
- Flag as an issue if environment-specific data is only provided for one environment

### 3. Test Data Clarity (CRITICAL)

- [ ] **No Assumed Data** — All test data explicitly stated, not assumed
  - Bad: "Use a valid address"
  - Good: "Use address: 123 Main St, London, UK, EC1A 1BB"
- [ ] **Data Sources Identified** — Where does test data come from?
- [ ] **Variable Data Marked** — Dynamic values clearly identified with placeholders
- [ ] **Sensitive Data Handled** — No hardcoded credentials or PII

#### Platform-Specific Data Requirements:
- [ ] **Provider Name Specified** — Full provider name (e.g., "Cityfibre", "Resolute Nexus", "AT&T Wireline")
- [ ] **Speed Value Format** — Correct format (e.g., "ETHERNET 100M", "ETHERNET 1000M")
- [ ] **Address Components** — Full address with: street, city, state/region, country, postal code
- [ ] **Product Type** — If non-default, explicitly stated (e.g., "Dedicated Internet", "Ethernet - Switched")
- [ ] **Term Length** — Contract term if relevant (e.g., "12 months", "36 months")
- [ ] **Access Medium** — If relevant (e.g., "Fiber", "Copper", "Wireless")

#### Environment-Agnostic Data Standard:
- [ ] **Instance IDs for All Environments** — If test requires specific instance, provide for Stage, UAT, AND Prod
  - Good: "Stage: 1609, UAT: 6407, Prod: 6415"
  - Bad: "Use instance 1609" (only works on Stage)

### 4. Test Steps (REQUIRED)

- [ ] **Steps Present** — Test has defined steps (not empty)
- [ ] **Steps Are Actionable** — Each step describes a specific action
  - Bad: "Test the API"
  - Good: "Send POST request to /quote endpoint with the following payload: {...}"
- [ ] **Steps Are Sequential** — Logical order that can be followed
- [ ] **No Ambiguous Steps** — Each step has only one interpretation
- [ ] **Technical Details Included** — API endpoints, request payloads, UI navigation paths

**Note on "Call to Test" Steps:**
- Zephyr Scale supports "Call to Test" steps that reference other test cases (e.g., QTO-T6, QTO-T7, QTO-T8)
- This is a **valid and encouraged pattern** for test reusability — do NOT penalize this
- Evaluate CTT usage against the catalog and ordering rules in `.claude/qa-testing-context.md` (CTTs section)
- Only flag as an issue if: the CTT key is invalid, required parameters are missing, or the ordering is wrong

### 5. Expected Results (REQUIRED)

- [ ] **Expected Results Present** — Each step has an expected result
- [ ] **Results Are Verifiable** — Can be objectively checked (not subjective)
  - Bad: "Response looks correct"
  - Good: "Response HTTP status is 200 and body contains 'services' array with at least 1 item"
- [ ] **Specific Values When Applicable** — Expected field values, status codes, messages
- [ ] **Error Scenarios Covered** — What happens when things go wrong?

#### Platform-Specific Expected Results:
- [ ] **HTTP Status Codes** — Expected status (200, 400, 404, etc.)
- [ ] **Response Structure** — Key fields to verify (services, mrc, nrc, term)
- [ ] **Provider Response** — Expected provider status (Completed, Failed, Pending)
- [ ] **Pricing Fields** — If checking pricing: MRC, NRC, or "no coverage" expectation

### 6. Test Type & Coverage (RECOMMENDED)

- [ ] **Test Type Identified** — Functional, Integration, E2E, Regression, Smoke
- [ ] **Priority Set** — High, Medium, Low based on business impact
- [ ] **Automation Status** — Manual, Automated, Candidate for Automation
- [ ] **Related Requirements** — Linked to user story or requirement (ACE ticket)

### 7. Maintainability (RECOMMENDED)

- [ ] **No Hardcoded Environment URLs** — Use environment variables or config
- [ ] **Version/Date Context** — When was this test created/last updated?
- [ ] **Author Identified** — Who created/owns this test?
- [ ] **Change History** — For modified tests, what changed and why?

---

## Phase 4: Output Format

### Test Case Review: {TEST_CASE_KEY}

**Title:** {test_case_title}
**Status:** {current_status}
**Folder:** {folder_path}

---

### Quality Score: {X}/10

| Category | Score | Issues |
|----------|-------|--------|
| Identification | {0-2} | {issues or "None"} |
| Preconditions | {0-2} | {issues or "None"} |
| Test Data | {0-2} | {issues or "None"} |
| Test Steps | {0-2} | {issues or "None"} |
| Expected Results | {0-2} | {issues or "None"} |

**Scoring Notes:**
- **Platform Label** is REQUIRED — missing platform label (RP1, RP2, LMX, UAP) deducts 1 point from Identification
- **Objective** is optional — ticket reference in title provides sufficient context
- **Environment** — listing multiple environments (Stage/UAT/Prod) means environment-agnostic (the standard)
- **Expected Results** — score based on whether intent is clear, not grammar perfection

---

### Critical Issues (Must Fix)
[List issues that make the test case unusable or unreliable]

### Improvements Needed
[List issues that reduce test quality but don't block execution]

### Suggestions
[Optional enhancements to improve the test case]

---

### Recommended Changes

Provide specific, actionable recommendations:

```
1. [Specific change with before/after example]
2. [Specific change with before/after example]
```

---

### Platform-Specific Findings

**Provider Coverage:**
- Provider: {provider_name or "NOT SPECIFIED"}
- Has valid test data: {Yes/No}

**Test Data Completeness:**
- Address: {Complete/Incomplete/Missing}
- Speed: {Specified/Missing}
- Product: {Specified/Using Default}
- Environment: {Specified/Missing}

**Automation Readiness:**
- Can be automated with Karate: {Yes/No/Needs Modification}
- Blocking issues for automation: {list or "None"}

---

## Phase 5: Multiple Test Case Summary

If reviewing multiple test cases, provide a summary table:

| Test Case | Score | Critical Issues | Status |
|-----------|-------|-----------------|--------|
| ACE-T1234 | 8/10 | 0 | Ready |
| ACE-T1235 | 5/10 | 2 | Needs Work |

**Overall Assessment:**
- Total test cases reviewed: {count}
- Ready for use: {count}
- Needs improvement: {count}
- Requires major rework: {count}

---

## Phase 6: Next Steps

**Question** (header: "Next Steps")
- Question: "What would you like to do next?"
- Options:
  - "Review more test cases" (description: "Review additional test cases")
  - "Generate improvement suggestions" (description: "Create detailed fix recommendations")
  - "Export findings" (description: "Save review results to a file")
  - "Done" (description: "End the review session")

### If Export findings:
Create a markdown file with the review results:
```
ZEPHYR-REVIEW-{date}-{test_case_key}.md
```

---

## Appendix: Test Data Reference

See `.claude/qa-testing-context.md` (Test Data Reference section) for the authoritative list of valid platform labels, speed formats, products, access mediums, and address field requirements.

$ARGUMENTS
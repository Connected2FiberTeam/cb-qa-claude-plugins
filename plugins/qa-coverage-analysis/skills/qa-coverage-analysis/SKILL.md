---
name: qa-coverage-analysis
description: Analyze Jira ticket test coverage — requirements, code changes, Zephyr Scale test cases, and Karate integration gaps. Works in both RemotePricing2 (RP2) and RemotePricingAPI (RP1). Use with a Jira ticket key e.g. /qa-coverage-analysis ACE-2908
allowed-tools: Bash(ls:*), Bash(git:*), Bash(python3:*), Bash(python3 <<:*), Bash(curl:*), Bash(echo:*), Bash(file:*), Bash(grep:*), Bash(find:*), Read, Grep, Glob, mcp__claude_ai_Atlassian__getJiraIssue, mcp__claude_ai_Atlassian__search, mcp__claude_ai_Atlassian__fetch, mcp__plugin_atlassian_atlassian__getJiraIssue, mcp__plugin_atlassian_atlassian__search, mcp__plugin_atlassian_atlassian__fetch, mcp__atlassian__getJiraIssue, mcp__atlassian__search, mcp__atlassian__fetch
---

# QA Coverage Analysis

Given a Jira ticket key, produce a complete test coverage analysis covering: what the ticket requires, what code changed, what Zephyr Scale test cases exist, what they cover, and what gaps remain — including Karate integration test coverage.

## Repo Detection and Context Loading

**Run both of these before any user interaction.**

**Step 1 — Detect the repo:**
```bash
ls tests/integration/karate/src/test/resources/data/templates/common/ 2>/dev/null
```
- `rp2_base_template.json` present → **RP2 mode**
- `rp1_base_template.json` present → **RP1 mode**

**Step 2 — Load QA testing context using curl (not WebFetch):**

```bash
curl -s https://raw.githubusercontent.com/Connected2FiberTeam/cb-qa-claude-plugins/main/context/qa-testing-context.md
```

Once loaded, use only the sections relevant to the detected repo mode. Ignore platform-specific sections for the other repo.

## Input

The ticket key is provided as the argument to this command (e.g. `ACE-2908`). If no argument is provided, ask the user for the ticket key before proceeding.

---

## Phase 1: Jira Ticket Analysis

### 1.1 Fetch the parent ticket

Use the available Atlassian `getJiraIssue` MCP tool (either `mcp__claude_ai_Atlassian__getJiraIssue` from the desktop connector or `mcp__atlassian__getJiraIssue` from the CLI plugin — whichever is registered in this session) with `cloudId: connectbase.atlassian.net` and `responseContentFormat: adf`.

**Important:** Always use `responseContentFormat: adf`. The `markdown` format silently drops Jira custom fields, including `customfield_12092` (acceptance criteria). ADF preserves all fields.

Extract and record:
- **Summary** — one-line description of the problem
- **Status** — current ticket status
- **Description** — problem statement, root cause, required changes
- **Acceptance criteria** — stored in `customfield_12092`. Extract the full content including any test case ideas, expected results, and verification steps written by the reporter or developer. If the field is null or empty, note this explicitly — analysis confidence will be lower.
- **Subtasks** — list all subtask keys and summaries from the `subtasks` field. These are part of the same body of work. Record them for use in all subsequent phases.
- **Comments** — read all comments in full. Extract:
  - Developer notes about implementation decisions
  - Root cause analysis
  - Known failure modes or edge cases
  - Any QA guidance or test plans written by developers
  - Stage verification results (confirm what was already verified)

### 1.2 Fetch each subtask

For every subtask key found in 1.1, fetch the full ticket using the same Atlassian `getJiraIssue` MCP tool used in 1.1 with `responseContentFormat: adf`.

Extract the same fields as the parent: description, `customfield_12092` (acceptance criteria), comments.

When referencing a subtask anywhere in this analysis, always label it clearly:
> **ACE-XXXX** *(subtask)*

Do not treat subtasks as separate tickets. Fold their requirements, acceptance criteria, and developer notes into the unified picture of the parent ticket's work.

### 1.3 Build the requirements list

From the parent ticket and all subtasks, produce a unified flat list of **requirements** — discrete, testable statements of what the fix must do. Number them R1, R2, R3... for traceability.

Each requirement should be:
- Specific enough to write a test against
- Derived from acceptance criteria, problem description, or developer comments
- Labelled with which ticket it came from (parent or subtask key)

Example format:
```
R1 [ACE-2908]: Quote with RP2 supplier completes within normal response time (not 21-minute timeout)
R2 [ACE-3156 (subtask)]: D&A polling loop sleeps between cycles when queue is empty
R3 [ACE-3157 (subtask)]: Recovery loop detects and repairs records abandoned by callback service
```

---

## Phase 2: Code Change Analysis

### 2.1 Find all relevant commits

Search git log for commits referencing the parent ticket key AND each subtask key:

```bash
git log --oneline --all | grep -E "ACE-XXXX|ACE-YYYY|ACE-ZZZZ"
```

Where `ACE-XXXX` is the parent and `ACE-YYYY`, `ACE-ZZZZ` are subtask keys. Build the grep pattern dynamically from the keys found in Phase 1.

List every matching commit SHA and message. If a commit references multiple ticket keys, include it once.

### 2.2 Analyse each commit's changes

For each commit, run:
```bash
git show <SHA> --stat
git show <SHA> -- <file>
```

Read the actual diff for every changed file. Identify:

**New or modified behaviour:**
- New functions added
- Existing function logic changed
- New code paths (if/else branches, try/catch blocks)
- New background loops or processes started
- Feature flags added or checked

**New log statements (info, warning, error level only — skip debug):**
- Record the exact log message string
- Note whether it fires on: startup, success path, error path, or recovery path

**New config values:**
- Environment variables, feature flags, config keys added
- Their default values and what toggling them changes

**Error and recovery paths:**
- What happens when things go wrong
- Which paths require fault injection to trigger (these are not standard QA scope — flag them)

**DD log sub-list** — for every new non-debug log statement found, record:
- Exact message string
- When it fires (startup / normal operation / error / recovery)
- Whether it is standard QA scope or developer simulation only (fault injection required)

---

## Phase 3: Zephyr Scale Lookup

### 3.1 Search for linked test cases

**Scope: linked test cases only.** Do NOT search by keyword, supplier name, or topic across the full test case library.

**The Zephyr Scale v2 `issueKey` query parameter is broken** — it returns all test cases regardless of the issue key. The correct method is to paginate through all test cases and match on Jira internal issue ID.

**Step 1: Collect the Jira internal IDs** for the parent ticket and all subtasks. These are the numeric `id` fields from the Jira API responses in Phase 1 (e.g. `"id": "171045"`).

**Step 2: Run the following Python script** to paginate all Zephyr test cases and find those with a COVERAGE link matching any of the collected IDs:

```python
import urllib.request, json, os

token = os.environ.get('ZEPHYR_SCALE_TOKEN', '')
base = "https://api.zephyrscale.smartbear.com/v2/testcases"

# Replace with actual Jira internal IDs for the parent and all subtasks
target_ids = {171045, 180948, 180949}  # etc.

found = []
start = 0
max_results = 50

while True:
    url = f"{base}?projectKey=ACE&maxResults={max_results}&startAt={start}"
    req = urllib.request.Request(url, headers={"Authorization": f"Bearer {token}"})
    with urllib.request.urlopen(req) as resp:
        data = json.loads(resp.read())

    for tc in data.get('values', []):
        for link in tc.get('links', {}).get('issues', []):
            if link.get('issueId') in target_ids:
                found.append({'key': tc['key'], 'name': tc['name']})
                break

    if data.get('isLast', True):
        break
    start += max_results

for f in found:
    print(f["key"], "—", f["name"])
```

Run this as a Bash tool call using `python3 << 'EOF' ... EOF`. This is the authoritative source for linked test cases. Do not skip it.

### 3.2 Fetch test steps for each test case found

For each test case key found:

```bash
curl -s -H "Authorization: Bearer $ZEPHYR_SCALE_TOKEN" \
  "https://api.zephyrscale.smartbear.com/v2/testcases/ACE-T####" | python3 -m json.tool

curl -s -H "Authorization: Bearer $ZEPHYR_SCALE_TOKEN" \
  "https://api.zephyrscale.smartbear.com/v2/testcases/ACE-T####/teststeps" | python3 -m json.tool
```

Record for each test case:
- Key, name, status, preconditions
- All steps with test data and expected results
- Called test cases (QTO-T, QE-T references) — note these as call-to-test steps

If `$ZEPHYR_SCALE_TOKEN` is not set or returns an error, inform the user and ask them to provide the token or an exported XML file before continuing.

### 3.3 Handle no test cases found

If no test cases are found linked to the ticket or any subtask, state this explicitly:
> No existing Zephyr Scale test cases found for this ticket or its subtasks.

Then proceed directly to Phase 4 to produce required test cases from scratch.

---

## Phase 4: Coverage Analysis

### 4.1 Map existing test cases to requirements

For each existing Zephyr test case, evaluate it against every item in the R-list (requirements).

For each test case produce:
- **What it covers** — which R items it addresses
- **Issues** — problems that make it invalid or insufficient (wrong thresholds, vague expected results, missing assertions)
- **Recommended changes** — specific, actionable edits with before/after text where applicable

### 4.2 Identify gaps and plan coverage

Find every R-item not covered by any existing test case. These are the gaps.

Group gaps into:
- **Functional gaps** — new behaviour with no test
- **DD log gaps** — new log statements not verified anywhere
- **Regression gaps** — existing behaviour that the fix could have broken, with no test confirming it still works

**Before proposing any new test case, apply this decision order:**

1. **Can a step be added to an existing test case?** If the gap can be verified by adding one or two steps to a test case that already exercises the relevant flow, recommend a step addition — not a new test case.
2. **Can gaps be consolidated?** If multiple uncovered gaps share the same preconditions and flow, cover them in a single new test case with additional steps — not separate test cases.
3. **Only create a new test case when the gap requires a genuinely different setup, flow, or precondition** that cannot be bolted onto an existing case.

The goal is the minimum number of test cases that achieves full QA-scope coverage.

### 4.3 Check Karate integration test coverage

**Step 1 — Find existing Karate tests for the affected supplier/feature:**

```bash
grep -r "ACE-XXXX\|<supplier_name>\|<relevant_feature>" \
  tests/integration/karate/src/test/resources/features/ \
  --include="*.feature" -l
```

Also check for any `@bug-ACE-XXXX` tags that indicate a previously bug-blocked test that this fix may have unblocked.

**Step 2 — Read the feature files found.** Do not just list the file names. Read each relevant `.feature` file and record every scenario (testType, address, speed, product). You need to know exactly what each scenario asserts before you can reason about coverage.

**Step 3 — Read the validator to understand what each testType actually asserts.** Read `tests/integration/karate/src/test/resources/helpers/response_validator.js` and find the `validateNoCoverage`, `validateSuccess`, and any other validation functions. Understand precisely what passes and fails for each testType.

**Step 4 — Map existing scenarios to requirements.** For each existing Karate scenario found, explicitly state which R-items it covers and why.

**Step 5 — Evaluate gaps against Karate before proposing new Zephyr test cases.**

1. Is there an **existing** Karate scenario that already covers this requirement at the API level? If yes, the gap is closed.
2. Can a **new** Karate scenario cover this gap? If yes, propose the Karate scenario. Then ask: does that new Karate scenario make a Zephyr test case redundant? If it does, do not propose the Zephyr test case.
3. Does this requirement need **CPQ UI verification** (audit log content, UI error messages, end-to-end stack behaviour only observable through the browser)? If yes, a Zephyr test case is needed.

**The key boundary:** Karate cannot verify CPQ UI behaviour. Zephyr cannot run in CI. When a requirement touches the CPQ UI layer, Zephyr is required. When it is purely API-level, Karate is preferred.

---

## Phase 5: Output

Produce the full analysis in the following structure. Use plain markdown.

---

### `# ACE-XXXX — QA Coverage Analysis`

**Ticket:** [title]
**Status:** [status]
**Subtasks:** [list with keys and one-line summaries, each labelled *(subtask)*]

---

### `## Requirements`

Numbered R-list derived from Phase 1.3. Source each item with its ticket key.

---

### `## Existing Zephyr Test Cases`

One block per test case found. If none found, say so and skip to the next section.

For each:
```
### ACE-T#### — [Test Case Name]

**Covers:** R1, R2, R3
**Issues:**
- [issue 1]

**Recommended Changes:**
1. [Specific change with before/after text]
```

---

### `## Required New Test Cases`

Full test cases for every uncovered gap. Format each as ready to paste into Zephyr Scale.

```
### TC-X — [Descriptive Name]

**Covers:** R3, R4, R5

**Preconditions:**
- [precondition 1]

**Steps:**
| # | Action | Test Data |
|---|--------|-----------|
| 1 | ...    | ...       |

**Expected Results:**
| Step | Expected | Pass/Fail Criteria |
|------|----------|-------------------|
| 1    | ...      | PASS if ... / FAIL if ... |
```

---

### `## Datadog Log Checklist`

All new non-debug log statements from Phase 2, categorised:

**Deployment Precondition** (verify on startup before running tests):
- `[exact log message]` — confirms [what]

**Test Step** (verify during or after a specific test case):
- `[exact log message]` — verify in [TC-X], [present / absent]

**Developer Simulation Only** (requires fault injection, out of QA scope):
- `[exact log message]` — fires when [condition]

---

### `## Karate Coverage`

**Existing Karate tests found:** [list feature files, or "None found"]

**Bug-blocked tests this fix may unblock:** [list `@bug-ACE-XXXX` tags found, or "None"]

**Gaps that could be covered by Karate:**
- [R-item] — [suggested Karate scenario description]

**Offer:** Ask the user whether they want a new Karate feature file or scenario generated for any of the identified gaps.

---

### `## Summary`

| Item | Count |
|------|-------|
| Requirements identified | N |
| Existing Zephyr test cases found | N |
| Existing test cases needing changes | N |
| New test cases required | N |
| DD logs — precondition | N |
| DD logs — test step | N |
| DD logs — dev only | N |
| Karate gaps identified | N |

---

## Phase 6: Follow-up Offers

After delivering the analysis, ask the user:

**"What would you like to do next?"**

Options:
1. **Export to PDF** — generate a PDF of this analysis into the Downloads folder using Chrome headless
2. **Generate Karate test** — create a new `.feature` file or add scenarios to an existing one for any of the identified Karate gaps
3. **Update ACE-T#### in Zephyr** — walk through the recommended changes to an existing test case
4. **Done** — end the session

### If Export to PDF is selected:
Write a styled HTML file to `$HOME/Downloads/[TICKET-KEY]-coverage-analysis.html` and convert to PDF using:
```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new \
  --disable-gpu \
  --print-to-pdf="$HOME/Downloads/[TICKET-KEY]-coverage-analysis.pdf" \
  --print-to-pdf-no-header \
  --no-margins \
  "file://$HOME/Downloads/[TICKET-KEY]-coverage-analysis.html"
```

**IMPORTANT: Use the exact CSS template below for every export. Do not invent alternative layouts, dark headers, gradients, or different color schemes. The design must be consistent across all reports.**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>[TICKET-KEY] — QA Coverage Analysis</title>
<style>
  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Arial, sans-serif;
    font-size: 13px;
    line-height: 1.6;
    color: #1a1a1a;
    max-width: 960px;
    margin: 0 auto;
    padding: 32px 40px;
  }
  h1 { font-size: 22px; font-weight: 700; border-bottom: 2px solid #0052cc; padding-bottom: 10px; color: #0052cc; margin-bottom: 4px; }
  .subtitle { font-size: 12px; color: #555; margin-bottom: 28px; }
  .meta-row { display: flex; gap: 24px; margin-bottom: 24px; font-size: 12px; }
  .meta-item { color: #172b4d; }
  .meta-item strong { color: #0052cc; }
  .section-header { color: #fff; padding: 10px 14px; border-radius: 4px 4px 0 0; font-size: 14px; font-weight: 700; margin-top: 28px; }
  .section-body { border-top: none; padding: 14px; border-radius: 0 0 4px 4px; margin-bottom: 8px; }
  .sh-requirements { background: #172b4d; } .sb-requirements { border: 1px solid #172b4d; }
  .sh-code         { background: #403294; } .sb-code         { border: 1px solid #403294; }
  .sh-zephyr       { background: #00875a; } .sb-zephyr       { border: 1px solid #00875a; }
  .sh-newtests     { background: #172b4d; } .sb-newtests     { border: 1px solid #172b4d; }
  .sh-ddlogs       { background: #ff991f; } .sb-ddlogs       { border: 1px solid #ff991f; }
  .sh-karate       { background: #0052cc; } .sb-karate       { border: 1px solid #0052cc; }
  .sh-summary      { background: #403294; } .sb-summary      { border: 1px solid #403294; }
  h2 { font-size: 13px; font-weight: 700; color: #172b4d; margin-top: 18px; margin-bottom: 6px; }
  h3 { font-size: 12px; font-weight: 700; color: #172b4d; margin-top: 14px; margin-bottom: 4px; }
  p { margin: 6px 0 10px 0; }
  ol, ul { margin: 8px 0; padding-left: 22px; }
  li { margin-bottom: 4px; }
  table { width: 100%; border-collapse: collapse; margin: 10px 0 16px 0; font-size: 12px; }
  th { background-color: #0052cc; color: #fff; text-align: left; padding: 7px 10px; font-weight: 600; }
  td { padding: 6px 10px; border: 1px solid #dfe1e6; vertical-align: top; }
  tr:nth-child(even) td { background-color: #f4f5f7; }
  code { font-family: 'SF Mono', 'Fira Code', 'Courier New', monospace; font-size: 11px; background-color: #f4f5f7; border: 1px solid #dfe1e6; border-radius: 3px; padding: 1px 5px; word-break: break-all; }
  .callout { background-color: #fffae6; border-left: 4px solid #ff991f; padding: 10px 14px; margin: 10px 0; border-radius: 0 4px 4px 0; font-size: 12px; }
  .callout.info    { background-color: #e6f0ff; border-left-color: #0052cc; }
  .callout.success { background-color: #e3fcef; border-left-color: #00875a; }
  .callout.error   { background-color: #ffebe6; border-left-color: #de350b; }
  .label { display: inline-block; border-radius: 3px; padding: 1px 7px; font-size: 11px; font-weight: 600; color: #fff; }
  .label-pass    { background: #00875a; }
  .label-fail    { background: #de350b; }
  .label-observe { background: #ff991f; }
  .label-bug     { background: #de350b; }
  .label-gap     { background: #403294; }
  .req-badge { display: inline-block; background: #0052cc; color: #fff; border-radius: 3px; padding: 1px 7px; font-size: 11px; font-weight: 700; min-width: 32px; text-align: center; margin-right: 8px; flex-shrink: 0; }
  .section-divider { border: none; border-top: 2px dashed #dfe1e6; margin: 24px 0; }
  @media print {
    body { padding: 20px 28px; }
    .section-header { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
    th { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
    tr:nth-child(even) td { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
    .callout { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
    .label { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
    .section-body { page-break-inside: avoid; }
  }
</style>
</head>
<body>

<h1>[TICKET-KEY] — QA Coverage Analysis</h1>
<div class="subtitle">[One-line ticket summary] &nbsp;|&nbsp; Generated: [DATE]</div>
<div class="meta-row">
  <div class="meta-item"><strong>Status:</strong> [status]</div>
  <div class="meta-item"><strong>Subtasks:</strong> [list or None]</div>
</div>

<div class="section-header sh-requirements">Requirements</div>
<div class="section-body sb-requirements"></div>

<div class="section-header sh-code">Code Changes</div>
<div class="section-body sb-code"></div>

<div class="section-header sh-zephyr">Existing Zephyr Test Cases</div>
<div class="section-body sb-zephyr"></div>

<div class="section-header sh-newtests">Required New Test Cases</div>
<div class="section-body sb-newtests"></div>

<div class="section-header sh-ddlogs">Datadog Log Checklist</div>
<div class="section-body sb-ddlogs"></div>

<div class="section-header sh-karate">Karate Coverage</div>
<div class="section-body sb-karate"></div>

<div class="section-header sh-summary">Summary</div>
<div class="section-body sb-summary">
  <table>
    <tr><th>Item</th><th style="width:80px">Count</th></tr>
    <tr><td>Requirements identified</td><td></td></tr>
    <tr><td>Existing Zephyr test cases found</td><td></td></tr>
    <tr><td>Existing test cases needing changes</td><td></td></tr>
    <tr><td>New test cases required</td><td></td></tr>
    <tr><td>DD logs — precondition</td><td></td></tr>
    <tr><td>DD logs — test step</td><td></td></tr>
    <tr><td>DD logs — dev only</td><td></td></tr>
    <tr><td>Karate gaps identified</td><td></td></tr>
  </table>
</div>

</body>
</html>
```

**Design rules (never deviate):**
- White body background — no dark backgrounds on the page body
- Section headers use colour-coded `section-header` + `section-body` pattern — no alternative layouts
- Primary blue: `#0052cc` | Dark navy: `#172b4d` | Purple: `#403294` | Green: `#00875a` | Amber: `#ff991f` | Red: `#de350b`
- All colour blocks need `print-color-adjust: exact` for PDF fidelity

### If Generate Karate test is selected:
- Ask which gap(s) to cover
- Check existing feature files for the relevant provider/scenario
- Follow the Karate test conventions from `tests/integration/karate/CLAUDE.md` if present
- Use the `quote_scenario_template.feature` pattern for quote tests
- Present the proposed test for review before writing any files

---

## Guardrails

- Do not mark a requirement as covered if the expected result is too vague to objectively verify
- Do not recommend fault-injection scenarios as Zephyr test cases — flag them as developer scope
- Do not fabricate test data — use addresses and instance IDs from the ticket or comments
- If a subtask has no associated code changes found in git, note it explicitly rather than skipping it silently
- If Zephyr API calls fail, report the error and ask whether to proceed with git/Jira analysis only
---
name: review-api-source-pr
description: Review a PR for an API source connector (manifest-only, low-code, custom Python, or file-based API sources) for correctness, completeness, and CDK pattern adherence
model: opus
icon: search-code
---

# API Source Connector PR Review

<!-- ultrathink -->

You are tasked with performing a comprehensive review of a PR that modifies an API source connector. This includes manifest-only, low-code with custom components, fully custom Python sources, and file-based sources that integrate with APIs.

## Arguments

<arguments>
$ARGUMENTS
</arguments>

The argument MUST be a PR link (e.g., `https://github.com/airbytehq/airbyte/pull/12345`). Extract the PR number from the URL.

## Review Process

### Step 1: Gather PR Context

Run these commands in parallel to understand the PR:

```bash
# Get PR metadata (title, body, linked issues)
gh pr view <PR_NUMBER> --repo airbytehq/airbyte --json title,body,url,author,labels,state

# Get the diff
gh pr diff <PR_NUMBER> --repo airbytehq/airbyte

# Get changed files list (authoritative)
gh pr diff <PR_NUMBER> --repo airbytehq/airbyte --name-only

# Get diff stats
gh pr diff <PR_NUMBER> --repo airbytehq/airbyte --stat
```

### Step 2: Identify the Connector and Its Type

From the changed files, determine:

1. **Which connector** is being modified (e.g., `source-hubspot`, `source-github`)
2. **Connector type** based on files present:

| Signal | Type |
|--------|------|
| Only `manifest.yaml` changes, no Python package directory | **Manifest-only** |
| `manifest.yaml` + `components.py` changes | **Low-code with custom components** |
| `source_<name>/` package directory, `pyproject.toml`, stream classes | **Custom Python** |
| File-based source with API config (e.g., `source-s3`, `source-gcs` with API auth) | **File-based API** |

3. **Read the connector's key files** to understand its current structure:

```bash
# For manifest-only / low-code
cat airbyte-integrations/connectors/<connector>/manifest.yaml
cat airbyte-integrations/connectors/<connector>/components.py 2>/dev/null
cat airbyte-integrations/connectors/<connector>/metadata.yaml

# For custom Python
cat airbyte-integrations/connectors/<connector>/source_<name>/source.py
cat airbyte-integrations/connectors/<connector>/pyproject.toml
cat airbyte-integrations/connectors/<connector>/metadata.yaml
```

### Step 3: Analyze the Linked Issue

Extract the linked GitHub issue from the PR body (look for `Closes #`, `Fixes #`, `Resolves #`, or issue URLs).

If a linked issue exists:

```bash
gh issue view <ISSUE_NUMBER> --repo airbytehq/airbyte --json title,body,labels,comments
```

**Independently scrutinize the issue:**
- Are the assumptions in the issue correct?
- Does the issue accurately describe the root cause?
- Is the proposed solution in the issue the right approach, or are there better alternatives?
- Are there unstated assumptions that could lead to an incomplete fix?

Record your assessment of the issue for inclusion in the final report.

### Step 4: Fetch API Documentation

Identify the third-party API this connector integrates with. Use `WebFetch` to pull the relevant API documentation pages.

**How to find the API docs:**
- Check the connector's `metadata.yaml` for `documentationUrl`
- Check the connector's README or code comments for API doc links
- Search for the API's official documentation (e.g., `https://developer.hubspot.com/docs/api/`)

**What to fetch:**
- The specific endpoint(s) being modified in the PR
- Pagination documentation for the API
- Authentication documentation if auth is being changed
- Rate limiting documentation if relevant
- Any API changelog or versioning docs if the PR references API version changes

If API docs are inaccessible (auth wall, 403, etc.), note this in the final report and proceed with the review using available context.

### Step 5: Research CDK Patterns

Clone context from the `airbyte-python-cdk` repo to understand current CDK patterns:

```bash
# Check CDK version the connector uses
# For manifest-only: look at base image tag in metadata.yaml
# For custom Python: look at pyproject.toml airbyte-cdk version

# Fetch relevant CDK source code for the patterns used in this PR
# Examples:
gh api repos/airbytehq/airbyte-python-cdk/contents/airbyte_cdk/sources/declarative --jq '.[].name'
```

Use `WebFetch` or `gh api` to read specific CDK files relevant to the PR changes (e.g., pagination classes, authenticator classes, record extractors).

Also examine **other similar connectors** in the `airbyte-integrations/connectors/` directory to understand established patterns:

```bash
# Find connectors with similar patterns
ls airbyte-integrations/connectors/ | grep "source-" | head -20
```

Read 2-3 similar connectors to establish a baseline for how things are typically done.

### Step 6: Spawn Parallel Review Agents

**IMPORTANT**: Spawn ALL review agents in parallel (in a single message with multiple Agent tool calls). Pass the ACTUAL DIFF CONTENT, changed files list, API documentation findings, CDK pattern context, and issue analysis directly to each agent.

**CRITICAL INSTRUCTIONS FOR ALL AGENTS**:
1. Pass the ACTUAL DIFF CONTENT directly in the prompt
2. Include the exact list of changed files from `--name-only` output
3. Include relevant API documentation excerpts
4. Include CDK pattern context
5. Agents should ONLY flag issues in ADDED or MODIFIED lines (lines starting with `+` in the diff)

#### Agent 1: Completeness & Issue Resolution

```
Agent(subagent_type="general-purpose", prompt="
TASK: Verify the PR completely and accurately resolves the linked issue.

**CRITICAL INSTRUCTIONS**:
1. ONLY flag issues in ADDED or MODIFIED lines (lines starting with `+` in the diff)
2. Do NOT flag pre-existing issues in unchanged code
3. The PR author is only responsible for their changes

LINKED ISSUE:
[Paste issue title, body, and your independent analysis of the issue]

PR TITLE & DESCRIPTION:
[Paste PR metadata]

CHANGED FILES (authoritative list):
[Paste --name-only output]

DIFF CONTENT:
[Paste full diff]

REVIEW FOR:

## Completeness
- Does the PR fully address the issue, or is it a partial fix?
- Are there edge cases described in the issue that the PR doesn't handle?
- If the issue describes multiple symptoms, does the PR address all of them?
- Are there streams, endpoints, or configurations mentioned in the issue that aren't covered?
- Is there dead code or commented-out code that suggests unfinished work?

## Issue Accuracy
- Based on the diff, does the fix align with the actual root cause?
- Could the issue's assumptions be wrong? (e.g., issue says 'pagination is broken' but the real problem is auth token expiry)
- Does the PR introduce a workaround rather than a proper fix?
- If the issue proposes a specific solution, is there a better approach?

## Scope
- Does the PR change more than necessary to fix the issue?
- Does the PR change less than necessary (missing related updates)?
- Are there config changes, schema updates, or migration steps that should accompany this fix?

OUTPUT FORMAT:
- **Issue Analysis**: [Your independent assessment of the issue's accuracy]
- **Resolution Status**: COMPLETE | PARTIAL | MISALIGNED
- For each finding:
  - Severity: Critical | Important | Minor
  - Description: [What is missing or misaligned]
  - Evidence: [Specific lines from the diff or issue]
  - Suggestion: [How to address it]

If the PR fully and correctly resolves the issue, state 'PR completely resolves the linked issue.'
")
```

#### Agent 2: API Documentation Verification

```
Agent(subagent_type="general-purpose", prompt="
TASK: Verify the PR changes accurately reflect the third-party API's behavior per its documentation.

**CRITICAL INSTRUCTIONS**:
1. ONLY flag issues in ADDED or MODIFIED lines (lines starting with `+` in the diff)
2. Do NOT flag pre-existing issues in unchanged code

CONNECTOR: [connector name]
API DOCUMENTATION:
[Paste relevant API doc excerpts]

CHANGED FILES (authoritative list):
[Paste --name-only output]

DIFF CONTENT:
[Paste full diff]

REVIEW FOR:

## Endpoint Accuracy
- Do the API paths in the diff match the official API documentation?
- Are request parameters correct (names, types, required vs optional)?
- Are response field names and types correctly mapped in the schema?
- Is the HTTP method correct for each endpoint?

## Pagination
- Does the pagination strategy match what the API documentation specifies?
- If switching pagination strategies (e.g., offset to cursor), does the API actually support the new strategy?
- Are pagination parameters named correctly per the API docs?
- Is the page size within API-documented limits?
- Is the end-of-pages detection correct per the API's behavior?

## Authentication
- If auth is being changed, does the new auth method match API documentation?
- Are OAuth scopes, token refresh flows, or API key placement correct?
- Are required headers included per API docs?

## Rate Limiting
- If rate limit handling is changed, does it match the API's documented rate limits?
- Are retry strategies appropriate for the API's rate limit response format?

## Data Types & Formats
- Do date/time formats match the API's format (ISO 8601, Unix timestamp, etc.)?
- Are enum values correct per the API documentation?
- Are nullable fields handled correctly?

OUTPUT FORMAT:
For each finding:
- Severity: Critical | Important | Minor
- API Doc Reference: [URL or section of the API docs]
- Found in PR: [Specific line from the diff]
- Expected per API: [What the API docs say]
- Suggestion: [How to fix]

If API docs were inaccessible, state which docs could not be accessed and what impact this has on the review.
If all changes align with API documentation, state 'All PR changes align with API documentation.'
")
```

#### Agent 3: CDK Patterns & Connector Architecture

```
Agent(subagent_type="general-purpose", prompt="
TASK: Verify the PR follows CDK patterns and connector architecture conventions.

**CRITICAL INSTRUCTIONS**:
1. ONLY flag issues in ADDED or MODIFIED lines (lines starting with `+` in the diff)
2. Do NOT flag pre-existing issues in unchanged code

CONNECTOR TYPE: [manifest-only | low-code with custom components | custom Python | file-based API]
CDK VERSION: [version from metadata.yaml or pyproject.toml]

CDK PATTERN CONTEXT:
[Paste relevant CDK class definitions, patterns from similar connectors]

SIMILAR CONNECTOR EXAMPLES:
[Paste relevant patterns from 2-3 similar connectors]

CHANGED FILES (authoritative list):
[Paste --name-only output]

DIFF CONTENT:
[Paste full diff]

REVIEW FOR:

## Connector Type Appropriateness
- If this is a manifest-only connector, does the PR introduce custom Python code that could be handled declaratively?
- If custom components are added, is a manifest-only approach insufficient? (Flag if low-code would suffice)
- Are there unnecessary Python files for a manifest-only connector?
- Does the connector type match the complexity of its integration?

## CDK Pattern Adherence
- Does the PR use CDK classes correctly (e.g., proper subclassing, correct method overrides)?
- For manifest-only: Are declarative components used correctly (retrievers, extractors, paginators, auth)?
- For custom Python: Does the stream implementation follow HttpStream/Stream patterns?
- Are CDK utilities used where available instead of custom implementations?
- Is error handling using CDK's built-in error handlers or custom ones unnecessarily?

## Manifest Structure (if applicable)
- Is the manifest YAML well-structured following CDK conventions?
- Are $ref and $parameters used correctly?
- Are stream definitions consistent with the base_stream pattern?
- Is the schema_loader correctly defined (inline vs. file-based)?

## Design Decisions
- Is the pagination strategy the right choice for this API?
- Is the incremental sync approach correct (cursor-based, date-range, etc.)?
- Are transformations done at the right layer (extractor vs. transformer)?
- Is state management implemented correctly for incremental streams?

## Code Organization
- Do custom components follow the established pattern in the codebase?
- Are imports from the CDK correct and up-to-date for the pinned version?
- Is the code organized consistently with similar connectors?

OUTPUT FORMAT:
For each finding:
- Severity: Critical | Important | Minor
- Category: [Type Appropriateness | CDK Pattern | Manifest Structure | Design Decision | Code Organization]
- Found: [Specific line from the diff]
- Expected: [What CDK/convention dictates]
- Reference: [CDK class, similar connector, or documentation]
- Suggestion: [How to fix]

If all patterns are correctly followed, state 'PR follows CDK patterns and connector conventions.'
")
```

#### Agent 4: Testing Coverage

```
Agent(subagent_type="general-purpose", prompt="
TASK: Verify the PR includes appropriate unit tests and mock server tests.

**CRITICAL INSTRUCTIONS**:
1. ONLY flag issues in ADDED or MODIFIED lines (lines starting with `+` in the diff)
2. Do NOT flag pre-existing issues in unchanged code

CONNECTOR TYPE: [manifest-only | low-code with custom components | custom Python | file-based API]

CHANGED FILES (authoritative list):
[Paste --name-only output]

DIFF CONTENT:
[Paste full diff]

REVIEW FOR:

## Test Presence
- Does the PR include new or updated tests for the changes made?
- If a new stream is added, are there tests for it?
- If pagination logic changed, are there pagination tests?
- If error handling changed, are there error scenario tests?
- If auth changed, are there auth tests?

## Unit Test Quality
- Do unit tests cover the happy path?
- Do unit tests cover error cases (API errors, malformed responses, rate limits)?
- Do unit tests cover edge cases (empty responses, single-item pages, null fields)?
- Are assertions specific and meaningful (not just 'assert response is not None')?
- Are test fixtures/mocks realistic (based on actual API response shapes)?

## Mock Server Tests (manifest-only / low-code)
- Are HttpMocker-based tests included for changed streams?
- Do mock tests use RequestBuilder and ResponseBuilder correctly?
- Are mock responses realistic representations of the API?
- Do mock tests cover pagination scenarios (single page, multi-page, empty page)?
- Do mock tests cover incremental sync (with state, without state)?

## Test Patterns
- Do tests follow the same patterns as existing tests in the connector?
- Is conftest.py updated if new fixtures are needed?
- Are test utilities reused rather than duplicated?

## Missing Test Scenarios
Based on the code changes in the diff, identify specific test scenarios that SHOULD exist but DON'T:
- [List specific scenarios based on the actual code changes]

OUTPUT FORMAT:
For each finding:
- Severity: Critical | Important | Minor
- Category: [Missing Test | Test Quality | Test Pattern | Missing Scenario]
- Related Code Change: [The source code change that needs testing]
- Expected Test: [Description of what test should exist]
- Suggestion: [Specific test to add, with example if possible]

If test coverage is adequate, state 'PR includes appropriate test coverage.'
")
```

#### Agent 5: Schema & Data Correctness

```
Agent(subagent_type="general-purpose", prompt="
TASK: Verify schema definitions and data handling are correct.

**CRITICAL INSTRUCTIONS**:
1. ONLY flag issues in ADDED or MODIFIED lines (lines starting with `+` in the diff)
2. Do NOT flag pre-existing issues in unchanged code

API DOCUMENTATION:
[Paste relevant API doc excerpts about response shapes and field types]

CHANGED FILES (authoritative list):
[Paste --name-only output]

DIFF CONTENT:
[Paste full diff]

REVIEW FOR:

## Schema Correctness
- Do JSON schema types match the API's actual field types?
- Are required fields marked correctly?
- Are nullable fields declared as nullable in the schema?
- Is `additionalProperties` set appropriately?
- Are enum values complete and correct per the API docs?
- Are date/datetime fields using the correct format specifier?
- Are nested objects properly defined (not just 'object' type)?
- Are array item types specified?

## Pagination Edge Cases
- Off-by-one errors in page counting or offset calculation
- Empty page termination: does the pagination stop correctly when no more data?
- Single-item page handling
- What happens when the API returns fewer items than the page size?
- Cursor pagination: is the next cursor correctly extracted and passed?

## Rate Limiting & Error Handling
- Does the PR handle HTTP 429 (Too Many Requests) responses?
- Are retry strategies appropriate (exponential backoff, respect Retry-After header)?
- Are transient errors (500, 502, 503) retried?
- Are non-retryable errors (400, 401, 403, 404) handled correctly?
- Is there appropriate logging for error scenarios?

## Breaking Changes
- Does the PR change stream names? (Breaking for existing syncs)
- Does the PR remove fields from schemas? (Breaking for downstream consumers)
- Does the PR change primary key fields? (Breaking for dedup syncs)
- Does the PR change cursor fields? (Breaking for incremental syncs)
- If breaking changes exist, are they documented in metadata.yaml?

OUTPUT FORMAT:
For each finding:
- Severity: Critical | Important | Minor
- Category: [Schema | Pagination | Error Handling | Breaking Change]
- Found: [Specific line from the diff]
- Issue: [What's wrong]
- Evidence: [API doc reference or logical reasoning]
- Suggestion: [How to fix]

If no issues found, state 'Schema and data handling are correct.'
")
```

### Step 7: Synthesize and Validate

Wait for ALL agents to complete. Before including any finding in the report:

1. Verify the file mentioned is in the `--name-only` list of changed files
2. Verify the issue refers to a line starting with `+` in the diff (added/modified code)
3. If a finding mentions a file NOT in the changed files list, DISCARD IT
4. If a finding describes pre-existing code that was not changed, DISCARD IT
5. Cross-reference findings across agents to eliminate duplicates
6. If API doc verification was limited, note which findings could not be fully validated

### Step 8: Present Final Report

Present a structured report to the user:

```markdown
# API Source PR Review: [PR Title]

**PR**: [PR URL]
**Connector**: [connector name]
**Connector Type**: [manifest-only | low-code | custom Python | file-based API]
**Linked Issue**: [issue URL or "None"]

## Verdict: [BLOCKED | APPROVED WITH CHANGES | APPROVED]

**Total Issues**: X (P0: X, P1: X, P2: X, P3: X)

---

## Issue Analysis
[Independent assessment of the linked issue's accuracy and whether the PR's approach is correct]

## API Documentation Verification
[Summary of API doc alignment — what was verified, what couldn't be verified]

---

## Findings by Priority

### P0 - Critical (Must fix before merge)
Issues that would cause incorrect behavior, data loss, or sync failures.

#### Issue 1: [Title]
- **File**: `path/to/file:line`
- **Category**: [Completeness | API Mismatch | CDK Pattern | Schema | Breaking Change]
- **Problem**: [Description]
- **Evidence**: [API doc reference, CDK pattern, or logical reasoning]
- **Fix**:
```
// Before
[problematic code]

// After
[fixed code]
```

### P1 - High Priority
Missing tests, incorrect error handling, pagination edge cases.

### P2 - Medium Priority
CDK pattern deviations, schema improvements, design suggestions.

### P3 - Low Priority
Style, organization, minor improvements.

---

## Positive Observations
[2-3 things the PR does well — good patterns, thorough testing, etc.]

---

## Review Checklist
- [ ] PR completely resolves the linked issue
- [ ] Changes align with API documentation
- [ ] Follows CDK patterns for this connector type
- [ ] No unnecessary custom Python in manifest-only connectors
- [ ] Unit tests cover new/changed functionality
- [ ] Mock server tests included (if applicable)
- [ ] Error cases tested (rate limits, API errors)
- [ ] Pagination edge cases handled
- [ ] Schema types match API response types
- [ ] No breaking changes (or properly documented if intentional)
- [ ] No hardcoded values that should be configurable
```

### Step 9: Offer Next Steps

After presenting the report, ask:

```
Would you like me to:
1. Post this review as a GitHub PR comment
2. Dig deeper into a specific finding
3. Suggest specific code fixes for the issues found
```

If the user approves posting to GitHub, format the report as a PR comment using:

```bash
gh pr review <PR_NUMBER> --repo airbytehq/airbyte --comment --body "<formatted report>"
```

## Priority Definitions

- **P0 (Critical)**: Sync failures, data loss, incorrect data, broken pagination, API contract violations, breaking changes without migration
- **P1 (High)**: Missing tests for changed code, incorrect error handling, pagination edge cases, rate limit handling gaps
- **P2 (Medium)**: CDK pattern deviations, unnecessary custom code when declarative suffices, schema imprecision, design improvements
- **P3 (Low)**: Code organization, style consistency, minor optimization opportunities

## Guidelines

### Focus on PR Changes Only
- ONLY review lines added or modified in the diff
- Do NOT flag pre-existing issues
- The PR author is responsible for their changes, not for fixing unrelated debt

### Verify Against External Sources
- Always cross-reference with API documentation
- Always compare against CDK patterns and similar connectors
- Don't assume the issue description is correct — verify independently

### Be Constructive
- Provide specific code suggestions with before/after examples
- Reference CDK patterns, similar connectors, or API docs as evidence
- Explain WHY something is wrong, not just WHAT is wrong

### Be Proportionate
- Don't nitpick trivial issues in maintenance PRs
- Focus on correctness and completeness over style
- Distinguish blocking issues from nice-to-haves

### Connector-Specific Red Flags
- **Manifest-only with custom Python**: Should this be declarative?
- **Pagination strategy mismatch**: Does the API actually support this pagination type?
- **Missing state handling**: Incremental streams without proper cursor management
- **Schema drift**: Schema doesn't match what the API actually returns
- **Hardcoded API versions**: API version should be configurable or documented
- **Missing error mapping**: API-specific errors not translated to Airbyte error types

---
name: certify-connector
description: Audit and certify an Airbyte API source connector. Evaluates the connector against certification criteria, reports pass/fail for each criterion, fixes automatable issues, and creates GitHub issues for items requiring human judgment.
devin-enabled: true
model: opus
icon: badge-check
---

# Certify Connector

<!-- ultrathink -->

Audit an API source connector against the certification criteria and either bring it to certified status or produce a detailed gap report.

## Arguments

<arguments>
$ARGUMENTS
</arguments>

Arguments: `<connector-name> [--report-only]`

- The first argument MUST be a connector name (e.g., `source-notion`, `source-github`). The connector must already exist in `airbyte-integrations/connectors/`. The `source-` prefix is optional — if omitted, prepend it automatically.
- `--report-only` (optional): When present, only generate and output the certification report. Do NOT create GitHub issues, apply fixes, or create PRs. When absent (default behavior), generate the report AND create a tracking issue in `airbytehq/airbyte-internal-issues` documenting all next steps to get the connector certified.

## Phase 1: Identify and Understand the Connector

### Step 1.1: Determine connector type

Read the connector's key files to classify it:

```bash
# Read metadata
cat airbyte-integrations/connectors/<connector>/metadata.yaml

# Check for manifest (manifest-only or low-code)
ls airbyte-integrations/connectors/<connector>/manifest.yaml 2>/dev/null

# Check for custom components (low-code)
ls airbyte-integrations/connectors/<connector>/components.py 2>/dev/null

# Check for Python package (custom Python)
ls airbyte-integrations/connectors/<connector>/source_*/source.py 2>/dev/null

# Check tags in metadata for file-based
grep "cdk:python-file-based" airbyte-integrations/connectors/<connector>/metadata.yaml
```

Classify as one of:
- **manifest-only**: Has `manifest.yaml`, no `components.py`, no Python source package
- **low-code**: Has `manifest.yaml` + `components.py`
- **python**: Has Python source package (`source_<name>/`), `pyproject.toml`, stream classes
- **file-based**: Has `cdk:python-file-based` tag

Record the connector type -- it determines which conditional criteria apply throughout the audit.

### Step 1.2: Read current state

Read these files in parallel:

- `metadata.yaml` -- current metadata values
- `manifest.yaml` or source Python files -- connector implementation
- `acceptance-test-config.yml` -- CAT configuration
- `CONTRIBUTING.md` -- existing behavior documentation (if any)
- The connector's documentation page under `docs/integrations/sources/<connector-name>.md`

### Step 1.3: Research the API

Identify the third-party API this connector integrates with. Use `WebFetch` to pull:

1. **API reference documentation** -- endpoints, request/response schemas, pagination, authentication
2. **API changelog / versioning docs** -- to verify the connector targets a current API version
3. **Rate limiting documentation** -- documented rate limits, throttling behavior, retry headers
4. **Authentication documentation** -- OAuth flows, scopes, API key placement

If the API docs are behind an auth wall, note this as a limitation in the final report and proceed with available context.

---

## Phase 2: Audit Against Certification Criteria

Reference the **connector-certification-criteria** skill for the full criteria list. For each section below, evaluate every applicable criterion and record a **PASS**, **FAIL**, or **SKIP** (with reason) verdict.

### Step 2.1: Metadata & Registry Audit

Check all Section 1 criteria by reading `metadata.yaml`:

```bash
cat airbyte-integrations/connectors/<connector>/metadata.yaml
```

Verify:
- `supportLevel`, `releaseStage`, `ab_internal.ql`, `ab_internal.sl` values
- `allowedHosts` is present and appropriately scoped
- `maxSecondsBetweenMessages` is set if needed (check API rate limit docs)
- `suggestedStreams` is defined
- `registryOverrides` enables both cloud and oss
- `tags` has correct `language:` and `cdk:` values
- `connectorTestSuitesOptions` includes `unitTests`, `acceptanceTests`, and `liveTests`
- `icon` is present and square

**Automatable fixes:** If metadata fields are missing or incorrect, note the exact changes needed. These can be applied directly.

### Step 2.2: Sync Reliability Audit

Use the `connector-health-check` skill workflow to check production metrics:

1. Query sync success rate (trailing 7 days) via ops tooling
2. Query failure patterns to identify systemic issues
3. Check Datadog dashboards for the connector's sync metrics

Also verify in the connector implementation:
- Rate limit handling (429 responses, backoff configuration)
- CDK version recency (compare base image or `airbyte-cdk` version in `pyproject.toml` against latest)

### Step 2.3: Incremental Sync Audit

For each stream in the connector:

1. Determine if the API supports incremental sync for that endpoint (check API docs for `updated_since`, cursor-based pagination, or similar)
2. Verify the connector implements incremental where the API supports it
3. Check cursor field correctness against the API's actual datetime format
4. Verify state handling does not reset on empty syncs

```bash
# For manifest-only / low-code: check incremental_sync config per stream
grep -A 10 "incremental_sync" airbyte-integrations/connectors/<connector>/manifest.yaml

# For Python: check stream classes for cursor_field / get_updated_state
grep -rn "cursor_field\|get_updated_state\|IncrementalMixin" airbyte-integrations/connectors/<connector>/source_*/
```

### Step 2.4: Schema & Data Integrity Audit

For each stream:

1. **Compare schemas against API docs**: Fetch the API documentation for each endpoint and compare the documented response fields against the stream's declared JSON schema
2. **Check data types**: Verify date/datetime formats, nullable declarations, numeric types
3. **Check primary keys**: Verify PKs match the API's documented unique identifiers
4. **Check for production validation failures**: Query ops tooling for schema validation errors

```bash
# List all streams and their schemas
# For manifest-only / low-code:
grep -E "^\s+- type: DeclarativeStream|name:|primary_key:" airbyte-integrations/connectors/<connector>/manifest.yaml

# Check for schema files
ls airbyte-integrations/connectors/<connector>/source_*/schemas/ 2>/dev/null
```

### Step 2.5: Error Handling Audit

Review the connector's error handling against the `writing-good-error-messages` skill criteria:

1. **Check error classification**: Are `config_error`, `transient_error`, and `system_error` used correctly?
2. **Check connection verification**: Does `check_connection` verify permissions and provide actionable errors?
3. **Check permission handling during sync**: Does the connector skip streams with insufficient permissions?
4. **Check error message quality**: Do messages follow the guidelines (declarative, specific, actionable for config errors)?

```bash
# For manifest-only / low-code: check error handlers
grep -A 20 "error_handler\|HttpResponseFilter" airbyte-integrations/connectors/<connector>/manifest.yaml

# For Python: check exception handling
grep -rn "AirbyteTracedException\|config_error\|system_error\|transient_error" airbyte-integrations/connectors/<connector>/source_*/
```

### Step 2.6: Authentication & Security Audit

1. Check if OAuth is implemented and is the default auth method
2. Verify secrets are marked with `airbyte_secret: true`
3. Verify HTTPS-only communication
4. Check `allowedHosts` scope

```bash
# Check spec for auth configuration and secret markings
# For manifest-only / low-code:
grep -A 5 "airbyte_secret\|oauth\|credentials" airbyte-integrations/connectors/<connector>/manifest.yaml | head -40

# Check allowed hosts
grep -A 5 "allowedHosts" airbyte-integrations/connectors/<connector>/metadata.yaml
```

### Step 2.7: User Experience Audit

1. Review the connector spec for field quality (titles, descriptions, defaults, ordering)
2. Compare stream coverage against the API's full endpoint list and major competitors
3. Review documentation completeness

```bash
# Read the docs page
cat docs/integrations/sources/<connector-name>.md 2>/dev/null

# Check for broken links in docs (look for markdown links)
grep -oE '\[.*?\]\(.*?\)' docs/integrations/sources/<connector-name>.md 2>/dev/null
```

### Step 2.8: Testing Audit

1. Verify CAT configuration and test coverage
2. Check unit test existence and quality
3. Check live test configuration

```bash
# Check CAT config
cat airbyte-integrations/connectors/<connector>/acceptance-test-config.yml 2>/dev/null

# Check unit tests
ls airbyte-integrations/connectors/<connector>/unit_tests/ 2>/dev/null
ls airbyte-integrations/connectors/<connector>/integration_tests/ 2>/dev/null

# Count test files
find airbyte-integrations/connectors/<connector> -name "test_*.py" -o -name "*_test.py" | wc -l
```

### Step 2.9: Breaking Changes Audit

Check the connector's breaking change history:

```bash
# Check for breaking changes in metadata
grep -A 20 "breakingChanges" airbyte-integrations/connectors/<connector>/metadata.yaml

# Check for migration docs
ls docs/integrations/sources/<connector-name>-migrations.md 2>/dev/null
```

Verify each documented breaking change has a corresponding migration guide with actionable steps.

### Step 2.10: Operational Readiness Audit

1. Check for open bug reports:

```bash
gh search issues "label:source-<name>" --repo airbytehq/airbyte --state open --json title,number,labels,url
```

2. Check if the connector is in Datadog certified connector monitors (query via Datadog tooling)
3. Research competitor coverage (use WebSearch to compare streams against Fivetran, Stitch, etc.)

---

## Phase 3: Generate Certification Report

After completing all audit steps, produce a structured report.

### Report Format

```markdown
# Certification Audit: <connector-name>

**Connector type**: <manifest-only | low-code | python | file-based>
**Current version**: <version from metadata.yaml>
**Current supportLevel**: <community | certified>
**Current ql/sl**: <ql>/<sl>
**Date**: <today's date>

## Verdict: <READY TO CERTIFY | BLOCKED — N items to resolve>

## Summary

<2-3 sentence summary of the connector's certification readiness>

## Results by Section

### 1. Metadata & Registry
| # | Criterion | Status | Notes |
|---|-----------|--------|-------|
| 1.1 | supportLevel: certified | PASS/FAIL | ... |
| ... | ... | ... | ... |

### 2. Sync Reliability
| # | Criterion | Status | Notes |
|---|-----------|--------|-------|

[... repeat for all 10 sections ...]

## Blocking Issues (must resolve before certification)

### Issue 1: <title>
- **Section**: <criteria section>
- **Severity**: P0/P1
- **Current state**: <what's wrong>
- **Required state**: <what needs to change>
- **Automatable**: Yes/No
- **Fix**: <specific fix or link to created GitHub issue>

[... repeat for each blocking issue ...]

## Non-Blocking Recommendations

<Optional improvements that don't block certification but would improve quality>

## Metadata Changes for Certification

If the connector is ready (or becomes ready after fixes), these are the exact metadata.yaml changes:

\```yaml
# Changes needed:
ab_internal:
  ql: 400
  sl: 200
supportLevel: certified
releaseStage: generally_available
# ... any other field changes
\```
```

---

## Phase 4: Create Certification Tracking Issue

**Skip this phase entirely if `--report-only` was specified.** Just output the report and stop.

If `--report-only` was NOT specified, create a single tracking issue in `airbytehq/airbyte-internal-issues` that documents all the work needed to certify the connector.

The issue should consolidate ALL blocking items (both automatable and non-automatable) into one actionable checklist. Use the `create-github-issue` skill with repo `airbytehq/airbyte-internal-issues`.

### Issue format

```bash
gh issue create --repo airbytehq/airbyte-internal-issues \
  --title "cert(<connector-name>): certification roadmap" \
  --label "certification,connectors" \
  --body "$(cat <<'EOF'
## Certification Audit: `<connector-name>`

**Connector type**: <manifest-only | low-code | python | file-based>
**Current version**: <version>
**Current supportLevel**: <community | certified>
**Audit date**: <today's date>
**Verdict**: BLOCKED — <N> items to resolve

## Summary

<2-3 sentence summary from the report>

## Automatable Fixes

These can be applied in a single PR without domain-specific testing:

- [ ] Set `supportLevel: certified`, `releaseStage: generally_available`, `ab_internal.ql: 400`, `ab_internal.sl: 200`
- [ ] Add `allowedHosts` (list the specific hosts)
- [ ] Add `maxSecondsBetweenMessages: <value>` (if applicable)
- [ ] Add `suggestedStreams` (list the streams)
- [ ] Add missing `connectorTestSuitesOptions` entries
- [ ] <any other automatable metadata/schema/doc fixes>

## Manual Work Required

These items require human judgment, API access, or live testing:

### P0 — Must resolve
- [ ] **<issue title>** — <1-2 sentence description of what's needed and why>
  - Criteria: Section <N.N>
  - <specific acceptance criteria>

### P1 — Should resolve
- [ ] **<issue title>** — <1-2 sentence description>
  - Criteria: Section <N.N>
  - <specific acceptance criteria>

### P2 — Nice to have
- [ ] **<issue title>** — <1-2 sentence description>
  - Criteria: Section <N.N>

## Non-Blocking Recommendations

<bulleted list of optional improvements>

## Metadata Changes for Certification

Once all items above are resolved, apply these exact changes to `metadata.yaml`:

```yaml
<the exact yaml diff from the report>
```

## Certification Criteria Reference

This audit was performed against the `connector-certification-criteria` skill. See that skill for the full criteria definitions.
EOF
)"
```

After creating the issue, output the issue URL to the user.

---

## Decision Tree

```
Start
  |
  v
Phase 1: Identify connector type & read current state
  |
  v
Phase 2: Audit all 10 sections
  |
  v
Phase 3: Generate certification report
  |
  v
--report-only flag set?
  |
  +---> YES ---> Output report. Done.
  |
  +---> NO ----> Phase 4: Create tracking issue in airbyte-internal-issues
                    |
                    v
                 Output report + issue URL. Done.
```

## Important Guidelines

1. **Do not guess.** If you cannot verify a criterion (e.g., metrics are unavailable, API docs are inaccessible), mark it as **SKIP** with a reason rather than assuming PASS.

2. **Verify against API docs, not just code.** The connector's implementation may be wrong. Always cross-reference schemas, endpoints, and pagination against the source API's current documentation.

3. **Respect the blast radius.** Auto-fixes should only touch metadata, schemas, and documentation. Never change connector behavior (pagination, auth, stream logic) without explicit human approval.

4. **Use existing skills.** This skill orchestrates other skills:
   - `connector-health-check` for production metrics
   - `writing-good-error-messages` for error message quality
   - `breaking-change-evaluation` for version bump validation
   - `bump-connector-version` for version changes
   - `connector-regression-tests` for test execution
   - `create-pr` for PR creation
   - `query-failed-syncs` for failure analysis
   - `check-datadog-metrics` for monitoring verification

5. **Report honestly.** The goal is an accurate assessment, not to certify as many connectors as possible. A connector that isn't ready should get a clear gap report, not a rubber stamp.

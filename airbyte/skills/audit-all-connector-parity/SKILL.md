---
name: audit-all-connector-parity
description: Audit parity for all ADP connectors in airbytehq/sonar against their DR sister connectors in airbytehq/airbyte. Runs audit-connector-parity on each connector and produces per-connector reports plus a summary. Use for fleet-wide parity assessment.
icon: git-compare
---

# Audit All ADP / DR Connector Parity

<!-- ultrathink -->

Run the `audit-connector-parity` skill on every ADP connector in the sonar repo, then produce per-connector report artifacts and a fleet-wide summary.

## Arguments

<arguments>
$ARGUMENTS
</arguments>

Optional arguments:
- **output directory**: Path to write report artifacts (default: `~/airbyte/docs/parity-reports/`)
- **connector list**: Comma-separated list of connector names to audit (default: all connectors found in sonar `integrations/`)

## Workflow

### Step 1: Discover All ADP Connectors

List all connector directories under `integrations/` in the sonar repo (at `~/airbyte/sonar` or via `gh api`).

```bash
ls ~/airbyte/sonar/integrations/
```

Each directory that contains a `connector.yaml` is an ADP connector. Collect the full list of connector names.

### Step 2: Create Output Directory

```bash
mkdir -p {output_directory}
```

Default: `~/airbyte/docs/parity-reports/`

### Step 3: Run Audits in Parallel

For each ADP connector, spawn a background agent that runs the `audit-connector-parity` workflow (Steps 1-5 from that skill). Run connectors in parallel batches to maximize throughput.

Each agent should:
1. Follow the full `audit-connector-parity` workflow for its assigned connector
2. Return the complete audit report as its output

**Important**: Pass the full audit instructions to each agent -- do not invoke the skill tool from within an agent. Instead, embed the audit logic directly in the agent prompt:

```
Agent(subagent_type="general-purpose", run_in_background=true, prompt="
You are auditing the ADP connector '{connector_name}' against its DR sister connector 'source-{connector_name}'.

SONAR REPO: ~/airbyte/sonar
AIRBYTE REPO: ~/airbyte/airbyte

STEP 1: Read the ADP connector at integrations/{connector_name}/connector.yaml in the sonar repo.
Extract: entities/actions (x-airbyte-entity, x-airbyte-action), auth methods (securitySchemes, x-airbyte-auth-config), API base URL/version, entity field definitions (x-airbyte-cache.entities[].fields), query parameters per operation.

STEP 2: Check if source-{connector_name} exists in airbyte-integrations/connectors/ in the airbyte repo.
If not found, report: 'No DR counterpart found' and stop.
If found, read metadata.yaml and extract data.supportLevel (certified or community) -- this is the DR certification status.
Also extract: streams, auth methods, API base URL/version, stream schemas (schemas/*.json or inline), query parameters per stream.

STEP 3: Compare across 4 dimensions:
1. Stream/Entity Coverage -- DR streams vs ADP entities (both directions). Note write-only ADP entities separately.
2. Authentication Parity -- Same auth methods? If gap, note if connector-sdk supports the missing type.
3. Schema/Field Coverage -- Field-by-field comparison for matched stream/entity pairs. Flag DR_ONLY, ADP_ONLY, TYPE_MISMATCH.
4. API Version & Query Parameters -- Same base URL/version? Same query params per matched pair? Exclude pagination params.

STEP 4: Produce the report in this exact format:

# ADP / DR Connector Parity Audit: {connector_name}

**ADP Connector**: `{connector_name}` (airbytehq/sonar)
**DR Connector**: `source-{connector_name}` (airbytehq/airbyte)
**DR Connector Support Level**: `{certified | community}` ← from `metadata.yaml` `data.supportLevel`
**Audit Date**: {today's date}

## DR Certification Status

**DR Connector Certified**: `{YES | NO}` ← `certified` if `data.supportLevel` is `certified`, otherwise `NO`

## Overall Parity Score

| Dimension | Status | Gaps |
|-----------|--------|------|
| DR Certification | YES/NO | -- |
| Stream/Entity Coverage | FULL/PARTIAL/LOW | count |
| Authentication | FULL/PARTIAL/LOW | count |
| Schema/Fields | FULL/PARTIAL/LOW | count |
| API Version | MATCH/MISMATCH | -- |
| Query Parameters | FULL/PARTIAL/LOW | count |

Then include the detailed findings for each dimension, priority action items (P0/P1/P2), and write-only ADP entities list.

Return the FULL report text as your output.
")
```

### Step 4: Collect Results and Write Artifacts

As each agent completes, write its report to a file:

```
{output_directory}/{date}_{connector_name}_parity_report.md
```

If a connector had no DR counterpart, still write a short report noting that.

### Step 5: Generate Summary Report

After all audits complete, produce a fleet-wide summary at `{output_directory}/{date}_summary.md`:

```markdown
# ADP / DR Connector Parity — Fleet Summary

**Audit Date**: {date}
**Connectors Audited**: {total_count}
**DR Counterpart Found**: {found_count}
**No DR Counterpart**: {missing_count}

## Parity Overview

| Connector | DR Exists | DR Certified | DR Support Level | Coverage | Auth | Schema | API Version | Params | Action Items |
|-----------|-----------|--------------|------------------|----------|------|--------|-------------|--------|--------------|
| {name} | YES/NO | YES/NO | certified/community | FULL/PARTIAL/LOW | FULL/PARTIAL/LOW | FULL/PARTIAL/LOW | MATCH/MISMATCH | FULL/PARTIAL/LOW | P0: X, P1: X, P2: X |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |

## Connectors Without DR Counterpart
- {list}

## Fleet-Wide Statistics
- **DR connector is certified**: X connectors
- **DR connector is NOT certified**: X connectors
- **Full parity across all dimensions**: X connectors
- **Has P0 (critical) gaps**: X connectors
- **Has P1 (important) gaps**: X connectors
- **Has P2 (minor) gaps only**: X connectors

## Top P0 Action Items Across Fleet
{aggregated list of all P0 items from all connector reports}

## Top P1 Action Items Across Fleet
{aggregated list of all P1 items from all connector reports}

## Individual Reports
{links to each per-connector report file}
```

### Step 6: Present Summary

Display the summary report to the user. Highlight:
- Connectors with P0 gaps that need immediate attention
- Connectors with no DR counterpart
- Overall fleet health

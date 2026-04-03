---
name: audit-connector-parity
description: Audit parity between an ADP (Agentic Data Platform) connector in airbytehq/sonar and its DR (Data Replication) sister connector in airbytehq/airbyte. Use when verifying that an ADP connector and its DR counterpart are aligned on streams/entities, authentication, schemas, API versions, and query parameters.
icon: git-compare
---

# Audit ADP / DR Connector Parity

<!-- ultrathink -->

Audit the alignment between an ADP connector (in `airbytehq/sonar`) and its DR sister connector (in `airbytehq/airbyte`). This skill assumes the ADP connector exists and checks whether a DR counterpart exists, then compares the two across multiple dimensions.

## Arguments

<arguments>
$ARGUMENTS
</arguments>

The argument MUST be the ADP connector name (e.g., `slack`, `stripe`, `granola`). This is the name as it appears under `integrations/` in the sonar repo.

## Prerequisites

This skill requires access to two repositories:

- **Sonar repo** (ADP connectors): `airbytehq/sonar` -- check for local clone first, fall back to GitHub API
- **Airbyte repo** (DR connectors): `airbytehq/airbyte` -- check for local clone first, fall back to GitHub API

To locate local clones, check these common paths in order:
1. `~/airbyte/sonar` and `~/airbyte/airbyte`
2. Sibling directories relative to the current working directory
3. If no local clone is found, use `gh api` to fetch files from GitHub

## Audit Workflow

### Step 1: Locate the ADP Connector

Find the ADP connector definition at `integrations/{connector_name}/connector.yaml` in the sonar repo. This is an OpenAPI 3.1.0 spec with Airbyte-specific extensions.

Extract from the ADP connector:
- **Entities and actions**: Parse all `x-airbyte-entity` and `x-airbyte-action` annotations on each operation. Build a map of `entity -> [actions]`.
- **Authentication methods**: Look at the `securitySchemes` and `x-airbyte-auth-config` sections. Record each auth type (Bearer, OAuth2, API Key, Basic).
- **API base URL and version**: From `servers[0].url` and any version path segments or headers.
- **Entity field definitions**: From `x-airbyte-cache.entities[].fields` for each entity.
- **Query parameters**: From each operation's `parameters` section.
- **Cache config**: From `x-airbyte-cache`.

### Step 2: Locate the DR Connector

Look for `source-{connector_name}` in `airbyte-integrations/connectors/` in the airbyte repo.

**If the DR connector does not exist**: Stop and report:
```
## Result: No DR Counterpart Found

ADP connector `{connector_name}` has no DR sister connector `source-{connector_name}` in airbytehq/airbyte.
```

**If the DR connector exists**, determine its type (manifest-only, low-code, or custom Python) and extract:
- **Streams**: All stream definitions from `manifest.yaml` or Python source files.
- **Authentication methods**: From the authenticator configuration.
- **API base URL and version**: From `url_base` or `requester.url_base`.
- **Stream schemas**: From `schemas/*.json` or inline schema definitions.
- **Query parameters**: From `request_parameters` on each stream/requester.

### Step 3: Spawn Parallel Audit Agents

Spawn ALL audit agents in a single message for parallel execution. Pass the extracted ADP and DR connector data to each agent.

#### Agent 1: Stream / Entity Coverage

```
Agent(subagent_type="general-purpose", prompt="
TASK: Compare stream and entity coverage between the ADP and DR connectors.

ADP CONNECTOR: {connector_name}
ADP ENTITIES AND ACTIONS:
{entity_action_map}

DR CONNECTOR: source-{connector_name}
DR STREAMS:
{stream_list}

COMPARE:

## DR -> ADP Coverage
For each DR stream, determine if the ADP connector has a corresponding entity with a `list` or `api_search` action.
- Match by name (exact or fuzzy -- e.g., `channel_messages` matches `channel_messages`)
- Match by API endpoint (if names differ but they hit the same endpoint)
- Flag any DR stream with no ADP entity counterpart

## ADP -> DR Coverage
For each ADP entity with a `list` or `api_search` action, determine if the DR connector has a corresponding stream.
- Note: ADP entities that ONLY have write actions (create, update, delete) with no list/api_search are expected to have no DR counterpart. List these separately as 'Write-only ADP entities (no DR equivalent expected)'.
- Flag any ADP list/search entity with no DR stream counterpart

OUTPUT FORMAT:
### Stream/Entity Coverage Matrix

| DR Stream | ADP Entity | ADP Actions | Status |
|-----------|-----------|-------------|--------|
| ... | ... | ... | MATCHED / MISSING_IN_ADP / MISSING_IN_DR |

### Write-Only ADP Entities (No DR Equivalent Expected)
- {entity}: {actions}

### Gaps
- **DR streams missing in ADP**: [list]
- **ADP list/search entities missing in DR**: [list]
")
```

#### Agent 2: Authentication Parity

```
Agent(subagent_type="general-purpose", prompt="
TASK: Compare authentication methods between the ADP and DR connectors, and check SDK support for any gaps.

ADP CONNECTOR: {connector_name}
ADP AUTH METHODS:
{adp_auth_details}

DR CONNECTOR: source-{connector_name}
DR AUTH METHODS:
{dr_auth_details}

CONNECTOR-SDK SUPPORTED AUTH TYPES:
- API Key (APIKeyAuthStrategy)
- Bearer Token (BearerAuthStrategy)
- HTTP Basic Auth (BasicAuthStrategy)
- OAuth 2.0 (OAuth2AuthStrategy) -- supports refresh token flow, configurable auth styles

COMPARE:

## Auth Method Parity
For each auth method in either connector:
- Is it present in both? If not, which side is missing it?
- For OAuth: are the same scopes configured?
- For API Key/Bearer: is the header name and format the same?

## SDK Support for Gaps
If the DR connector supports an auth method that the ADP connector lacks:
- Check if the connector-sdk (in sonar) supports that auth type
- If not supported by connector-sdk, check if the airbyte-cdk (airbyte-python-cdk) supports it
- Flag whether implementing the missing auth requires SDK-level work

OUTPUT FORMAT:
### Authentication Comparison

| Auth Method | DR | ADP | Parity | Notes |
|-------------|------|------|--------|-------|
| ... | ... | ... | MATCH / GAP | ... |

### OAuth Scope Comparison (if applicable)
| Scope | DR | ADP |
|-------|------|------|

### SDK Support for Missing Auth Types
- {missing_type}: Connector-SDK support: YES/NO, CDK support: YES/NO
")
```

#### Agent 3: Schema / Field Coverage

```
Agent(subagent_type="general-purpose", prompt="
TASK: Compare field-level schema definitions between matched ADP entities and DR streams.

For each matched stream/entity pair, compare the fields defined in the DR JSON schema vs. the ADP entity field definitions.

ADP CONNECTOR: {connector_name}
ADP ENTITY FIELDS:
{adp_entity_fields}

DR CONNECTOR: source-{connector_name}
DR STREAM SCHEMAS:
{dr_stream_schemas}

COMPARE:

For each matched pair:
- Fields present in DR but missing in ADP (potential data loss for ADP users)
- Fields present in ADP but missing in DR (ADP may expose more data)
- Type mismatches between DR and ADP for the same field
- Nested object structure differences

OUTPUT FORMAT:
### Schema Comparison: {entity/stream name}

| Field | DR Type | ADP Type | Status |
|-------|---------|----------|--------|
| ... | ... | ... | MATCH / DR_ONLY / ADP_ONLY / TYPE_MISMATCH |

### Summary
- **Fields in DR but not ADP**: X fields across Y entities
- **Fields in ADP but not DR**: X fields across Y entities
- **Type mismatches**: X fields
")
```

#### Agent 4: API Version & Query Parameter Parity

```
Agent(subagent_type="general-purpose", prompt="
TASK: Compare API versions and query parameters between the ADP and DR connectors.

ADP CONNECTOR: {connector_name}
ADP BASE URL: {adp_base_url}
ADP API VERSION: {adp_api_version}
ADP QUERY PARAMETERS PER ENTITY:
{adp_params}

DR CONNECTOR: source-{connector_name}
DR BASE URL: {dr_base_url}
DR API VERSION: {dr_api_version}
DR QUERY PARAMETERS PER STREAM:
{dr_params}

COMPARE:

## API Version
- Are both connectors using the same API version?
- If different, note the versions and any known breaking changes between them

## Query Parameters
For each matched stream/entity pair:
- Parameters present in DR but missing in ADP
- Parameters present in ADP but missing in DR
- Different default values for the same parameter
- Different valid ranges for the same parameter (e.g., page size limits)

Exclude pagination-specific parameters (cursor, limit, page, starting_after, etc.) from this comparison since pagination is handled differently by design.

OUTPUT FORMAT:
### API Version Comparison

| Aspect | DR | ADP | Status |
|--------|------|------|--------|
| Base URL | ... | ... | MATCH / DIFFERENT |
| API Version | ... | ... | MATCH / DIFFERENT |

### Query Parameter Comparison: {entity/stream name}

| Parameter | DR | ADP | Status |
|-----------|------|------|--------|
| ... | present (default: X) | present (default: Y) | MATCH / DIFFERENT_DEFAULT / DR_ONLY / ADP_ONLY |
")
```

### Step 4: Synthesize Results

Wait for ALL agents to complete. Compile the results into a single report.

### Step 5: Present Final Report

```markdown
# ADP / DR Connector Parity Audit: {connector_name}

**ADP Connector**: `{connector_name}` (airbytehq/sonar)
**DR Connector**: `source-{connector_name}` (airbytehq/airbyte)
**Audit Date**: {date}

## Overall Parity Score

| Dimension | Status | Gaps |
|-----------|--------|------|
| Stream/Entity Coverage | {FULL / PARTIAL / LOW} | {count} gaps |
| Authentication | {FULL / PARTIAL / LOW} | {count} gaps |
| Schema/Fields | {FULL / PARTIAL / LOW} | {count} mismatches |
| API Version | {MATCH / MISMATCH} | -- |
| Query Parameters | {FULL / PARTIAL / LOW} | {count} gaps |

---

## 1. Stream / Entity Coverage
{Agent 1 output}

## 2. Authentication Parity
{Agent 2 output}

## 3. Schema / Field Coverage
{Agent 3 output}

## 4. API Version & Query Parameters
{Agent 4 output}

---

## Priority Action Items

### P0 -- Critical Gaps
Issues that mean the ADP connector cannot serve users who depend on DR functionality.
{list}

### P1 -- Important Gaps
Missing features or fields that reduce ADP connector utility.
{list}

### P2 -- Minor Gaps
Nice-to-have improvements for full alignment.
{list}

---

## Write-Only ADP Entities (No DR Equivalent Expected)
{list from Agent 1}
```

### Step 6: Offer Next Steps

After presenting the report, ask:

```
Would you like me to:
1. Investigate a specific gap in more detail
2. Check connector-sdk or CDK support for a missing feature
3. Draft implementation tickets for the gaps found
```

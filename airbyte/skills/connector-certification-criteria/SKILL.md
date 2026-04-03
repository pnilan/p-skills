---
name: connector-certification-criteria
description: Reference criteria for certifying an Airbyte API source connector. Covers manifest-only, low-code, Python CDK, and file-based connectors. Use when evaluating whether a connector meets the certified quality bar.
devin-enabled: true
icon: shield-check
---

# Connector Certification Criteria

This document defines the quality bar for an Airbyte API source connector to be classified as **certified** (`supportLevel: certified`). Certified connectors are Airbyte-maintained, production-ready, and eligible for support SLAs.

Certification maps to `ab_internal.ql: 400` and `ab_internal.sl: 200` in metadata.

## Connector Type Reference

Criteria are universal unless marked with a connector-type tag:

| Tag | Applies to |
|-----|-----------|
| `[manifest-only]` | Pure YAML declarative connectors with no custom Python code |
| `[low-code]` | Declarative connectors with custom Python components (`components.py`) |
| `[python]` | Custom Python CDK connectors with stream classes |
| `[file-based]` | File-based CDK connectors (S3, GCS, Azure Blob, etc.) |

If no tag is present, the criterion applies to **all** connector types.

---

## 1. Metadata & Registry

### 1.1 Required metadata.yaml fields

- [ ] `supportLevel: certified`
- [ ] `releaseStage: generally_available`
- [ ] `ab_internal.ql: 400`
- [ ] `ab_internal.sl: 200`
- [ ] `definitionId` is set and matches the registry
- [ ] `dockerRepository` follows `airbyte/source-{name}` convention
- [ ] `icon` is present and references a square SVG

### 1.2 AllowedHosts

- [ ] `allowedHosts` is defined and restricts outbound traffic to only the APIs the connector communicates with
- [ ] No overly broad wildcards (e.g., `*.com` is not acceptable; `*.api.hubspot.com` is)

### 1.3 maxSecondsBetweenMessages

- [ ] If any stream's API enforces rate limits that could cause gaps longer than 90 seconds between messages, `maxSecondsBetweenMessages` is set to the longest possible gap (in seconds)
- [ ] If multiple streams have different rate limits, use the highest value

### 1.4 Suggested streams

- [ ] `suggestedStreams` is defined in metadata with a reasonable default set of streams for new connections
- [ ] If all streams are equally important or the connector has fewer than 5 streams, document why `suggestedStreams` is omitted

### 1.5 Registry presence

- [ ] `registryOverrides.cloud.enabled: true`
- [ ] `registryOverrides.oss.enabled: true`

### 1.6 Tags

- [ ] `tags` includes the correct `language:` tag (`language:manifest-only`, `language:python`, `language:java`)
- [ ] `tags` includes the correct `cdk:` tag (`cdk:low-code`, `cdk:python`, `cdk:python-file-based`)

### 1.7 Test suites declared

- [ ] `connectorTestSuitesOptions` includes `unitTests`
- [ ] `connectorTestSuitesOptions` includes `acceptanceTests` (CAT)
- [ ] `connectorTestSuitesOptions` includes `liveTests` with at least one connection ID

---

## 2. Sync Reliability

### 2.1 Sync success rate

- [ ] **>=99% sync success rate** over the trailing 7 days (config errors excluded)
- [ ] Verify via Datadog dashboards or ops tooling (`connector-health-check`)

### 2.2 State checkpointing

- [ ] The connector emits state messages with no gap longer than 15 minutes
- [ ] This is CDK-provided for manifest-only and low-code connectors; verify it is not overridden
- [ ] `[python]` Custom Python connectors must explicitly implement checkpointing if processing large volumes

### 2.3 State counts

- [ ] State messages include record counts (CDK-provided; verify not bypassed)

### 2.4 Rate limit handling

- [ ] The connector gracefully handles HTTP 429 responses with backoff (exponential, `Retry-After` header, or both)
- [ ] If the API uses non-standard rate limit signaling (e.g., HTTP 200 with rate limit in body), the connector handles it
- [ ] If a rate limit is exhausted for more than 10 minutes, the connector exits gracefully rather than hanging
- [ ] `[manifest-only]` `[low-code]` Error handlers in `manifest.yaml` include 429 in `HttpResponseFilter` with appropriate backoff
- [ ] `[python]` Rate limiting is implemented via CDK's `call_rate` module or custom backoff decorators

### 2.5 CDK version

- [ ] The connector is on a recent CDK version (within the last 3 minor releases)
- [ ] `[manifest-only]` Base image (`connectorBuildOptions.baseImage`) is current
- [ ] `[python]` `[low-code]` `airbyte-cdk` version in `pyproject.toml` is current

---

## 3. Incremental Sync

### 3.1 Incremental support

- [ ] All streams that support incremental sync implement it (cursor-based, date-range, or API-native)
- [ ] The incremental strategy matches the API's capabilities (e.g., use server-side filtering if the API supports `updated_since` parameters rather than client-side filtering)
- [ ] Streams that cannot support incremental have a documented reason (API limitation, no cursor field available)

### 3.2 Cursor correctness

- [ ] Cursor fields use the correct datetime format matching the API's response format
- [ ] If the API uses multiple datetime formats across endpoints, each stream declares the correct format
- [ ] Cursor comparisons handle timezone-aware and timezone-naive datetimes correctly
- [ ] `[manifest-only]` `[low-code]` `cursor_datetime_formats` lists all formats the stream may encounter (including legacy state formats from prior connector versions)

### 3.3 State persistence

- [ ] After a successful incremental sync, the saved state allows the next sync to resume without re-reading already-synced data
- [ ] State is not lost or reset on empty syncs (no records in the time window)
- [ ] `[python]` If using custom state management, state serialization/deserialization is tested

---

## 4. Schema & Data Integrity

### 4.1 Schema completeness

- [ ] All streams declare a JSON schema
- [ ] Schemas include all fields returned by the API (no unexpected fields in production syncs -- verify via Datadog or ops tooling)
- [ ] Schemas match the latest API documentation (fetch and compare against current API docs)
- [ ] `additionalProperties: true` is set on all object types to handle undocumented fields gracefully

### 4.2 Data types

- [ ] All date/datetime fields declare the correct `format` and `airbyte_type` (e.g., `"format": "date-time"`)
- [ ] Numeric fields use the appropriate type (`integer` vs `number`) matching the API's actual responses
- [ ] Nullable fields are declared as `["string", "null"]` (or equivalent) where the API can return null
- [ ] No fields fail schema validation in production syncs (verify via ops tooling)

### 4.3 Primary keys

- [ ] All streams declare a primary key unless no suitable PK exists in the API response
- [ ] If no PK is declared, the reason is documented (e.g., the API returns denormalized rows with no unique identifier)
- [ ] Primary keys match the API's documented unique identifiers

### 4.4 Foreign key relationships

- [ ] Where the API embeds foreign key references (e.g., `employee_id` in an employee details endpoint), these are available in the stream's records
- [ ] Sub-resource streams that depend on a parent stream's IDs (e.g., `/projects/{id}/tasks`) correctly propagate the parent ID into child records

---

## 5. Error Handling

### 5.1 Error classification

- [ ] Errors are classified using the correct `FailureType`:
  - `config_error` for user-fixable issues (bad credentials, missing permissions, invalid config)
  - `transient_error` for temporary failures (rate limits, timeouts, 5xx responses)
  - `system_error` for unexpected internal failures
- [ ] Both `message` (user-facing) and `internal_message` (developer-facing) are populated on all raised exceptions
- [ ] Reference the `writing-good-error-messages` skill for message quality standards

### 5.2 Check connection

- [ ] `check_connection` verifies credentials are valid and have sufficient permissions
- [ ] `check_connection` runs in under 15 seconds (ideally under 7 seconds)
- [ ] If credentials lack specific permissions, the error message names exactly which permissions are missing and how to grant them
- [ ] `check_connection` does not perform expensive operations (e.g., listing all records)

### 5.3 Permission-aware stream reading

- [ ] During sync, if a selected stream cannot be read due to insufficient permissions, the connector skips that stream with a clear log message indicating which permission is missing
- [ ] The sync does not fail entirely due to one stream's permission issue

### 5.4 Transient error resilience

- [ ] HTTP 5xx responses are retried with backoff
- [ ] Network timeouts are retried
- [ ] The connector distinguishes between retryable and non-retryable errors (e.g., 400 is not retried, 503 is)
- [ ] `[manifest-only]` `[low-code]` Error handlers in `manifest.yaml` cover standard HTTP error codes

### 5.5 Proactive config validation

- [ ] Invalid configuration values are caught early (during `check` or at the start of `read`) with actionable error messages
- [ ] The connector uses the API's documented error codes to map errors to specific user-facing messages

---

## 6. Authentication & Security

### 6.1 OAuth (if applicable)

- [ ] OAuth is implemented and is the default authentication method (appears first in the UI)
- [ ] OAuth on Airbyte Cloud does not ask for `client_id`, `client_secret`, or other credentials that Airbyte supplies automatically
- [ ] Token refresh is handled transparently by the CDK authenticator
- [ ] If access tokens are short-lived, the connector handles mid-sync token expiration (retry on 401 after refresh)

### 6.2 Secrets

- [ ] All credential fields in the spec are marked with `"airbyte_secret": true`
- [ ] No secrets are logged or included in error messages

### 6.3 Transport security

- [ ] The connector uses HTTPS exclusively for all API communication
- [ ] No HTTP fallback or insecure options are exposed in the configuration

---

## 7. User Experience

### 7.1 Spec / configuration fields

- [ ] Every input field is relevant to the user and does not expose implementation details
- [ ] Fields are only marked `required` if absolutely necessary
- [ ] Each field has a descriptive `title` and `description` (tooltip) that explains what it does, valid values, and its impact on connector behavior
- [ ] Fields have sensible default values where possible
- [ ] Field ordering follows a logical flow (connection/auth first, then data selection, then advanced options)

### 7.2 Stream coverage

- [ ] The connector covers all major endpoints offered by the API (compare against competitor connectors and the API's documentation)
- [ ] If endpoints are intentionally excluded, the reason is documented

### 7.3 Documentation

- [ ] Connector documentation follows the standard Airbyte docs template
- [ ] All required fields are listed under the Prerequisites section
- [ ] Setup guide links to relevant external documentation for each configuration field
- [ ] If an API key or credential is required, the exact permissions/scopes needed are documented
- [ ] Non-obvious behavior and any complex domain model are documented
- [ ] Documentation is written in user-relevant language with no implementation details
- [ ] No broken links on the page
- [ ] Links point to relevant page anchors rather than top-level URLs where applicable
- [ ] `[low-code]` `[python]` A `CONTRIBUTING.md` file documents all unique, non-obvious connector behaviors that deviate from standard CDK patterns

### 7.4 Changelog

- [ ] The connector has a changelog in its documentation page
- [ ] Each version entry includes the version number, date, and a user-facing description of changes
- [ ] Breaking changes are clearly called out in changelog entries

---

## 8. Testing

### 8.1 Connector Acceptance Tests (CAT)

- [ ] All CAT tests pass: `spec`, `connection`, `discovery`, `basic_read`, `full_refresh`, `incremental`
- [ ] `acceptance-test-config.yml` is present and correctly configured
- [ ] `expected_records` are defined for CAT tests

### 8.2 Unit tests

- [ ] `[manifest-only]` Unit tests exist under `unit_tests/` covering at minimum:
  - Schema validation for all streams
  - Any custom `components.py` logic (if present as a low-code connector)
- [ ] `[low-code]` Unit tests cover all custom Python components in `components.py`
- [ ] `[python]` Unit tests cover:
  - All stream implementations (happy path and error cases)
  - Pagination logic
  - Incremental sync / state handling
  - Authentication
  - Error handling paths
- [ ] `[file-based]` Unit tests cover file parsing, type mapping, and any custom file handling
- [ ] Invalid configuration inputs have unit tests verifying that actionable error messages are returned

### 8.3 Integration / live tests

- [ ] `connectorTestSuitesOptions` includes `liveTests` with at least one active connection ID
- [ ] Live tests pass for `spec`, `check`, `discover`, and `read` operations
- [ ] If CAT alone cannot cover certain edge cases, custom integration tests exist

### 8.4 Test recency

- [ ] Tests have been run against the current connector version (not just a historical version)
- [ ] Test sandbox accounts/credentials are active and not expired

---

## 9. Breaking Changes

### 9.1 Breaking change history

- [ ] All past breaking changes are documented in `metadata.yaml` under `releases.breakingChanges` with:
  - `message`: User-facing description of what changed and required action
  - `upgradeDeadline`: A reasonable deadline (typically 2+ weeks from release)
  - `scopedImpact`: Lists affected streams under `scopeType: stream` with `impactedScopes`
- [ ] A migration guide exists in `docs/integrations/sources/{connector-name}-migrations.md` for each breaking change
- [ ] Migration guides include step-by-step instructions (refresh schema, clear data, etc.)

### 9.2 Current state

- [ ] No pending breaking changes without a documented upgrade path
- [ ] The connector's current version has no known breaking issues in the field

---

## 10. Operational Readiness

### 10.1 Bug backlog

- [ ] All outstanding bug reports have been resolved, or unresolvable bugs have documented justification for why they cannot or should not be fixed
- [ ] No P0/P1 open issues against the connector

### 10.2 Monitoring

- [ ] The connector is included in certified connector Datadog monitors
- [ ] Alerting is configured for sync failure rate regressions

### 10.3 Competitive parity

- [ ] The connector covers all endpoints/streams offered by major competitors (Fivetran, Stitch, etc.)
- [ ] Any intentional gaps vs. competitors are documented as GitHub issues with justification
- [ ] Minimum Lovable Product (MLP) requirements from competitor research are either implemented or tracked as issues

---

## Certification Checklist Summary

When certifying a connector, update the following in a single PR:

1. **metadata.yaml**:
   - `supportLevel: certified`
   - `releaseStage: generally_available`
   - `ab_internal.ql: 400`
   - `ab_internal.sl: 200`
   - `allowedHosts` (if missing)
   - `maxSecondsBetweenMessages` (if applicable)
   - `suggestedStreams` (if applicable)
   - `connectorTestSuitesOptions` includes all required test suites

2. **Version bump**: MINOR version bump for the certification release

3. **Documentation**: Verify docs are complete per Section 7.3

4. **CONTRIBUTING.md**: `[low-code]` `[python]` Document unique behaviors

5. **Tests**: All test suites pass on the certification version

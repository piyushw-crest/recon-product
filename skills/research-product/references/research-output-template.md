# Research Output Template

Use this template as the structure for the `research-brief.md` file written to `research_results/<product_slug>/`.

Every section below should be populated. If a section does not apply, include it with a note explaining why (e.g., "N/A -- this product uses API collection, not log files."). This makes gaps explicit and prevents downstream consumers from wondering if information was simply missed.

---

## Template starts here

````markdown
# Research Brief: <Product/Vendor Name>

> **Generated:** <date>
> **Researcher:** AI-assisted research via /research-product
> **Status:** DRAFT | READY FOR REVIEW
> **Confidence:** HIGH | MEDIUM | LOW (overall confidence in completeness)

## 1. Product Overview

### 1.1 What is it?

<2-5 sentences describing the product, what it does, and what kind of organization uses it.>

### 1.2 Vendor

- **Vendor name:** <vendor>
- **Product name:** <product>
- **Product category:** <e.g., endpoint security, network firewall, identity provider, cloud CSPM>
- **Vendor documentation portal:** <URL>

### 1.3 Data generated

<What kinds of data does this product generate relevant for programmatic collection? Security events, audit logs, metrics, configuration state, alerts, etc. List the major categories.>

### 1.4 Competitive SIEM integration coverage

Which of the four named SIEM/security platforms already ship an integration for this product?
Check: **Elastic**, **Splunk**, **Panther**, **Rapid7**.

| Integration name | Vendor | Supported data types / log sources / endpoints | Collection method | Source URL | Notes / gaps |
|-----------------|--------|------------------------------------------------|-------------------|------------|--------------|
| <integration name> | <Elastic / Splunk / Panther / Rapid7> | <e.g., "Alerts API, Audit Logs API", "firewall.log, ids.log"> | <REST API / syslog / S3 / etc.> | <link to integration page> | <version, gaps, deprecated?> |

> If a platform has no integration for this product, add a row with "None found" in the Integration name column, the vendor name, and a note "No integration found as of <research date>."

See `references/competitive-siem-coverage.md` for the detailed findings written during research.

## 2. Data Collection Method

### 2.1 Recommended method

- **Collection mechanism:** <e.g., `REST API polling`, `syslog TCP/UDP`, `S3 object store`, `local log file`, `cloud event stream`>
- **Rationale:** <why this method is recommended>

### 2.2 Alternative methods

<List other viable methods with brief pros/cons. If only one method exists, state that.>

| Method | Mechanism type | Pros | Cons |
|--------|---------------|------|------|
| <method 1> | <REST API / syslog / S3 / etc.> | <pros> | <cons> |
| <method 2> | <type> | <pros> | <cons> |

### 2.3 Vendor-side setup required

<What does the user need to configure on the vendor side to enable data export? API key creation, syslog forwarding rules, S3 bucket policies, etc.>

## 3. Data Source Details

### 3.1 Connection and authentication

<Detailed connection information for the recommended collection method.>

**For API-based:**
- Base URL: `<url>`
- API version: <version>
- Authentication: <see below>
- Rate limits: <requests per minute/hour, burst limits>
- Documentation: <link to auth docs>

**Authentication detail (choose the applicable block):**

*If API key or static Bearer token:*
- Method: <API key header / API key query param / Bearer token>
- Header format: `<e.g., Authorization: Bearer <token>, X-API-Key: <key>>`
- Credential creation steps: <how to obtain the key>
- Required scopes/permissions: <list>
- Token lifetime: <expiry, or "does not expire">

*If OAuth2 (authorization_code or client_credentials):*
- Method: OAuth2
- Grant type: <`authorization_code` / `client_credentials` / both>
- Authorization URL: `<url>` (authorization_code only)
- Token URL: `<url>`
- Refresh URL: `<url>` (often same as token URL)
- Scopes required: <list of scopes needed for data collection>
- All available scopes: <full list if documented>
- Client registration: <how to create a client_id / client_secret — app registration steps, admin console, etc.>
- Token lifetime: <access token expiry, e.g., "30 days", "1 hour">
- Refresh token lifetime: <if documented>
- Token response format: `access_token`, `refresh_token`, `expires_in`, `token_type`
- Additional notes: <PKCE required? Specific redirect URI requirements? VPC/tenant-specific URLs?>

*If interactive-only token generation (manual PAT, browser token page):*
- Method: Manual token generation (interactive only)
- Note: <1-2 sentences describing the manual process. State this is not a standard OAuth2 flow and is not suitable as the primary auth method for an automated connector. If a proper OAuth2 flow also exists, reference it above as the primary method.>

*If vendor-specific token exchange (non-standard):*
- Method: Custom token exchange
- Steps: <describe the exact steps — e.g., POST to /api/auth with API key → receive access token + refresh token → use Bearer on subsequent calls>
- Refresh mechanism: <how and when to refresh>
- Implementation note: This is not a standard OAuth2 flow. Any connector must manage the token exchange manually using the steps above.

**For syslog-based:**
- Protocol: <TCP/UDP/TLS>
- Default port: <port>
- Syslog format: <RFC 3164/5424>
- Message format inside envelope: <CEF/LEEF/KV/JSON/free text>

**For cloud ingest:**
- Service: <S3/SQS, Event Hub, Pub/Sub, etc.>
- Authentication: <IAM role, connection string, service account, etc.>
- Required permissions: <list>

### 3.2 Endpoints / data paths

<For APIs: list all relevant endpoints with path, HTTP method, and purpose.>
<For logs: list file paths per OS.>
<For cloud: list bucket/topic/hub names and path patterns.>

| Endpoint / Path | Method | Purpose | Event types |
|----------------|--------|---------|-------------|
| <path> | <GET/POST> | <description> | <event types returned> |

### 3.3 Pagination (API only)

- **Mechanism:** <offset, cursor, link-header, keyset, page-number, none>
- **Page size parameter:** `<param_name>` (default: <n>, max: <n>)
- **Next page indicator:** `<field_name>` in response body / `Link` header / etc.
- **Termination condition:** <empty results array, null cursor, count < page_size, etc.>
- **Documentation:** <link>

### 3.4 Time-based filtering (API only)

- **Start time parameter:** `<param_name>`
- **End time parameter:** `<param_name>` (if applicable)
- **Time format:** <ISO 8601, Unix epoch seconds/milliseconds, custom>
- **Timezone:** <UTC, configurable, local>
- **Sort order:** <ascending/descending by default, configurable?>
- **Incremental collection strategy:** <use last event timestamp as next start, cursor includes time state, window-based snapshot only, etc.>

### 3.5 Reference documentation

<Comprehensive list of documentation links used in this research.>

| Title | URL | Relevance |
|-------|-----|-----------|
| <doc title> | <url> | <what it covers> |

## 4. Data Format and Structure

### 4.1 Format overview

- **Wire format:** <JSON, NDJSON, syslog+CEF, syslog+KV, CSV, XML, multiline text>
- **Encoding:** <UTF-8, ASCII, etc.>
- **Compression:** <gzip, none, etc.>
- **Envelope structure:** <e.g., `{"data": [...], "meta": {...}}` or flat array>

### 4.2 Event types

<List all distinct event types/categories the product generates.>

| Event type | Description | Relative volume | Priority |
|-----------|-------------|-----------------|----------|
| <type> | <description> | <high/medium/low> | <high/medium/low for integration> |

### 4.3 Field inventory

<For each event type (or shared across types), list the fields.>

#### Common fields (present in all/most event types)

| Field path | Type | Description | Example value | Always present? |
|-----------|------|-------------|---------------|-----------------|
| <field> | <string/int/float/bool/object/array> | <description> | <example> | <yes/no> |

#### <Event Type 1> specific fields

| Field path | Type | Description | Example value |
|-----------|------|-------------|---------------|
| <field> | <type> | <description> | <example> |

<Repeat for each event type.>

### 4.4 Sample data

<Include or reference representative sample events. If inline, use fenced code blocks. If separate files, reference them.>

See `references/sample-events/` for complete sample data files:
- `<event_type_1>.json` -- <description>
- `<event_type_2>.json` -- <description>
- `<event_type>.log` -- <description>

<If field inventories or schema analyses were too large to include inline, reference the files:>
See `references/field-schema-analysis.md` for the complete field inventory extracted from <source>.
See `temp/<subfolder>/` for the raw source artifacts (cloned repos, downloaded schemas, etc.).

#### Inline sample (most common event type)
```json
<paste one representative event here>
```

### 4.5 Timestamp handling

- **Primary timestamp field:** `<field_name>`
- **Format:** <ISO 8601, Unix epoch seconds, Unix epoch milliseconds, custom format string>
- **Timezone:** <always UTC, local timezone, timezone offset included, configurable>
- **Suggested role mapping:**
  - `<field_name>` → primary event timestamp
  - `<field_name>` → window/period start
  - `<field_name>` → window/period end
  - `<field_name>` → entity first seen / last seen
- **Additional timestamp fields:** <list any secondary timestamps and their meaning>

> Platform-specific field mapping (e.g. `@timestamp` in ECS, `_time` in CIM, `time` in OCSF) is out of scope for this research brief and belongs in the implementation phase.

### 4.6 Special parsing considerations

<Note anything that makes parsing non-trivial: multiline patterns, embedded JSON in string fields, variable schemas, field name inconsistencies across API versions, binary-encoded fields, etc. Note which fields require flattening decisions for nested/array structures.>

## 5. Data Model Analysis

### 5.1 Field categorization

<For each event type, categorize fields by functional role.>

| Event type | Identity fields | Timestamp fields | IP/Network fields | User/Account fields | Severity/Status fields |
|-----------|----------------|-----------------|------------------|--------------------|-----------------------|
| <type> | <list> | <list> | <list> | <list> | <list> |

### 5.2 Normalization candidates

<Which fields are strong candidates for mapping to common security data models? List candidates without committing to a specific target schema.>

| Source field | Functional meaning | Candidate mapping (OCSF / CIM / ECS / Chronicle UDM) |
|-------------|-------------------|------------------------------------------------------|
| <field> | <what it represents> | <e.g., OCSF: device.uid, CIM: src, ECS: host.id> |

### 5.3 High-value fields for filtering and correlation

<Which fields are most useful for search, alerting, and correlation in any SIEM or observability platform?>

- **Primary correlation key:** <field that uniquely identifies an entity or event>
- **Severity/risk indicator:** <field and its possible values>
- **Actor/source fields:** <fields representing the initiating entity>
- **Target/destination fields:** <fields representing the affected entity>
- **MITRE ATT&CK references:** <fields containing technique IDs, tactic names, or CVE IDs if present>

### 5.4 Geo/network enrichment candidates

<IP address fields that are candidates for geo or ASN enrichment.>

| Field | Content | Public/Private | Enrichment candidate |
|-------|---------|----------------|---------------------|
| <field> | <description> | <public/private/mixed> | <yes/no and why> |

## 6. Configuration Plan

### 6.1 Required configuration variables

| Variable | Type | Title | Description | Default | Show user |
|----------|------|-------|-------------|---------|-----------|
| <var_name> | <string/secret/integer/boolean/url> | <display title> | <help text> | <default or none> | <yes/no> |

### 6.2 Optional configuration variables

| Variable | Type | Title | Description | Default | Show user |
|----------|------|-------|-------------|---------|-----------|
| `interval` | string | Polling Interval | How often to poll the API | `5m` | yes |
| `initial_interval` | string | Initial Lookback Window | How far back to fetch on first run | `24h` | yes |
| `page_size` | integer | Page Size | Results per page | <vendor default> | no |
| `http_client_timeout` | string | HTTP Client Timeout | Request timeout | `60s` | no |
| `proxy_url` | url | Proxy URL | Optional HTTP/HTTPS proxy | — | no |
| `tls_config` | object | TLS Configuration | Custom TLS settings (CA cert, client cert, verify peer) | — | no |
| `store_raw_payload` | boolean | Store Raw Payload | Retain the unprocessed API response alongside normalized fields | `false` | yes |
| `tags` | string | Tags | User-defined tags to attach to collected events | — | yes |

### 6.3 Deployment notes

<Any notes about deployment architecture, network requirements, firewall rules, proxy considerations, etc. Use "collector host" to refer to the machine running the connector.>

## 7. Recommended Connector Architecture

### 7.1 Connector identifier

`<connector_slug>` (lowercase, underscores)

### 7.2 Recommended data collectors

<What logical groupings of data should be separate collectors/streams? One row per recommended collector.>

| Collector name | Collection method | Source endpoint/path | Description |
|----------------|------------------|---------------------|-------------|
| <name> | <REST API / syslog / S3 / etc.> | <endpoint or path> | <what it collects> |

### 7.3 Architecture rationale

<Why this breakdown? Different update frequencies, fundamentally different data shapes, different volume profiles, different authentication surfaces, or other reasons?>

### 7.4 Estimated implementation complexity

- **Authentication complexity:** <simple (static API key) / moderate (token exchange, custom flow) / complex (OAuth2 with refresh, multi-step)>
- **Pagination complexity:** <none / simple (page number + total pages) / moderate (cursor-based) / complex (multi-phase, nested pagination)>
- **Data parsing complexity:** <simple (flat JSON) / moderate (nested objects, multiple event types) / complex (binary fields, variable schema, multiline, routing logic)>
- **Field count estimate:** <approximate number of distinct fields per collector>

## 8. Open Questions and Gaps

<List anything that could not be determined from research and requires user input, vendor clarification, or hands-on testing.>

| # | Question | Impact | Suggested resolution |
|---|----------|--------|---------------------|
| 1 | <question> | <high/medium/low> | <how to resolve> |

## 9. Source Attribution

<List all sources used in this research with how they were accessed.>

| Source | URL | Access method | Date |
|--------|-----|---------------|------|
| <title> | <url> | <web search / user provided / local file / own knowledge> | <date> |
````

---

## Usage notes

- Sections 1-4 form the factual research foundation.
- Section 5 (Data model analysis) is a platform-neutral categorization layer — it identifies field roles and normalization candidates without committing to any target schema.
- Section 6 (Configuration) bridges research to implementation planning.
- Section 7 (Architecture) is the connector design recommendation.
- Section 8 (Open questions) captures what still needs human judgment.
- Section 9 (Attribution) provides traceability for all claims.

The brief should be thorough enough that someone can pass it directly to any integration build workflow as the primary input.

## Companion artifacts

The research brief is the primary output, but it is supported by additional files in the same directory:

- **`test-api.py`** -- *(API collection only)* Standalone Python script that exercises the exact API flow proposed for the connector. Tests connectivity, authentication, pagination, and response structure. Run it against the real vendor API and share the resulting archive for development. See `references/test-api-script-spec.md` in the skill directory for the full specification.
- **`data-model-analysis.md`** -- Field categorization and normalization candidates, written during Phase 4.
- **`configuration-plan.md`** -- Full connector configuration variable plan, written during Phase 5.
- **`references/`** -- Curated research artifacts: detailed field analyses, API spec notes, sample events. These are polished enough for downstream consumers.
- **`temp/`** -- Raw downloaded artifacts: cloned repos, SDK sources, large schema files, analysis scripts. Retained as reference for the human and for reproducibility.

When the brief references detailed findings that are too large to include inline, it should point to the appropriate file in `references/` or `temp/` with a path and one-line description.

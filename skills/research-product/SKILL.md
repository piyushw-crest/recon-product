---
name: research-product
description: "Research a vendor, product, or feature to collect all information needed before building a connector or integration for any target platform. Investigates data collection methods, API or log documentation, sample data formats, field schemas, normalization candidates, and configuration requirements. Outputs a structured research brief to research_results/<product>/. Invoke manually with /research-product."
license: Apache-2.0
metadata:
  version: "1.0"
disable-model-invocation: true
---

# Research Product

You are the **research orchestrator**. Your job is to thoroughly investigate a vendor, product, or feature and produce a structured research brief that a downstream integration builder can use as the primary input for building a connector or integration on any target platform.

You delegate parallel research and analysis to `deep-research` subagents, synthesize their findings with any locally provided reference material and your own grounded knowledge, and write the final brief to disk.

Subagents are **write-capable** -- they can download repositories, install packages, run Python analysis scripts, and write findings to files on disk. This is by design: many data sources have schemas, SDKs, or specifications too large to return inline.

## What you provide

Include any combination of the following when you invoke this command.
Use `@`-mentions for files/folders and paste links inline.

| Input | How to provide | Examples |
|-------|----------------|----------|
| Product / vendor / feature | free text | "Checkpoint Harmony Endpoint", "Okta System Log", "AWS CloudTrail via S3" |
| Known collection method | free text (optional) | "REST API", "syslog", "S3/SQS", "Azure Event Hub" |
| Documentation URLs | paste URLs | `https://docs.vendor.com/api/v2`, `https://docs.vendor.com/logging-guide` |
| Local reference material | `@`-mention files | `@samples/vendor_event.json`, `@notes/vendor-api-notes.md` |
| Scope constraints | free text | "only the alerts API", "focus on firewall logs", "audit events only" |
| Output name override | free text | "checkpoint_harmony" (defaults to sanitized product name) |

Anything typed after `/research-product` is your research goal.

### Invocation examples

```
/research-product Checkpoint Harmony Endpoint security events
  API docs: https://developer.checkpoint.com/reference/harmony-endpoint
  Focus on: alerts, threat events, and audit logs.
  Known method: REST API with pagination.
```

```
/research-product Palo Alto Cortex XDR
  @notes/cortex-xdr-api-rough-notes.md
  Need to investigate both the Incidents API and Alerts API.
```

```
/research-product Cisco Meraki syslog events
  https://documentation.meraki.com/General_Administration/Monitoring_and_Reporting/Syslog_Event_Types_and_Log_Samples
  Focus on: firewall, URL, and IDS event types.
  Known method: syslog over UDP/TCP.
```

```
/research-product AWS Security Hub findings via S3/SQS
  Need full schema of ASFF finding format and S3 delivery configuration.
```

## Before you start -- load references

Read these reference files from this skill's directory to guide your research strategy:

1. `references/data-collection-methods.md` -- understand collection method types and what to investigate for each
2. `references/research-output-template.md` -- the structure your final brief must follow
3. `references/competitive-siem-checklist.md` -- always load this; defines where to search each of the four platforms (Elastic, Google SecOps, Splunk, Sumo Logic) and what to capture
4. Based on the identified collection method, read the applicable checklist:
   - `references/api-research-checklist.md` -- for REST API-based collection
   - `references/log-file-research-checklist.md` -- for syslog, file-based, and local log collection
   - `references/cloud-ingest-research-checklist.md` -- for S3/SQS, Event Hub, Pub/Sub, and similar cloud delivery
5. If the collection method is (or turns out to be) API-based, also read:
   - `references/test-api-script-spec.md` -- specification for the API test script generated in Phase 7

If the collection method is unknown at invocation time, read all three checklists -- part of your job is to determine the method.

Do **not** load implementation-phase skills (pipeline builders, schema mappers, connector frameworks, etc.). Those are for implementation, not research.

## Output location

Write all research output to:

```
research_results/<product_slug>/
```

Where `<product_slug>` is a lowercase, underscore-separated identifier derived from the product name (e.g., `checkpoint_harmony_endpoint`, `palo_alto_cortex_xdr`, `cisco_meraki`). The user may override this with the "Output name override" input.

Create this directory structure:

```
research_results/<product_slug>/
  research-brief.md             # the main structured research brief
  test-api.py                   # API connectivity & flow test script (API collection only)
  configuration-plan.md         # planned connector configuration variables
  references/                   # curated research artifacts for downstream consumers
    unified-field-mapping.md    # single authoritative file: per-schema source metadata +
                                #   19-column master table (raw field × ECS/UDM/Splunk/Sumo)
    competitive-siem-coverage.md  # narrative SIEM integration findings per platform
    api-spec-notes.md           # API endpoint details, request/response examples (if API)
    log-format-notes.md         # log format details, sample lines (if log-based)
    sample-events/              # representative sample data files
      <event_type>.json         # one file per event type or data format variant
      <event_type>.log
  temp/                         # intermediate working artifacts and raw downloads
    field-catalog.md            # raw field inventory (Track C subagent output)
    data-model-intermediate.md  # field categorization (Phase 4 output)
    ecs-field-mapping.md        # ECS mappings (Track E subagent output, merged into unified)
    udm-field-mapping.md        # UDM mappings (Track E subagent output, merged into unified)
    splunk-field-mapping.md     # Splunk OCSF mappings (Track E subagent output, merged into unified)
    sumo-field-mapping.md       # Sumo CSE mappings (Track E subagent output, merged into unified)
    <descriptive-subfolder>/    # cloned repos, SDK sources, large schema files, scripts
```

Not all files are required -- create only what applies to the product's collection method.

**Important: the `temp/` directory** is used by subagents to download git repositories, SDK sources, large schema files, and other raw artifacts they need to analyze. Do not delete `temp/` after research completes -- it serves as a reference for the human and may be useful for follow-up work.

## Workflow

### Phase 1: Parse and plan

1. Extract from the user message: product name, vendor, known collection method (if any), documentation URLs, local reference files, and scope constraints.
2. Read any `@`-mentioned local files.
3. Fetch any documentation URLs provided inline to get initial context.
4. Determine the output slug and create the output directory.
5. Identify which research tracks to pursue based on what is known and unknown.

### Phase 2: Parallel research

Launch multiple `deep-research` subagents in parallel using the Task tool. Each subagent focuses on a specific research track. **You should launch as many parallel agents as makes sense for the product -- typically 2-4 agents.**

**IMPORTANT -- subagent context and capabilities:**

- Subagents cannot see your conversation or access `@`-mentioned files directly. Include any relevant content from local reference files and fetched URLs in the task prompt.
- Subagents are **write-capable**. Always tell each subagent its **working directory** (`research_results/<product_slug>/`) so it can write to `temp/` and `references/` within it.
- Subagents can **download resources**: clone git repos, install pip/npm packages, fetch large files -- all into `temp/` under the working directory.
- Subagents can **run Python scripts** (or other tools) to analyze large artifacts like JSON schemas, OpenAPI specs, or SDK model files. Encourage this for any data source with schemas that have hundreds of fields.
- Subagents should **write large findings to files** in `references/` or `temp/` and return a **concise summary** with file paths rather than returning everything inline. This keeps context manageable.

**Template for subagent instructions** -- include a block like this in every subagent task prompt:

```
Working directory: research_results/<product_slug>/
- Download raw artifacts to: research_results/<product_slug>/temp/
- Write curated findings to: research_results/<product_slug>/references/
- If your findings are extensive (large field inventories, full schema analyses, many sample events),
  write them to a .md file in references/ or temp/ and return a summary with file paths.
- Use Python to analyze large files (JSON schemas, OpenAPI specs, SDK models) rather than
  trying to read them manually.
- Do not remove anything from temp/ -- it is kept as reference.
```

#### Research Track A: Product overview and data collection methods

Instruct the subagent to investigate:
- What the product/feature is and what kind of data it generates
- All available methods for collecting/exporting data (API, syslog, file export, cloud streaming, SIEM forwarding, etc.)
- Which method is best suited for a programmatic integration and why
- Official vendor documentation links for each collection method
- Any known limitations, rate limits, or licensing requirements for data access
- Competitive SIEM coverage is handled in Research Track E (see below) — do not duplicate that work here

Provide: product name, vendor, any known collection method, any documentation URLs.

#### Research Track B: Data source deep dive

Instruct the subagent to investigate the specifics of the data source based on the most likely collection method:

**For APIs:**
- Base URL and endpoint paths
- Authentication method (API key, OAuth2, Bearer token, Basic auth, custom headers)
- **OAuth2 deep dive (critical):** If the API uses OAuth2, identify ALL supported grant types (client_credentials, authorization_code, etc.) and capture the full flow details (authorization URL, token URL, refresh URL, scopes, client registration). Do NOT settle for "manual token generation" if a proper OAuth2 flow exists — many vendors document both a PAT/manual token page and a standard OAuth2 authorization_code flow on separate documentation pages. See `api-research-checklist.md` for the detailed OAuth2 investigation checklist.
- Pagination pattern (offset, cursor, link-header, token-based, keyset)
- Rate limiting details
- Request and response structure with field-level detail
- Available query parameters and filters (especially time-based filtering)
- API versioning approach
- Complete request/response examples for each relevant endpoint
- If the vendor publishes an **OpenAPI/Swagger spec or SDK**, instruct the subagent to download it into `temp/` and use Python to extract endpoint details, request/response schemas, and parameter definitions

**For logs/syslog:**
- Log format (syslog RFC 3164/5424, CEF, LEEF, key-value, JSON, CSV, multiline)
- Default log file paths per OS
- Syslog facility and severity usage
- Message structure and delimiters
- Sample log lines for each event type

**For cloud ingest (S3/SQS, Event Hub, Pub/Sub, etc.):**
- Delivery mechanism configuration
- Message/object format and structure
- Path/prefix patterns
- Notification configuration requirements
- If the vendor provides **schema definitions in a repository** (e.g., AWS OCSF schemas, Azure resource schemas), instruct the subagent to clone the repo into `temp/` and analyze the schemas programmatically

Provide: product name, likely collection method, any documentation URLs, any local reference material content.

#### Research Track C: Event types and field schema

Instruct the subagent to investigate:
- All distinct event types, categories, or log sources the product generates
- Field names, types, and descriptions for each event type
- Common fields across event types vs. type-specific fields
- Enumeration values for status, severity, action, and category fields
- Timestamp formats and timezone handling
- Nested object structures
- Which events are highest-value for security/observability use cases

**For data sources with large schemas:** Instruct the subagent to download the schema source (git repo, SDK package, JSON schema file) into `temp/` and use Python to programmatically extract field inventories, type information, and enum values. The subagent should write the complete field analysis to `temp/field-catalog.md` (or multiple files if per-event-type breakdowns are needed) and return a summary. Include at the top of `temp/field-catalog.md`: product name, research date, and the source URL(s) / file path(s) from which the field list was extracted.

Provide: product name, any documentation URLs, any sample data content from local files.

#### Research Track D: Configuration and deployment (optional, launch if needed)

Instruct the subagent to investigate:
- What configuration the end user needs to provide (credentials, URLs, paths, filters)
- How to enable/configure data export on the vendor side
- Network requirements (ports, protocols, firewall rules)
- Common deployment architectures
- Prerequisites and permissions needed

Provide: product name, collection method, any documentation URLs.

#### Research Track E: Competitive SIEM integration coverage

Instruct the subagent to check the following four platforms for existing integrations with this product: **Elastic**, **Google SecOps**, **Splunk**, **Sumo Logic**.

**Full research guidance is in `references/competitive-siem-checklist.md`** (loaded at startup). Pass the entire contents of that file to the subagent in the task prompt so it has the per-platform search strategies, capture requirements, and output format without needing to load files itself.

Key instructions to include in the subagent task prompt:
- Follow the per-platform search strategy in the checklist exactly (catalogue search, GitHub repo check, documentation index)
- For each platform: capture integration name, supported data types / log sources / endpoints, target schema (ECS / UDM / OCSF/CIM / OCSF), collection method, version, and any gaps
- If a platform has no integration: document the absence explicitly (what was searched, date) — do not leave it blank
- **Elastic — ECS field mapping (critical):** use the ingest pipeline YAML files as the primary source. For each data stream, fetch `packages/<slug>/data_stream/<stream>/elasticsearch/ingest-pipeline/default.yml` from the `elastic/integrations` GitHub repo. Run a Python script to extract all `rename` processor entries (field → target_field). Cross-check `set` processors against the docs "Exported fields" table. The resulting table has columns: `data_stream | raw source field | ECS field (target) | ECS type | ECS description`. Write to **`temp/ecs-field-mapping.md`** using the format in the checklist. Include metadata at the top: integration slug, version, pipeline YAML source URL, docs URL, extraction method, any gaps/caveats. Also include a summary in the Elastic section of `references/competitive-siem-coverage.md`.
- **Google SecOps — UDM field mapping (critical):** fetch the parser documentation page and capture the complete UDM mapping table (columns: Log field | UDM field | UDM type | UDM description | Logic). Write to **`temp/udm-field-mapping.md`** using the format in the checklist. Include metadata at the top: log type ID, display name, supported formats, parser docs URL, change log URL, extraction method, any gaps/caveats (e.g. "parser docs returned 404; reconstructed from change log"). Also include a summary in the Google SecOps section of `references/competitive-siem-coverage.md`.
- **Splunk — OCSF field mapping (capture only if vendor-published):** check the TA Splunkbase Documentation tab, the TA GitHub README/docs, and `splunk/splunk-ocsf-framework` for an explicit OCSF field mapping table. Do NOT infer mappings — only capture what is published by the vendor. Write to **`temp/splunk-field-mapping.md`** (mapping table with columns `raw field | OCSF field | OCSF class | OCSF type | OCSF description` if found, or a structured absence note if not). Include metadata at the top: TA name, Splunkbase URL, whether OCSF mapping was vendor-published, any gaps/caveats. Also include a summary in the Splunk section of `references/competitive-siem-coverage.md`.
- **Sumo Logic — Cloud SIEM mapper rules (critical):** clone `https://github.com/SumoLogic/cloud-siem-content-catalog` into `temp/cloud-siem-content-catalog/`. Search the `mappings/` directory for the product. Run a Python script to parse the mapper JSON files and extract `rawField → fields` pairs (with `recordType`). Enrich with CSE schema attribute type and description from `https://help.sumologic.com/docs/cse/schema/schema-attributes/`. Write to **`temp/sumo-field-mapping.md`** with columns `raw field | CSE attribute | record type | CSE type | CSE description`. Include metadata at the top: mapper file path, mapper GitHub URL, record types covered, extraction method, any gaps/caveats. Also include a summary in the Sumo Logic section of `references/competitive-siem-coverage.md`.
- Write full findings to `references/competitive-siem-coverage.md` using the output format defined in the checklist
- Return a concise summary with the file path and a one-line status per platform (found / not found), plus confirmation that all four intermediate mapping files were written to `temp/`: `ecs-field-mapping.md`, `udm-field-mapping.md`, `splunk-field-mapping.md`, `sumo-field-mapping.md`. These are intermediate artifacts — the orchestrator will merge them into `references/unified-field-mapping.md` in Phase 6.

Provide: product name, vendor name, full contents of `references/competitive-siem-checklist.md`.

### Phase 3: Synthesize and supplement

After all subagents return:

1. **Read subagent-written files.** Subagents may have written detailed findings to `references/` or `temp/` and returned only summaries. Read the files they reference to get the full picture. The subagent summaries will tell you which files to read and when.
2. **Merge findings** from all research tracks into a unified understanding.
3. **Cross-reference** subagent findings with any local reference material the user provided.
4. **Fill gaps** using your own grounded knowledge of the vendor/product. Only include information you are confident is accurate and can be attributed to known documentation, specifications, or widely established facts. Flag any details that could not be verified with a `[UNVERIFIED]` marker.
5. **Resolve conflicts** between subagent findings. When sources disagree, prefer official vendor documentation over third-party sources.
6. **Collect sample data** -- extract or compile representative sample events from documentation, API response examples, or log format guides. Save each as a separate file in the `sample-events/` subdirectory.
7. **Review temp/ artifacts** if needed. Subagents may have downloaded repos, SDKs, or schemas into `temp/`. You can inspect these directly if you need more detail than what the subagent summaries and reference files provide.

### Phase 4: Data model analysis

Perform a field categorization and normalization analysis:

1. For each identified field from the product's data, categorize it by functional role:
   - **Identity fields** — entity IDs, names, unique keys
   - **Timestamp fields** — event time, window start/end, creation time
   - **IP/network fields** — source IPs, destination IPs, public vs. private
   - **User/account fields** — usernames, account IDs, email addresses
   - **Severity/status fields** — risk levels, criticality, action outcomes
   - **Nested/array structures** — complex objects that require flattening decisions
2. Identify which fields are highest-value for filtering, correlation, and alerting in any SIEM or observability platform.
3. Note normalization candidates — fields that would map well to common security data models (OCSF, CIM, ECS, Chronicle UDM, etc.) — but do not commit to any specific schema. The choice of target schema is a downstream implementation decision.
4. Note IP address fields that are strong candidates for geo or ASN enrichment.
5. Write the analysis to `temp/data-model-intermediate.md`. Format: one row per field with columns `field | category | high-value (yes/no) | normalization candidates | enrichment candidate (yes/no)`. This file is an intermediate artifact used in Phase 6 to populate the `Category` column of `references/unified-field-mapping.md`.

### Phase 5: Configuration planning

Based on the identified collection method, plan the connector configuration:

1. Determine required vs. optional configuration variables.
2. For each variable, specify: name, description, type, whether it's required, and a sensible default value.
3. Identify the configuration surface any connector would need: credentials, base URL, polling interval, filtering options, page size, proxy, TLS settings.
4. Write the plan to `configuration-plan.md`.

### Phase 6: Write research brief

Compile the full research brief following the template in `references/research-output-template.md`. Write it to `research_results/<product_slug>/research-brief.md`.

The brief must be self-contained -- a reader should be able to use it as the primary input to any integration build workflow and have everything they need.

**Step 6a — Build `references/unified-field-mapping.md` (do this before writing the brief):**

Read the six intermediate files written by Track C, Phase 4, and Track E, then merge them into the single authoritative unified mapping file:

1. Read `temp/field-catalog.md` → canonical raw field list (each row becomes a row in the unified table; provides `Raw field`, `Description`, `Data type` columns)
2. Read `temp/data-model-intermediate.md` → field category per raw field (provides `Category` column)
3. Read `temp/ecs-field-mapping.md` → ECS mapping per raw field (provides `ECS field`, `ECS type`, `ECS description` columns; also extract ECS metadata block)
4. Read `temp/udm-field-mapping.md` → UDM mapping per raw field (provides `UDM field`, `UDM type`, `UDM description`, `UDM logic` columns; also extract UDM metadata block)
5. Read `temp/splunk-field-mapping.md` → Splunk OCSF mapping per raw field (provides `Splunk OCSF field`, `Splunk OCSF class`, `Splunk OCSF type`, `Splunk OCSF description` columns; also extract Splunk metadata block)
6. Read `temp/sumo-field-mapping.md` → Sumo CSE mapping per raw field (provides `Sumo CSE attribute`, `Sumo record type`, `Sumo CSE type`, `Sumo CSE description` columns; also extract Sumo metadata block)

Join rule: match rows by raw field name (exact match first; suffix/partial match as fallback for cases like `jsonPayload.requesterIP` vs `requesterIP`). Schema mapping entries that cannot be joined to any raw field go into a `### Unmatched schema mappings` section at the bottom with a note. Use `—` for any cell where a schema has no mapping for that field.

Write `references/unified-field-mapping.md` with this structure:
- **Product context** table (product, vendor, research date, raw field source)
- **ECS source metadata** table (integration found, slug, version, pipeline YAML URL, docs URL, extraction method, gaps/caveats)
- **UDM source metadata** table (parser found, log type ID, display name, supported formats, parser docs URL, change log URL, extraction method, gaps/caveats)
- **Splunk OCSF source metadata** table (TA found, TA name, Splunkbase URL, OCSF mapping published yes/no, source URL, extraction method, gaps/caveats)
- **Sumo Logic Cloud SIEM source metadata** table (mapper rule found, mapper file path, mapper GitHub URL, record types covered, extraction method, gaps/caveats)
- **19-column field mapping table** with header: `Raw field | Description | Data type | Category | ECS field | ECS type | ECS description | UDM field | UDM type | UDM description | UDM logic | Splunk OCSF field | Splunk OCSF class | Splunk OCSF type | Splunk OCSF description | Sumo CSE attribute | Sumo record type | Sumo CSE type | Sumo CSE description`
- **Unmatched schema mappings** section for any schema fields that had no raw field match

If any intermediate file is missing, write the appropriate "not found" / "not published" values in that schema's metadata table and use `—` for all cells in that schema's columns.

**Step 6b — Write section 4.3 of the research brief:**

Section 4.3 in the brief shows a condensed cross-reference table (field paths only, no descriptions) followed by a pointer to the full unified file:

```
#### Schema field mapping (all platforms)

> Full mapping with raw types, per-schema descriptions, field categories, and source metadata is in
> `references/unified-field-mapping.md`.

| Raw field | ECS field | UDM field | Splunk OCSF field | Sumo CSE attribute |
|-----------|-----------|-----------|-------------------|--------------------|
| <field>   | <ecs.f>   | <udm.f>   | <ocsf.f or —>     | <cse.attr or —>    |
```

Do not reproduce the full 19-column table inline in the brief — reference the file instead.

### Phase 7: API test script (API collection only)

**Skip this phase entirely if the recommended collection method is not REST API-based.** This phase only applies when the research has identified a REST API as the collection method.

After the research brief and all companion artifacts are written, generate a standalone Python test script that exercises the exact API flow proposed for the connector. This lets a human validate connectivity, authentication, pagination, and response structure against a real (or mock) API before any connector build work begins.

1. **Read the specification:** Load `references/test-api-script-spec.md` from this skill's directory. It defines every requirement for the script in detail — file structure, CLI arguments, output files, error handling, and the relationship to the proposed collection flow.

2. **Gather inputs from earlier phases.** The script is synthesized from research already completed:
   - **Authentication method and credential creation steps** → from section 3.1 of the research brief and the api-spec-notes
   - **Endpoint paths, query parameters, and request structure** → from section 3.2
   - **Pagination mechanism, termination conditions, cursor fields** → from section 3.3
   - **Time-based filtering parameters and formats** → from section 3.4
   - **Configuration variables** → from `configuration-plan.md`

3. **Write the script** to `research_results/<product_slug>/test-api.py`. Key requirements (see spec for full detail):
   - **Standard library only** — `urllib.request`, `json`, `logging`, `argparse`, `ssl`, etc. No third-party dependencies.
   - **Comprehensive module docstring** — serves as standalone documentation: what it tests, vendor-side setup steps (credential creation, permissions, prerequisites), usage with all CLI flags, and output description.
   - **Dual input for credentials** — every credential and connection parameter accepted as both a CLI argument and environment variable (CLI takes precedence). Use `argparse` with `default=os.environ.get(...)`.
   - **Base URL always configurable** — full URL including scheme (`https://...`), even if the vendor has a single static URL. This enables pointing at mock servers.
   - **`--max-pages` always present** — safety limit to prevent infinite pagination during testing.
   - **TLS verification disabled** — this tests API flow, not certificate health.
   - **Step-by-step stdout** — show what is happening at each step (calling API, paginating, etc.) without printing raw request/response bodies or any sensitive data.
   - **Output directory** with two files:
     - `test-api.log` — verbose log (superset of stdout, written via Python `logging`)
     - `trace.json` — detailed request/response trace: full URLs, headers, response bodies, pagination state transitions. Auth values redacted.
   - **Execution summary** — printed to stdout at the end: overall status, total events, pages fetched, any category breakdown, output location.
   - **Archive** — compress the output directory as `.tar.gz` and print the path with instructions to share it with integration maintainers.
   - **Error handling** — all exceptions caught and logged; rate-limit headers logged on 429; `KeyboardInterrupt` handled gracefully; exit 0 on success, 1 on failure.

4. **Mirror the proposed collection flow.** The script's request sequence, pagination logic, and termination conditions must match what was described in the research brief for the connector. This is the core value of the script — if it works, the connector implementation should work too.

### Phase 8: Verify and report

1. Verify all output files are written and well-formed.
2. If `test-api.py` was generated (API collection method), verify the script has no syntax errors by running `python3 -m py_compile research_results/<product_slug>/test-api.py`.
3. List all files created with their paths.
4. Provide a concise summary to the user:
   - Product overview (1-2 sentences)
   - Recommended collection method and why
   - Number of distinct event types/data sources identified
   - Key findings or surprises
   - Gaps or areas that need user input
   - If `test-api.py` was generated: remind the user to run it against the real API (with credentials) and share the resulting archive back for development
   - Suggested next step (pass the brief to the integration build workflow for your target platform)

## Research quality standards

- **Ground all claims in sources.** Every factual statement in the brief should be traceable to vendor documentation, official specs, or widely established technical references. When using your own knowledge, explicitly note it.
- **Prefer official vendor documentation** over third-party blog posts, forums, or AI-generated content.
- **Include direct links** to source documentation wherever possible.
- **Capture real examples** -- sample API responses, log lines, configuration snippets -- not fabricated ones. If you must construct an example to illustrate structure, mark it `[CONSTRUCTED EXAMPLE]`.
- **Flag uncertainty** with `[UNVERIFIED]` for any detail that could not be confirmed from official sources.
- **Be specific, not generic.** "The API uses pagination" is not useful. "The API uses cursor-based pagination via a `next_cursor` field in the response body; pass it as the `cursor` query parameter" is useful.
- **Cover edge cases.** Note rate limits, maximum page sizes, required permissions, deprecated endpoints, known bugs, and any gotchas.

## Guardrails

- Do not fabricate sample data that looks real. Sample data must come from documentation or be clearly marked as constructed.
- Do not start building the connector or integration. This skill produces research only.
- Do not load implementation-phase skills. Those are for the build phase.
- If a product has multiple viable collection methods, document all of them with a recommendation and rationale, but produce detailed deep-dive material for the recommended method.
- If research reveals the product does not expose data in a way that any standard collection method can consume, say so clearly in the brief.

## Handoff

After this command completes, continue with:

1. **If `test-api.py` was generated** (API collection method): run the script against the real vendor API to validate connectivity and collect trace data. Share the resulting `.tar.gz` archive back — the trace file is valuable input for connector development and data normalization work.
2. Pass `research_results/<product_slug>/research-brief.md` to the integration build workflow for your target platform, providing additional sample data files from `research_results/<product_slug>/references/sample-events/` via `@`-mentions as needed.

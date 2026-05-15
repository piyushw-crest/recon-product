# Competitive SIEM Checklist

Use this checklist for **Research Track E**. Your job is to determine whether each of the four named SIEM/security platforms already ships an integration, add-on, connector, plugin, or log source for the product under research. For each platform, follow the specific search strategy below, capture the required details, and write your findings to `references/competitive-siem-coverage.md` in the working directory.

In addition, extract structured field-level mapping tables from each platform that publishes them and write them to **intermediate files in `temp/`**. These are working artifacts used by the orchestrator in Phase 6 to build the single authoritative `references/unified-field-mapping.md`:
- **Elastic** → `temp/ecs-field-mapping.md` (from GitHub ingest pipeline YAML files)
- **Google SecOps** → `temp/udm-field-mapping.md` (from parser UDM mapping table)
- **Splunk** → `temp/splunk-field-mapping.md` (only if vendor-published OCSF mapping found)
- **Sumo Logic** → `temp/sumo-field-mapping.md` (from `cloud-siem-content-catalog` mapper rules)

**Do NOT write these to `references/` — write to `temp/` only.** The orchestrator merges all four files into `references/unified-field-mapping.md` after all subagents complete.

## Platforms to check

### Elastic

**Canonical URL:** https://www.elastic.co/docs/reference/integrations/

**Where to look:**

1. **Integrations reference catalogue** — https://www.elastic.co/docs/reference/integrations/
   - Browse the full alphabetical list or use Ctrl+F / page search to find the vendor or product name.
   - If found: follow the link to the integration reference page. Note the integration slug, data streams listed, and what the description says about supported APIs and log sources.

2. **GitHub source** — https://github.com/elastic/integrations
   - Browse or search the `packages/` directory for a folder name matching the vendor or product (e.g., `packages/okta`, `packages/crowdstrike`).
   - Inside a matching package:
     - `manifest.yml` → package title, description, version, and categories
     - `data_stream/*/manifest.yml` → each data stream name and its `input` type
     - `docs/README.md` → which APIs, log sources, or endpoints are covered
     - `CHANGELOG.md` → release history, added features, known gaps

3. **Integration README cross-reference** — README files often list exactly which API endpoints or log event types are collected. Use these to populate the "Supported data types / endpoints" column in the output table.

4. **ECS field mapping — ingest pipeline YAML files (primary source):**
   - URL pattern: `https://github.com/elastic/integrations/tree/main/packages/<slug>/data_stream/<stream>/elasticsearch/ingest-pipeline/`
   - File: `default.yml` inside each data stream's `elasticsearch/ingest-pipeline/` directory
   - These YAML files contain `rename` processors (and occasionally `set` processors) that show the exact raw→ECS field mappings written by Elastic engineers. A `rename` processor has two keys: `field` (the raw source field name) and `target_field` (the ECS destination field path).
   - **Run a Python script** to parse each `default.yml` and extract all `rename` processor entries. Produce a table: `data_stream | raw source field | ECS field (target)`.
   - For `set` processors that write a static or computed value to an ECS field (no raw source field), note the ECS target field and mark the `Raw source field` column as `[set processor]`.

5. **ECS field mapping — docs "Exported fields" table (cross-check only):**
   - URL: `https://www.elastic.co/docs/reference/integrations/<slug>`
   - Each data stream section has an "Exported fields" dropdown. Use this **only** to cross-check the pipeline output and catch any ECS fields the pipeline YAML may not expose (e.g., fields populated by agent or framework outside the pipeline).
   - Filter to ECS-namespace prefixes: `email.*`, `event.*`, `source.*`, `destination.*`, `user.*`, `network.*`, `threat.*`, `url.*`, `host.*`, `file.*`, `process.*`, `dns.*`, `tls.*`, `http.*`
   - Exclude: `data_stream.*`, `event.dataset`, `event.module`, `input.type`, `log.offset`, and any `<vendor>.<product>.*` prefixed fields

**What to capture:**
- Package name / slug (e.g., `okta`, `checkpoint_harmony_endpoint`)
- Kibana integration display name
- List of data streams with a one-line description of what each collects
- Input type per data stream (REST API polling, syslog, S3, filestream, etc.)
- Current version and date of last update (from `CHANGELOG.md` or the docs page)
- Source URL (the `https://www.elastic.co/docs/reference/integrations/<slug>` page)
- Any documented gaps, deprecated streams, or "coming soon" notes
- **ECS field mapping** (see output instructions below)

**ECS field mapping output (write to two places):**
1. Include in the `### Elastic` section of `references/competitive-siem-coverage.md`
2. **Also write separately to `temp/ecs-field-mapping.md`** using this format:

```markdown
# ECS Field Mapping: <Product Name>

> Primary source: packages/<slug>/data_stream/*/elasticsearch/ingest-pipeline/default.yml
> (rename processors extracted via Python; set-processor fields cross-checked from docs Exported fields table)
> Integration page: https://www.elastic.co/docs/reference/integrations/<slug>
> Captured: <date>

| Data stream | Raw source field | ECS field (target) | Type | Description |
|-------------|------------------|--------------------|------|-------------|
| <stream> | <raw.field.name> | <ecs.field.path> | <type> | <description> |
| <stream> | [set processor] | <ecs.field.path> | <type> | <static/computed value> |
```

If no Elastic integration exists for this product, write the following to `temp/ecs-field-mapping.md`:
```
No Elastic integration found for <product> as of <date>. ECS field mapping not applicable.
Searched: https://www.elastic.co/docs/reference/integrations/ and https://github.com/elastic/integrations packages/
```

---

### Google SecOps

**Canonical URL:** https://docs.cloud.google.com/chronicle/docs/ingestion/default-parsers/

**Where to look:**

1. **Default parsers list** — https://docs.cloud.google.com/chronicle/docs/ingestion/default-parsers/
   - This page lists all standard parsers alphabetically. Search for the vendor or product name.
   - If found: follow the link to the parser's dedicated documentation page.

2. **Parser documentation page** — each parser page contains:
   - A description of the log format(s) handled (JSON, syslog, CEF, etc.)
   - The ingestion method (direct push from vendor, webhook, API connector, forwarder, etc.)
   - A **UDM mapping table** (columns: `Log field` | `UDM mapping` | `Logic`) — capture the full table
   - The `metadata.log_type` value (the Chronicle log type name used when ingesting)
   - Supported sample log formats

3. **Google SecOps Marketplace / Content Hub** — https://cloud.google.com/chronicle/docs/soar/marketplace-and-integrations/
   - Check for SOAR integrations (connectors/playbook actions) separate from the SIEM parser

**What to capture:**
- Parser page URL (e.g., `https://docs.cloud.google.com/chronicle/docs/ingestion/default-parsers/abnormal-security`)
- Chronicle log type name (`metadata.log_type`)
- Supported log formats (JSON, syslog+JSON, CEF, etc.)
- Ingestion method (Chronicle forwarder, vendor-push, webhook, etc.)
- **Full UDM mapping table** (see output instructions below)
- Any gaps — fields not mapped, log types not handled

**UDM field mapping output (write to two places):**
1. Include in the `### Google SecOps` section of `references/competitive-siem-coverage.md`
2. **Also write separately to `temp/udm-field-mapping.md`** using this format:

```markdown
# UDM Field Mapping: <Product Name>

> Source: https://docs.cloud.google.com/chronicle/docs/ingestion/default-parsers/<slug>#udm_mapping_table
> Captured from the UDM Mapping Table on the parser documentation page.

| Log field (raw) | UDM mapping | Logic / notes |
|-----------------|-------------|---------------|
| <raw_field> | <udm.object.field> | <mapping logic> |
```

If no Google SecOps parser exists for this product, write the following to `temp/udm-field-mapping.md`:
```
No Google SecOps default parser found for <product> as of <date>. UDM field mapping not applicable.
Searched: https://docs.cloud.google.com/chronicle/docs/ingestion/parser-list/supported-default-parsers
```

---

### Splunk

**Canonical URL:** https://splunkbase.splunk.com/apps?page=1

**Where to look:**

1. **Splunkbase app search** — https://splunkbase.splunk.com/apps?page=1
   - Use the search bar to find apps and add-ons for the product by vendor name and product name.
   - Filter by **Category** (e.g., "Security, Fraud & Compliance", "Endpoint", "Network Security") to narrow results.
   - Splunk add-ons (Technology Add-ons, abbreviated "TA") are the primary integration artifacts — they define field extractions, data models, and sourcetypes. Look specifically for items named "Splunk Add-on for <product>" or "TA-<product>".

2. **App detail page** — for each matching app/add-on found:
   - Note the **app name**, **version**, **last updated date**, and **Splunk version compatibility**.
   - Read the **description** and **documentation** tab for which log sources, sourcetypes, or API endpoints are supported.
   - Check the **README** or linked GitHub repository (many add-ons link to their source repo on Splunkbase).

3. **GitHub source** — search https://github.com/splunk and the vendor's own GitHub org for add-on repos (common naming: `splunk-add-on-for-<product>`, `TA-<product-slug>`).
   - Inside a repo: `README.md` describes what is collected; `default/props.conf` lists sourcetypes; `default/transforms.conf` lists field extractions.

4. **Splunk Common Information Model (CIM)** — note which CIM data models the add-on maps to (e.g., Authentication, Network Traffic, Endpoint). This indicates the data categories covered.

5. **Splunk OCSF field mapping (capture only if vendor-published):**
   - Search the TA's Splunkbase Documentation tab for an explicit OCSF field mapping table or named OCSF class.
   - Search the TA GitHub `README.md` and any `docs/` folder for an OCSF mapping table.
   - Check `https://github.com/splunk/splunk-ocsf-framework` for product-specific mapping config files (look for the vendor/product name under any `mappings/` or `configs/` directories).
   - **Do NOT infer or construct an OCSF mapping** — only capture what is explicitly published by Splunk or the add-on developer. If nothing is found, document the absence explicitly.

**What to capture:**
- Add-on / app name as listed on Splunkbase (e.g., "Splunk Add-on for CrowdStrike FDR")
- Splunkbase URL for the add-on
- Sourcetypes defined (each sourcetype typically corresponds to a log source or event category)
- CIM data models mapped (Authentication, Endpoint, Network Traffic, etc.)
- Whether it is Splunk-supported, partner-supported, or community-supported
- Any documented gaps or known limitations from the README
- **Splunk OCSF field mapping** (see output instructions below)

**Splunk OCSF field mapping output — write to `temp/splunk-field-mapping.md`:**

If an explicit OCSF field mapping table is found in the TA documentation or GitHub:

```markdown
# Splunk OCSF Field Mapping: <Product Name>

> Source: <Splunkbase URL or GitHub URL where the mapping was found>
> Captured from: <TA README / Splunkbase Documentation tab / splunk-ocsf-framework config>
> Date: <date>

## OCSF class alignment

| Sourcetype | OCSF class | Class UID | Source URL |
|------------|-----------|-----------|------------|
| <sourcetype> | <class name> | <uid> | <URL> |

## Field mapping table

| Raw field / sourcetype field | OCSF field (target) | Notes |
|------------------------------|---------------------|-------|
| <raw> | <ocsf.field> | <from TA docs> |
```

If no Splunk OCSF mapping is published by the vendor, write:

```markdown
# Splunk OCSF Field Mapping: <Product Name>

No Splunk OCSF field mapping found for <product> as of <date>.

Searched:
- Splunkbase Documentation tab: <URL>
- TA GitHub README: <URL or "not found">
- splunk/splunk-ocsf-framework: https://github.com/splunk/splunk-ocsf-framework (no product config found)

CIM data model alignment (from TA): <list CIM models if found, else "not documented">
```

Also include the Splunk OCSF content (or absence note) in the `### Splunk` section of `references/competitive-siem-coverage.md`.

---

### Sumo Logic

**Canonical URL:** https://help.sumologic.com/docs/integrations/

**Where to look:**

1. **Sumo Logic integration docs** — https://help.sumologic.com/docs/integrations/
   - Browse by category (Security, Cloud Security, Email Security, etc.) for the vendor or product name.
   - Each integration page describes the Sumo Logic App, the log sources or source categories required, and what dashboards / queries the app provides.

2. **Sumo Logic App Catalog** — https://www.sumologic.com/application/
   - Search for the vendor or product name. Each app listing shows the app name, version, and a description of what is monitored.

3. **GitHub source** — https://github.com/SumoLogic/sumologic-content
   - Browse or search for a directory matching the vendor or product name.
   - Inside a matching directory: look for `README.md` (describes what is collected and configured), dashboard JSON files (list query fields), and any parser or FER (Field Extraction Rule) definitions.

4. **Sumo Logic Cloud SIEM mapper rules (vendor-authoritative field mapping):**
   - Clone `https://github.com/SumoLogic/cloud-siem-content-catalog` into `temp/cloud-siem-content-catalog/`
   - Browse the `mappings/` directory for a subdirectory matching the vendor or product name (e.g., `mappings/AbnormalSecurity/`, `mappings/Okta/`)
   - Each mapper file is a JSON file. The key fields to extract:
     - `"vendor"` and `"product"` — confirm the match
     - `"recordType"` — the Cloud SIEM schema record type (e.g., `Email`, `Authentication`, `NetworkHTTP`)
     - `"mappings"` array — each entry has `"rawField"` (the source log field) and `"fields"` (array of Cloud SIEM schema attribute names it maps to)
   - **Run a Python script** to extract all `rawField → fields` pairs across all mapper files for the product. Produce a table: `raw log field | Cloud SIEM schema attribute(s) | record type`
   - The Cloud SIEM schema attributes are documented at: `https://help.sumologic.com/docs/cse/schema/schema-attributes/`

**What to capture:**
- App name as listed in Sumo Logic docs or App Catalog
- Sumo Logic docs URL for the app
- Log sources / source categories required (e.g., `abnormal_security`, HTTP source, S3 source)
- Dashboards provided (list titles — they indicate what data is usable)
- Field Extraction Rules (FERs) if documented — field names extracted from raw logs
- Cloud SIEM record type(s) (from mapper rule `"recordType"` field)
- Whether it is officially supported by Sumo Logic or community-maintained
- Any documented gaps or known limitations
- **Sumo Logic Cloud SIEM field mapping** (see output instructions below)

**Sumo Logic Cloud SIEM field mapping output — write to `temp/sumo-field-mapping.md`:**

If mapper rules are found in `cloud-siem-content-catalog`:

```markdown
# Sumo Logic Cloud SIEM Field Mapping: <Product Name>

> Source: https://github.com/SumoLogic/cloud-siem-content-catalog
> Mapper file(s): mappings/<vendor>/<product>/<mapper-file>.json
> Cloud SIEM record type: <recordType>
> Schema reference: https://help.sumologic.com/docs/cse/schema/schema-attributes/
> Date: <date>

## Field mapping table

| Raw log field | Cloud SIEM schema attribute(s) | Record type |
|---------------|-------------------------------|-------------|
| <rawField> | <schema attribute> | <recordType> |
```

If no mapper rule is found:

```markdown
# Sumo Logic Cloud SIEM Field Mapping: <Product Name>

No Sumo Logic Cloud SIEM mapper rule found for <product> in SumoLogic/cloud-siem-content-catalog as of <date>.
Searched mappings/ directory for: <vendor name>, <product name>
```

Also include the Sumo Logic mapper content (or absence note) in the `### Sumo Logic` section of `references/competitive-siem-coverage.md`.

---

## Output format

Write all findings to `references/competitive-siem-coverage.md` using this structure:

```markdown
# Competitive SIEM Integration Coverage: <Product Name>

> Researched: <date>

## Summary table

| Integration name | Vendor | Supported data types / log sources / endpoints | Schema | Collection method | Source URL | Notes / gaps |
|-----------------|--------|------------------------------------------------|--------|-------------------|------------|--------------|
| <name> | <Elastic / Google SecOps / Splunk / Sumo Logic> | <list> | <ECS / UDM / OCSF/CIM / OCSF> | <REST API / syslog / S3 / etc.> | <link> | <gaps, version, deprecated?> |
| None found | Splunk | — | — | — | https://splunkbase.splunk.com/apps?page=1 | No add-on found as of <date> |

## Detailed findings

### Elastic
<findings — package slug, data streams and input types, version, README summary of covered APIs, source URL>

#### ECS field mapping
<include the ECS fields table here — same content as written to temp/ecs-field-mapping.md>

### Google SecOps
<findings — parser page URL, log type name, supported formats, ingestion method, gaps>

#### UDM field mapping
<include the UDM mapping table here — same content as written to temp/udm-field-mapping.md>

### Splunk
<findings — add-on name, Splunkbase URL, sourcetypes, CIM data models mapped, version, support tier>

#### Splunk OCSF field mapping
<include content from temp/splunk-field-mapping.md — either the mapping table or the absence note>

### Sumo Logic
<findings — app name, docs URL, source categories, dashboards, FERs, version, support tier>

#### Sumo Logic Cloud SIEM field mapping
<include content from temp/sumo-field-mapping.md — either the mapping table or the absence note>
```

**If a platform has no integration:** write a one-paragraph entry under its heading confirming what was searched (URLs visited, search terms used), what was not found, and the date — so the absence is documented explicitly, not left as a gap.

---

## Checklist

- [ ] Elastic integrations reference catalogue searched by vendor name: https://www.elastic.co/docs/reference/integrations/
- [ ] Elastic integrations reference catalogue searched by product name
- [ ] Elastic GitHub `elastic/integrations` `packages/` directory checked for matching folder
- [ ] Elastic README cross-referenced for covered API endpoints / log event types
- [ ] Elastic ingest pipeline YAML fetched for each data stream (`packages/<slug>/data_stream/<stream>/elasticsearch/ingest-pipeline/default.yml`)
- [ ] Elastic `rename` processors extracted via Python script → raw→ECS mapping table built
- [ ] Elastic `set`-processor ECS fields cross-checked against docs Exported fields table
- [ ] ECS field mapping written to `temp/ecs-field-mapping.md`
- [ ] Google SecOps default parsers list searched by vendor name: https://docs.cloud.google.com/chronicle/docs/ingestion/parser-list/supported-default-parsers
- [ ] Google SecOps default parsers list searched by product name
- [ ] Google SecOps parser documentation page fetched and UDM mapping table captured
- [ ] Google SecOps Content Hub / Marketplace checked for SOAR integrations
- [ ] UDM field mapping written to `temp/udm-field-mapping.md`
- [ ] Splunk Splunkbase searched by vendor name: https://splunkbase.splunk.com/apps?page=1
- [ ] Splunk Splunkbase searched by product name
- [ ] Splunk add-on GitHub source repo checked (if linked from Splunkbase or found in https://github.com/splunk)
- [ ] Splunk CIM data models noted
- [ ] Splunk TA Splunkbase Documentation tab checked for explicit OCSF mapping table
- [ ] Splunk TA GitHub README and docs/ folder checked for OCSF mapping
- [ ] `splunk/splunk-ocsf-framework` GitHub repo checked for product-specific config: https://github.com/splunk/splunk-ocsf-framework
- [ ] Splunk OCSF field mapping written to `temp/splunk-field-mapping.md` (or absence documented)
- [ ] Sumo Logic integration docs searched: https://help.sumologic.com/docs/integrations/
- [ ] Sumo Logic App Catalog searched: https://www.sumologic.com/application/
- [ ] Sumo Logic GitHub `SumoLogic/sumologic-content` repo checked for matching directory
- [ ] `SumoLogic/cloud-siem-content-catalog` cloned to `temp/cloud-siem-content-catalog/`
- [ ] `mappings/` directory searched for vendor/product name
- [ ] Mapper JSON parsed via Python script → `rawField → fields` mapping table extracted
- [ ] Cloud SIEM record type noted from mapper `"recordType"` field
- [ ] Sumo Logic Cloud SIEM field mapping written to `temp/sumo-field-mapping.md` (or absence documented)
- [ ] Findings written to `references/competitive-siem-coverage.md`
- [ ] Summary returned with one-line status per platform (found / not found) and source URL per entry
- [ ] Confirmation returned that all four intermediate mapping files were written to `temp/`: `ecs-field-mapping.md`, `udm-field-mapping.md`, `splunk-field-mapping.md`, `sumo-field-mapping.md`

---

## Note for the orchestrator

The four files in `temp/` written by this subagent (`ecs-field-mapping.md`, `udm-field-mapping.md`, `splunk-field-mapping.md`, `sumo-field-mapping.md`) are **intermediate working artifacts**. The orchestrator reads these files in Phase 6 alongside `temp/field-catalog.md` and `temp/data-model-intermediate.md` to build the single authoritative file `references/unified-field-mapping.md`.

The unified file contains:
- Per-schema source metadata blocks (integration found, source URLs, version, extraction method, gaps/caveats)
- A 19-column master table: `Raw field | Description | Data type | Category | ECS field | ECS type | ECS description | UDM field | UDM type | UDM description | UDM logic | Splunk OCSF field | Splunk OCSF class | Splunk OCSF type | Splunk OCSF description | Sumo CSE attribute | Sumo record type | Sumo CSE type | Sumo CSE description`
- An `### Unmatched schema mappings` section for schema fields that could not be joined to a raw product field

Do not write to `references/unified-field-mapping.md` from this subagent — that is the orchestrator's responsibility.

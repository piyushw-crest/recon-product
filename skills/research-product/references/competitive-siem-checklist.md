# Competitive SIEM Checklist

Use this checklist for **Research Track E**. Your job is to determine whether each of the four named SIEM/security platforms already ships an integration, add-on, connector, plugin, or log source for the product under research. For each platform, follow the specific search strategy below, capture the required details, and write your findings to `references/competitive-siem-coverage.md` in the working directory.

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

**What to capture:**
- Package name / slug (e.g., `okta`, `checkpoint_harmony_endpoint`)
- Kibana integration display name
- List of data streams with a one-line description of what each collects
- Input type per data stream (REST API polling, syslog, S3, filestream, etc.)
- Current version and date of last update (from `CHANGELOG.md` or the docs page)
- Source URL (the `https://www.elastic.co/docs/reference/integrations/<slug>` page)
- Any documented gaps, deprecated streams, or "coming soon" notes

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

**What to capture:**
- Add-on / app name as listed on Splunkbase (e.g., "Splunk Add-on for CrowdStrike FDR")
- Splunkbase URL for the add-on
- Sourcetypes defined (each sourcetype typically corresponds to a log source or event category)
- CIM data models mapped (Authentication, Endpoint, Network Traffic, etc.)
- Current version and last updated date
- Whether it is Splunk-supported, partner-supported, or community-supported
- Any documented gaps or known limitations from the README

---

### Panther

**Canonical URL:** https://panther.com/integrations/overview

**Where to look:**

1. **Panther integrations overview page** — https://panther.com/integrations/overview
   - Browse the "Log Sources" category for the vendor or product name.
   - Each log source listing links to a dedicated page describing the integration, transport method, and what is monitored.

2. **Panther documentation — supported log sources** — https://docs.panther.com/data-onboarding/supported-logs
   - Searchable reference for all supported log sources. Each entry names the log source and links to the schema definition.

3. **panther-analysis GitHub repository** — https://github.com/panther-labs/panther-analysis
   - Look in the `schemas/` directory (or `log_types/`) for YAML schema files matching the product name.
   - Check the `rules/` directory for detection rules referencing this product — the presence of detection rules confirms which log types are actively ingested.
   - Schema YAML files define the exact fields Panther parses; capture the schema name, field list summary, and any `ReferenceURL` noted.

**What to capture:**
- Log source / schema name as listed in Panther (e.g., `Okta.SystemLog`, `CrowdStrike.FDREvent`)
- Link from the integrations overview page for this log source
- Transport / delivery method (S3, HTTP endpoint, syslog, direct API pull, etc.)
- Top-level event types or schema fields documented
- Whether detection rules exist for this source in `panther-analysis` (signals production use)
- Current schema version

---

### Rapid7

**Canonical URL:** https://extensions.rapid7.com/extension/

**Where to look:**

1. **Rapid7 Extensions library** — https://extensions.rapid7.com/extension/
   - Browse or search by vendor/product name. Each entry is an InsightConnect plugin or InsightIDR extension.
   - Each plugin page lists the **actions** and **triggers** it supports; actions map to outbound API calls, triggers map to inbound events or polling.

2. **insightconnect-plugins GitHub repository** — https://github.com/rapid7/insightconnect-plugins
   - Browse the `plugins/` directory for a subdirectory matching the vendor or product name (e.g., `plugins/okta`, `plugins/crowdstrike_falcon`).
   - Inside a plugin directory: read `plugin.spec.yaml` for the plugin title, description, and full list of `actions` and `triggers`. Each action/trigger documents its inputs, outputs, and the underlying API endpoint it calls.

3. **InsightIDR native log sources** — https://docs.rapid7.com/insightidr/log-sources
   - InsightIDR (Rapid7's SIEM) has native log source parsers separate from InsightConnect automation plugins. Search the page for the product name.
   - These define which event types InsightIDR can ingest and parse natively.

**What to capture:**
- Plugin name and ID from the Extensions library (e.g., `crowdstrike_falcon`, version)
- Extension library URL for the plugin
- Actions supported (e.g., "Get Detections", "List Incidents") — each maps to an API endpoint
- Triggers supported (e.g., "New Detection") — indicate webhook or polling support
- Whether an InsightIDR native log source parser also exists (document separately)
- Current plugin version and last updated date

---

## Output format

Write all findings to `references/competitive-siem-coverage.md` using this structure:

```markdown
# Competitive SIEM Integration Coverage: <Product Name>

> Researched: <date>

## Summary table

| Integration name | Vendor | Supported data types / log sources / endpoints | Collection method | Source URL | Notes / gaps |
|-----------------|--------|------------------------------------------------|-------------------|------------|--------------|
| <name> | <Elastic / Splunk / Panther / Rapid7> | <list> | <REST API / syslog / S3 / etc.> | <link> | <gaps, version, deprecated?> |
| None found | Splunk | — | — | https://splunkbase.splunk.com/apps?page=1 | No add-on found as of <date> |

## Detailed findings

### Elastic
<findings — package slug, data streams and input types, version, README summary of covered APIs, source URL>

### Splunk
<findings — add-on name, Splunkbase URL, sourcetypes, CIM data models mapped, version, support tier>

### Panther
<findings — schema/log source name, integrations overview URL, transport, field summary, detection rules present?>

### Rapid7
<findings — plugin name, extensions URL, actions/triggers list, InsightIDR parser if any, version>
```

**If a platform has no integration:** write a one-paragraph entry under its heading confirming what was searched (URLs visited, search terms used), what was not found, and the date — so the absence is documented explicitly, not left as a gap.

---

## Checklist

- [ ] Elastic integrations reference catalogue searched by vendor name: https://www.elastic.co/docs/reference/integrations/
- [ ] Elastic integrations reference catalogue searched by product name
- [ ] Elastic GitHub `elastic/integrations` `packages/` directory checked for matching folder
- [ ] Elastic README cross-referenced for covered API endpoints / log event types
- [ ] Splunk Splunkbase searched by vendor name: https://splunkbase.splunk.com/apps?page=1
- [ ] Splunk Splunkbase searched by product name
- [ ] Splunk add-on GitHub source repo checked (if linked from Splunkbase or found in https://github.com/splunk)
- [ ] Panther integrations overview page checked: https://panther.com/integrations/overview
- [ ] Panther docs supported log sources page checked: https://docs.panther.com/data-onboarding/supported-logs
- [ ] `panther-labs/panther-analysis` GitHub repo checked for schema and detection rule files
- [ ] Rapid7 Extensions library searched: https://extensions.rapid7.com/extension/
- [ ] `rapid7/insightconnect-plugins` GitHub repo checked for matching plugin directory
- [ ] InsightIDR native log sources page checked: https://docs.rapid7.com/insightidr/log-sources
- [ ] Findings written to `references/competitive-siem-coverage.md`
- [ ] Summary returned with one-line status per platform (found / not found) and source URL per entry

# Recon Agent Workspace

Internal repo for AI-assisted product research — thoroughly investigating vendor products, APIs, log sources, field schemas, sample data, and competitive SIEM coverage before any integration build.

## Requirements

| Tool | Purpose | Install |
|------|---------|---------|
| [Git](https://git-scm.com/) | Clone this repository | Follow [official docs](https://git-scm.com/downloads) |
| [Python 3](https://www.python.org/downloads/) | Run generated `test-api.py` connectivity scripts | Follow [official docs](https://www.python.org/downloads/) |

Generated `test-api.py` scripts use the Python standard library only — no `pip install` required.

## Installing the skills and agents

The only requirement is to clone this repository. The `skills/` and `agents/` directories at the repo root contain the skill library and agent definitions.

Copy both folders into the tool config directory you use. Typical locations are `~/.cursor` or `~/.claude` in your home directory if you want skills available in every project, or a project-specific `.cursor` or `.claude` directory at the workspace root if you only need them for one codebase.

**Example — install into your home config (global):**

If `~/.cursor` does not exist yet, create it first (`mkdir -p ~/.cursor`). Then:

```bash
git clone https://github.com/your-org/recon_agent.git
cd recon_agent
cp -r skills agents ~/.cursor/
```

Use `~/.claude` instead of `~/.cursor` in the copy destination if that is what your environment expects. For a single project, copy into `/path/to/your/project/.cursor` or `.claude` instead (create that directory first if it is missing).

> **Note:** `research_results/` is gitignored. All research outputs are written locally and never committed.

## What you can research

`/research-product` accepts a wide range of inputs and scopes:

| Input type | How to provide | Examples |
|------------|----------------|----------|
| Full product analysis | Product / vendor name | `"Checkpoint Harmony Endpoint"`, `"Okta System Log"` |
| Specific entity or log type | Free text scope constraint | `"alerts API only"`, `"focus on firewall and IDS logs"`, `"audit events only"` |
| Specific collection method | Free text | `"REST API"`, `"syslog"`, `"S3/SQS"`, `"Azure Event Hub"` |
| Documentation URLs | Paste URLs inline | `https://docs.vendor.com/api/v2` |
| Local reference files | `@`-mention files | `@samples/vendor_event.json`, `@notes/vendor-api-notes.md` |
| Custom output slug | Free text | `"checkpoint_harmony"` (defaults to sanitised product name) |

Anything typed after `/research-product` is your research goal. Pass as much or as little as you know — the agent will determine the collection method and fill gaps automatically.

## Supported data collection methods

The skill covers all three major collection categories:

- **REST API** — authentication deep-dive (API key, OAuth2 all grant types, Bearer, Basic, custom headers), pagination patterns (cursor, offset, link-header, token-based, keyset), rate limits, request/response structure, OpenAPI/Swagger spec download and programmatic analysis
- **Syslog / log files** — RFC 3164/5424, CEF, LEEF, JSON, key-value, CSV, and multiline formats; log file paths, syslog facilities, sample log lines per event type
- **Cloud ingest** — S3/SQS, Azure Event Hub, Google Pub/Sub, Kafka; delivery configuration, message/object format, path and prefix patterns, schema repository cloning and analysis

If the collection method is unknown at invocation time, the agent investigates all options and recommends the best one.

## Main Skills

### `/research-product`

The primary research skill. Investigates a vendor, product, or log source end-to-end by launching parallel `deep-research` subagents across five research tracks, then synthesises their findings into a structured brief.

**Research tracks:**

- **Track A — Product overview:** what the product is, all available data collection methods, which is best suited for a programmatic integration, and known limitations or licensing requirements
- **Track B — Data source deep dive:** for REST APIs — endpoint paths, authentication flows (full OAuth2 grant-type investigation), pagination, rate limits, complete request/response examples, OpenAPI spec download and analysis; for log sources — format, sample lines, syslog structure; for cloud ingest — delivery config, message format, schema repo analysis
- **Track C — Event types and field schema:** all distinct event types and log sources, field names/types/descriptions, enumeration values, timestamp formats, nested structures; downloads SDKs or schema repos and analyzes them programmatically for large schemas; saves representative sample events (`.json` / `.log`) to `references/sample-events/` — one file per event type
- **Track D — Configuration and deployment:** user-facing configuration variables, vendor-side setup steps, network requirements, permissions
- **Track E — Competitive SIEM coverage:** checks **Elastic**, **Splunk**, **Panther**, and **Rapid7** for existing integrations with the product; captures integration name, supported log sources, collection method, version, and gaps; explicitly documents absence when a platform has no coverage

Writes a self-contained `research-brief.md` and all companion artifacts to `research_results/<product_slug>/`. For REST APIs, also generates a `test-api.py` connectivity and pagination validation script (stdlib only, CLI + environment variable credential input, step-by-step stdout, `trace.json` + `.tar.gz` archive output).

**Invocation examples:**

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

---

### `/anonymize-logs`

Sanitises raw log samples line-by-line without altering structure. Replaces PII, credentials, IP addresses, hostnames, and other sensitive values with realistic placeholders. Works with NDJSON, syslog, and multiline log formats.

Use this before sharing logs for review or including them in research output as sample events.

---

### `deep-research` (background agent)

Not invoked directly. Spawned in parallel by `/research-product` to handle web search, page fetching, repository cloning, and Python-based schema analysis. Writes large findings — field inventories, schema analyses, sample events — to files under `references/` or `temp/` and returns concise summaries with file paths to keep context manageable.

## Output layout

`/research-product` writes all output under `research_results/<product_slug>/`, where `<product_slug>` is a lowercase, underscore-separated identifier (e.g. `checkpoint_harmony_endpoint`, `palo_alto_cortex_xdr`).

```
research_results/<product_slug>/
  research-brief.md               # primary deliverable — self-contained structured brief
  test-api.py                     # API connectivity & pagination test script (REST APIs only)
  data-model-analysis.md          # field categorisation and normalisation candidates
  configuration-plan.md           # connector configuration variables and defaults
  references/
    api-spec-notes.md             # endpoint details, request/response examples (REST APIs)
    log-format-notes.md           # log format details, sample lines (log-based sources)
    field-schema-analysis.md      # complete field inventories, types, enumerations
    competitive-siem-coverage.md  # Elastic, Splunk, Panther, Rapid7 coverage findings
    sample-events/
      <event_type>.json           # one file per event type or format variant
      <event_type>.log
  temp/                           # raw downloaded artifacts (repos, SDKs, specs, HTML docs)
```

Not all files are created for every product — only what applies to the collection method. The `temp/` directory is kept after research completes as a reference for follow-up work. Everything under `research_results/` is gitignored and stays local.

## Next steps after research

1. **REST APIs only:** run `test-api.py` against the real vendor API to validate connectivity, authentication, and pagination. Share the resulting `.tar.gz` archive — the trace file is valuable input for connector development and data normalisation work.
2. Pass `research_results/<product_slug>/research-brief.md` to the integration build workflow for your target platform. `@`-mention sample event files from `references/sample-events/` as additional context.

# Data Collection Methods

This reference describes the collection method types available for building connectors and integrations, what each is used for, and what research information is needed for each.

Use this to determine which collection method fits the product being researched and what details to investigate.

## Collection method decision tree

```
Does the vendor expose a REST/HTTP API for retrieving events?
  YES → REST API polling
  NO ↓

Does the product write local log files?
  YES → Local log file collection
  NO ↓

Does the product send syslog messages?
  YES → Syslog receiver (TCP/UDP)
  NO ↓

Does the vendor deliver data to a cloud message queue or object store?
  S3 bucket → S3 object store (with optional SQS notification)
  Azure Event Hub → Azure Event Hub stream
  Google Pub/Sub → GCP Pub/Sub stream
  Azure Blob Storage → Azure Blob Storage polling
  GCS bucket → GCS object store
  Kafka topic → Kafka consumer
  NO ↓

Does the vendor push data to a webhook/HTTP endpoint?
  YES → Webhook receiver (HTTP endpoint)
  NO ↓

Can data be exported as flat files (CSV, JSON) and dropped to a path?
  YES → Local log file collection
  NO → product may not expose data via any standard collection method; document the gap
```

## Collection methods reference

### REST API Polling

**When to use:** The product exposes a REST API for retrieving events, logs, or metrics on demand.

**What to research:**
- Base URL and API version
- Authentication method and credential types
- All relevant endpoints (list with paths)
- Request parameters: required, optional, filtering, time range
- Response format: JSON structure, envelope vs. array, nested objects
- Pagination: mechanism (offset, cursor, link-header, keyset, page number), field names, termination condition
- Rate limiting: limits, headers, retry-after behavior
- Timestamp handling: format, timezone, field names for time-range queries
- Error response format and status codes
- Webhook alternative (some products offer both pull and push)
- API permissions/scopes required
- OpenAPI/Swagger spec or SDK availability

**Key research outputs:**
- Complete endpoint list with auth, pagination, and time-filter details
- Sample request/response pairs for each endpoint
- Incremental collection strategy (cursor, timestamp bookmark, or snapshot-only)

---

### Local Log File Collection

**When to use:** The product writes log files to disk on the host where the collector runs.

**What to research:**
- Default log file paths per OS (Linux, Windows, macOS)
- Log format: syslog, JSON/NDJSON, CSV, key-value, multiline, custom delimited
- Log rotation behavior (size, time, naming pattern)
- Character encoding
- Multiline patterns (if applicable): what constitutes a single event
- All distinct log types/files and what events each contains
- Timestamp format within log lines
- Sample log lines for each event type

**Key research outputs:**
- File path glob patterns per OS
- Format description sufficient to write a parser
- Representative sample log lines

---

### Syslog Receiver (TCP/UDP)

**When to use:** The product sends syslog messages over the network to a listener.

**What to research:**
- Syslog RFC version: 3164 (BSD) or 5424 (IETF)
- Message format inside syslog envelope: CEF, LEEF, key-value, free text, JSON
- Syslog facility and severity usage
- Default source port(s)
- Whether the product supports TLS for syslog
- Timezone handling: are timestamps in UTC or local? Does the message include timezone?
- Message structure and delimiter patterns
- All distinct event types by facility, severity, or message ID
- Sample syslog lines for each event type

**Key research outputs:**
- Protocol and port details
- Message format description and parsing pattern sketch
- Representative sample syslog lines per event type

---

### S3 Object Store (with optional queue notification)

**When to use:** The vendor or cloud service delivers data as objects in an S3 bucket, optionally with SQS notifications.

**What to research:**
- Object format: JSON, NDJSON, CSV, gzip-compressed, Parquet
- Object path/prefix pattern and partitioning scheme
- Whether objects contain single events or batches
- Object naming convention and timestamp encoding in path
- SQS notification configuration (if used)
- IAM permissions required
- Cross-account access patterns
- Data retention and lifecycle policies
- Sample object content

**Key research outputs:**
- Bucket/prefix structure and naming convention
- Object format and envelope structure
- IAM policy requirements
- SQS vs. polling trade-offs

---

### Azure Event Hub Stream

**When to use:** The vendor or Azure service streams data through Azure Event Hubs.

**What to research:**
- Event Hub namespace and hub name configuration
- Consumer group setup
- Message format: JSON envelope, nested records, batch arrays
- Authentication: connection string, managed identity, SAS token
- Partitioning scheme
- Schema of individual events within the Event Hub message
- Storage account for checkpointing
- Sample event content

**Key research outputs:**
- Event Hub topology and naming conventions
- Message envelope structure
- Authentication options and required permissions
- Checkpoint storage requirements

---

### GCP Pub/Sub Stream

**When to use:** The vendor or GCP service streams data through Google Cloud Pub/Sub.

**What to research:**
- Pub/Sub topic and subscription configuration
- Message format and attributes
- Authentication: service account JSON key, workload identity federation
- Required IAM roles
- Ordering requirements
- Dead letter topic configuration
- Schema of individual messages
- Sample message content

**Key research outputs:**
- Topic/subscription topology
- Message structure and attributes
- Service account and IAM requirements

---

### Azure Blob Storage

**When to use:** Data is delivered as blobs in Azure Storage containers.

**What to research:**
- Container name and blob path/prefix patterns
- Blob format: JSON, NDJSON, CSV, gzip
- Authentication: connection string, SAS token, managed identity
- Blob naming convention and partitioning
- Poll interval and change detection approach
- Sample blob content

---

### GCS Object Store

**When to use:** Data is delivered as objects in GCS buckets.

**What to research:**
- Bucket name and object prefix patterns
- Object format and compression
- Authentication: service account
- Object naming and partitioning
- Pub/Sub notifications for new objects (optional)
- Sample object content

---

### Kafka Consumer

**When to use:** Data is available on Kafka topics.

**What to research:**
- Topic name(s) and partitioning strategy
- Message format: JSON, Avro, Protobuf
- Authentication: SASL (PLAIN, SCRAM, GSSAPI), mTLS, no auth
- Consumer group configuration
- Schema registry URL and compatibility mode (if Avro/Protobuf)
- Offset management approach
- Sample messages

**Key research outputs:**
- Topic names and partition count
- Message format and schema registry details (if applicable)
- Authentication mechanism

---

### Webhook Receiver (HTTP Endpoint)

**When to use:** The vendor pushes data to an HTTP endpoint that the collector listens on.

**What to research:**
- Webhook payload format (JSON body, form data)
- Authentication of incoming requests (HMAC signature, shared secret, mTLS)
- Event delivery guarantees (at-least-once, retry behavior)
- Webhook registration/configuration on the vendor side
- Payload structure for each event type
- Rate and size limits on the vendor side
- Sample webhook payloads

**Key research outputs:**
- Payload format per event type
- Signature/auth verification mechanism
- Retry and delivery guarantee behavior

---

## Multiple collection methods

Many products support more than one collection method. When this is the case:

1. Document all available methods.
2. Recommend the best method based on:
   - **Completeness**: which method provides the most event types and field detail
   - **Timeliness**: which has the lowest latency from event occurrence to collection
   - **Reliability**: which has the best delivery guarantees
   - **Simplicity**: which requires the least user configuration
   - **Adoption**: which method is most commonly used by existing third-party integrations for this product (check Splunk add-ons, Sentinel connectors, open-source clients)
3. If two methods are close in quality, consider documenting both with detailed deep-dive material for the recommended one and a summary for the alternative.

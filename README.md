# Web-Based ETL System Design

A practical guide to turning external API data into a service people pay a monthly subscription to use.

## Table of contents

- [1. Quick Problem Description](#1-quick-problem-description)
- [2. Observability/Monitoring/Alerting](#2-observabilitymonitoringalerting)
- [3. Logging](#3-logging)
- [4. System Components](#4-system-components)
  - [4.1 Data Ingest](#41-data-ingest)
    - [4.1.1 Managed Secrets Store](#411-managed-secrets-store)
    - [4.1.2 Resilient Reader Fleet](#412-resilient-reader-fleet)
    - [4.1.3 Resilient Message Queue](#413-resilient-message-queue)
    - [4.1.4 Resilient DB Writer](#414-resilient-db-writer)
    - [4.1.5 Failed Ingest Messages / Quarantine](#415-failed-ingest-messages--quarantine)
    - [4.1.6 Monitoring/Alerting](#416-monitoringalerting)
  - [4.2 Database](#42-database)
    - [4.2.1 Service Discovery and Credentials](#421-service-discovery-and-credentials)
    - [4.2.2 Primary DB and Failover Standby](#422-primary-db-and-failover-standby)
    - [4.2.3 Read Replicas](#423-read-replicas)
    - [4.2.4 Backup and Restore](#424-backup-and-restore)
    - [4.2.5 Monitoring/Alerting](#425-monitoringalerting)
  - [4.3 Resilient Logging Subsystem](#43-resilient-logging-subsystem)
    - [4.3.1 Monitoring/Alerting](#431-monitoringalerting)
- [License](#license)

## 1. Quick Problem Description

External data sources provide updates through REST polling, WebSocket push, or other supported methods. Our service collects them, adds value, and makes the results available to paying subscribers. The ETL pipeline **extracts** (fetches), **transforms** (prepares), and **loads** (stores) the data; subscriber requests use our API to access the results.

<!-- When the diagram changes, update this URL to the commit containing the new SVG to avoid stale cached images. -->
![System overview: external data sources 1 through N each connect to a dedicated resilient reader fleet. Readers extract data into the queue; the DB writer validates, transforms, and loads it. A failed-message quarantine provides controlled replay. The shared Managed Secrets Store sits between the DB writer and middleware. The database is centered between ingestion and subscriber-query columns, with elbow connectors entering its left and right sides. The Resilient Logging Subsystem sits at the lower left and collects logs from every component.](https://raw.githubusercontent.com/SixbucksSolutions/web-based-etl-system-design/f283c4b3429cfbcf3fb8fc0c67fae4a5899f4408/docs/system-overview.svg)

The dotted web service box marks our sphere of control; the providers and subscribers are outside it. A separate dotted box groups the external data sources. Each source connects to its own reader fleet using the transfer methods it supports. The Managed Secrets Store holds credentials and connection profiles between the DB writer and middleware; eight short, double-ended dotted arrows represent requests and responses from components that need those settings. Readers perform extraction; the DB writer validates, transforms, and loads observations. Its failed-message side path retains unprocessable work for investigation and controlled replay through the queue. Both fleets feed the shared queue and DB writer, and the ingestion and query paths connect to the database through right-angle side connections. At the lower left, eight short dotted arrows into the Resilient Logging Subsystem represent logs from every component going to one shared destination. The API and middleware boxes show responsibilities; middleware typically runs before an endpoint handler and may also process its response.

## 2. Observability/Monitoring/Alerting

Design [monitoring and alerting](https://sre.google/sre-book/monitoring-distributed-systems/) from the start. The owner should learn about problems **no later than customers do**. At every layer, ask what can break, how we'll detect it, what the system should do, and when to notify the owner. A dashboard alone won't wake anyone up. Every component in section 4 closes with **Monitoring/Alerting**, covering what to watch and when to act as we design that component.

**Silence can be its own problem.** Send recurring test alerts through production detection, routing, and delivery; record arrival at the destination before filtering them out of routine human notifications. If that heartbeat stops, an independent watchdog must raise its **own alert**, potentially **out of band (OOB)** through a channel that doesn't depend on the primary alerting system. Occasionally test actual owner notifications too.

## 3. Logging

**Every component sends logs to exactly one shared, searchable destination: the Resilient Logging Subsystem.** **Prefer a managed logging service over self-hosting — RECOMMENDED**, to reduce infrastructure and maintenance work; [section 4.3](#43-resilient-logging-subsystem) gives examples. One logical destination can have redundant collectors and storage; it must not depend on one host. Correlated logs are essential for diagnosing failures across a distributed system. Use [structured fields](https://opentelemetry.io/docs/specs/otel/logs/data-model/): UTC timestamp, component, instance, severity, and an execution ID (for example, a GUID). Carry a shared correlation or trace ID across requests and queue messages, alongside each execution's own ID, so one search follows the work across readers, writers, and API calls. Nobody should have to guess where a component put its logs.

**Everything must log enough, at the right level, to reconstruct what happened.** Record meaningful starts, outcomes, durations, retries, and failures. Use consistent [levels](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging/overview#log-level): **Trace/Debug** for detailed diagnostics, **Information** for normal milestones, **Warning** for recoverable degradation, **Error** for failed operations, and **Critical** when a component cannot function. Filtering by component, execution, and severity should make an incident understandable without drowning it in noise. Expected duplicate writes are normal outcomes, not errors. Keep secrets and sensitive payloads out. Buffer and retry log delivery within limits; monitor dropped logs and missing test events, with independent alerting if the logging subsystem fails.

## 4. System Components

### 4.1 Data Ingest

Move source updates through readers and a message queue into the database.

#### 4.1.1 Managed Secrets Store

A prototype may keep credentials in code or plaintext `.env`/deployment configs. A stolen key or bearer token may impersonate the customer within its permissions. Use an access-controlled [Managed Secrets Store](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html), limit reader access, and plan provider-supported rotation and revocation. **Startup-injected environment variables normally require reader restarts to pick up credential changes; fetching the current secret from the Managed Secrets Store for each collection cycle can pick up rotations without restarting.** Any caching needs an explicit refresh policy. A managed service can improve availability and reduce maintenance; DigitalOcean documents a [Secrets Manager CLI](https://docs.digitalocean.com/reference/doctl/reference/secrets/) and [API](https://docs.digitalocean.com/reference/api/reference/security/) for readers on Droplets. Protect credentials used to access the Managed Secrets Store, check regional resilience, and keep secrets out of code, images, browser bundles, and logs.

Named **connection profiles** can bundle a service's hostname or URL, port, TLS settings, and credentials in this same store; [AWS uses this pattern for database secrets](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_secret_json_structure.html). A separate configuration service is not required for these settings. Deployment supplies the store location, profile names, and a securely provisioned identity for accessing them. Limit access per workload and prefer provider-supported workload identity where available. Use bounded caching with an explicit refresh policy; stable DNS names and managed endpoints handle changing service nodes.

#### 4.1.2 Resilient Reader Fleet

Readers own **Extract (E)**: collect source payloads and publish them with source identity, available source timestamps, receive time, and a correlation ID. The aim is to survive failures without customers noticing an interruption in data flow. Each data source has its own fleet; each reader independently collects updates using that source's supported protocol and publishes its copy to the queue. Spread readers across [independent failure domains](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/availability-zones.html)—preferably separate availability zones (AZs), or separate regions when regional outages are in scope. Healthy readers can keep collecting while a failed host or network link recovers. Budget redundant calls against the provider's shared rate limits, and handle duplicate or out-of-order updates safely downstream.

Each reader retrieves its named **queue-publisher connection profile** from the [Managed Secrets Store](#411-managed-secrets-store). For RabbitMQ, this includes the broker's stable hostname, port, TLS settings, and publisher credentials; for SQS, it includes the queue URL and region, with authorization through the workload's AWS identity. Readers publish through that endpoint without tracking individual queue nodes. Refresh cached settings and reconnect when needed.

#### 4.1.3 Resilient Message Queue

The queue gives us two key advantages:

- **Loose coupling:** Readers publish messages; DB writers consume them. That simple transfer contract lets either component be changed or replaced independently, provided the message format and delivery expectations stay compatible.
- **Backpressure:** When the database is heavily loaded, writers can slow consumption and keep in-flight work bounded. The [queue absorbs the backlog](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) while readers keep collecting and publishing; readers do not need to handle database write timeouts. This buffer has limits: monitor queue capacity and message age so a prolonged slowdown does not exhaust storage or exceed the freshness budget.

Message queuing is a well-solved problem. **Prefer a managed queue service over self-hosting — RECOMMENDED**, so the provider handles more of the infrastructure, maintenance, and availability work. [Amazon SQS](https://aws.amazon.com/sqs/) and hosted RabbitMQ are examples; choose a service that fits your environment and delivery requirements. The aim is for every reader's copy to reach a DB writer, with redundant copies protecting an observation if one is lost before queue acceptance. Check successful enqueueing, retry uncertain sends, and expect duplicate deliveries. For successful processing, writers acknowledge or delete messages only after database commit; failed work follows the [quarantine path](#415-failed-ingest-messages--quarantine).

[RabbitMQ](https://www.rabbitmq.com/docs/download) with [quorum queues](https://www.rabbitmq.com/docs/quorum-queues) remains our concrete example, rather than a required product. If self-hosting, use a [three-node cluster](https://www.rabbitmq.com/docs/clustering) across separate hosts and, where possible, three failure domains, with three replicas per queue and persistent storage. Check routing and require [publisher confirms](https://www.rabbitmq.com/docs/confirms); quorum queues confirm after majority replication. Test failover with whichever service you choose. Consider [Kafka](https://kafka.apache.org/intro/) when retained event history and independent replay become requirements.

#### 4.1.4 Resilient DB Writer

DB writers own **Transform and Load (T & L)**: validate the message and source contract, normalize fields, units, timestamps, and asset identifiers into the service's expected observation, then write it to the database. Preserve the source identity and observation time through transformation. Invalid or incompatible data follows the [quarantine path](#415-failed-ingest-messages--quarantine), so a successfully received payload cannot silently become incorrect stored data.

Each DB writer retrieves its own **queue-consumer** and **database-writer connection profiles** from the [Managed Secrets Store](#411-managed-secrets-store). The first supplies the queue endpoint and receive/acknowledge authorization; the second supplies the database hostname, port, database name, TLS settings, and login credentials. Give readers publish permissions and writers consume permissions appropriate to the chosen queue. Keep credentials out of code and logs, and follow the database's [service discovery and credential refresh guidance](#421-service-discovery-and-credentials).

The goal is for every queued copy to be processed successfully, even when another writer has already stored the same observation. **Database uniqueness constraints are required** on a stable source-observation key, so the first committed copy wins. Use an atomic [`INSERT … ON CONFLICT … DO NOTHING`](https://www.postgresql.org/docs/current/sql-insert.html) targeted at that key; an expected duplicate is a successful no-op, not a reason to crash or page the owner. Commit the database transaction before acknowledging or deleting the queue message. Back off and slow or pause consumption during database outages; quarantine persistent message-specific failures through the recovery process below. The [database section](#42-database) owns the observation key and its constraints.

#### 4.1.5 Failed Ingest Messages / Quarantine

Messages that cannot be processed after appropriate bounded retries belong in a durable **dead-letter queue (DLQ)** or equivalent quarantine. Preserve the payload, source identity and timestamps, correlation ID, and failure details so the owner can investigate, fix the cause, and deliberately replay the message through normal processing. Logs explain failures; quarantine retains the work. Confirm durable handoff before acknowledging or deleting the original, or use a verified provider-managed redrive mechanism; [dead-lettering guarantees depend on configuration](https://www.rabbitmq.com/docs/dlx). Set [retention](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) long enough for investigation and recovery. Replay must use the same duplicate-safe writer, without endless automatic retry loops.

#### 4.1.6 Monitoring/Alerting

**Managed Secrets Store:** Watch failed or slow secret retrievals, denied or unexpected access, credential expiry where applicable, and failed rotations. Distinguish Managed Secrets Store failures from upstream authentication errors; a healthy reader can still be locked out. Alert on suspected misuse and access failures that threaten ingestion, using audit metadata without logging secret values.

**Reader fleet:** Track healthy versus expected reader counts, defining health by timely collection and progress. For REST readers, track per-reader rolling-hour HTTPS connection counts, request attempts, `200` responses, non-`200` responses grouped by status, and socket failures. For WebSocket readers, watch connection and reconnect counts, socket failures, and time since the last expected event; an open connection alone does not prove progress. Track requests and connections separately because keep-alive can reuse a connection. Alert promptly on missing polls, expected events, or stalled progress; use hourly views to recognize a healthy pattern of life. Compare upstream results with reader progress and independent checks to locate failures, and keep watching response times and source-data age: a `200 OK` can still be stale, and freshness is unknown without reliable source metadata.

**Message queue:** Explicitly monitor and alert on **queue size (number of waiting messages)** and **maximum message time in queue (age of the oldest waiting message)**. Set thresholds for available capacity and the subscribers' freshness budget, so alerts arrive before the queue fills or data becomes too old. Also watch availability, node and replica health, unconfirmed publishes, publish-confirm latency, and publish/consume errors. Alert when a node or replica failure reduces resilience, even if messages are still flowing. A growing backlog can mean writers are blocked or too slow, even when the queue is healthy.

**DB writer:** Watch validation and transformation failures, failed or slow writes, repeated retries, and time since the last successful database update. Count expected duplicates separately from failed writes, and keep freshness tied to source observation time. Fresh queued data with stalled writes points toward the writer or database. Check that fresh, complete data actually reaches subscribers; a successful write alone doesn't prove that. Run monitoring outside the application's host, notify the owner before freshness or coverage becomes unacceptable, and test that alerts arrive.

**Failed ingest messages / DLQ:** **Metrics and alerts are mandatory.** Monitor new failed-message arrivals, queue size, oldest unresolved message age, remaining retention time, failed handoffs, and replay failures. Alert the owner promptly on new quarantined messages and escalate when missing observations threaten coverage or freshness. These represent work the service has not completed; an empty main queue is not proof of health when the DLQ is growing. Verify recovery before resolving the alert.

### 4.2 Database

The database supports both ingestion and subscriber queries. Our example uses [DigitalOcean Managed PostgreSQL](https://docs.digitalocean.com/products/databases/postgresql/) with TimescaleDB enabled, a useful foundation for observations organized around assets and time. Managed hosting lets the provider operate the infrastructure and backup facilities while we focus on schema, access permissions, queries, and retention. We still choose the availability configuration and test recovery; DigitalOcean documents this [shared responsibility](https://docs.digitalocean.com/products/databases/postgresql/concepts/responsibility-model/). Under its [plan rollout](https://docs.digitalocean.com/release-notes/upcoming/dbaas-plan-changes/), eligible existing Standard clusters retain both standby and read-only nodes; new Standard clusters subject to the restrictions support neither. Plan for Advanced when those features are needed, and verify [TimescaleDB extension compatibility](https://docs.digitalocean.com/products/databases/postgresql/details/supported-extensions/) before migration.

Define the stable source-observation key and enforce first-committed-copy-wins here, so independent writers share the same storage contract. For [TimescaleDB hypertables](https://www.tigerdata.com/docs/build/performance-optimization/hypertables-and-unique-indexes), unique keys must include every partitioning column, including time; use a source timestamp that stays the same across repeated deliveries of that observation.

#### 4.2.1 Service Discovery and Credentials

DB writers and the service API retrieve separate named connection profiles from the [Managed Secrets Store](#411-managed-secrets-store), containing the database hostname, port, database name, TLS settings, and their respective login credentials. Use the provider's stable DNS hostname instead of pinning an IP address, so clients can follow [managed failover](https://docs.digitalocean.com/products/databases/postgresql/concepts/advanced-edition/). Give writers and query clients separate accounts with only the permissions they need, and use the provider's [TLS verification settings](https://docs.digitalocean.com/products/databases/postgresql/how-to/connect/). Define how clients refresh settings and replace pooled connections after credential changes; fetching a new secret does not update an existing connection. Clients must also reconnect after interrupted connections during failover.

#### 4.2.2 Primary DB and Failover Standby

**Start with a single Standard instance — RECOMMENDED.** Let ingestion and subscriber queries share it while measured performance meets their targets, with managed backups and tested restores. When the acceptable recovery window calls for it, plan a [failover standby](https://docs.digitalocean.com/products/databases/postgresql/how-to/add-standby-nodes/) ready to take over after primary failure; availability needs can justify this even at low load. Add one to an eligible existing cluster or plan a migration to a compatible HA offering. DigitalOcean describes eligible Standard deployments as offering [failover in minutes](https://docs.digitalocean.com/products/databases/postgresql/how-to/create/); test how long application reads and writes actually take to recover. A standby supplies failover capacity, while a dedicated read replica supplies query capacity.

#### 4.2.3 Read Replicas

On a plan that supports it, add a [read replica](https://docs.digitalocean.com/products/databases/postgresql/how-to/add-read-only-nodes/) when sustained query latency shows subscriber reads competing with ingestion for CPU or I/O, after checking queries, indexes, and connection pooling. This gives subscriber queries their own capacity. If writes are the bottleneck, optimize them or resize the primary; a replica must still replay those writes. Include replication lag in the subscriber freshness budget, and define how the API behaves when replicas are unavailable or too stale.

#### 4.2.4 Backup and Restore

**Backups and tested restores are table stakes with paying customers.** DigitalOcean provides [daily backups and point-in-time recovery](https://docs.digitalocean.com/products/databases/postgresql/details/features/) over a seven-day window. Define acceptable data loss and recovery time, then regularly [restore to a separate cluster](https://docs.digitalocean.com/products/databases/postgresql/how-to/restore-from-backups/), verify TimescaleDB data and representative queries, and rehearse application cutover. Standbys improve availability; backups recover from unwanted changes that replicas also copy. Plan retention to meet business needs: deleting a DigitalOcean cluster also deletes its managed backups.

#### 4.2.5 Monitoring/Alerting

**Discovery and credentials:** Watch secret retrieval, DNS resolution, connection, TLS verification, and login failures. Distinguish a failed lookup or credential rotation from an unhealthy database, and alert when clients cannot establish usable connections.

**Primary and standby:** Combine [DigitalOcean's database metrics](https://docs.digitalocean.com/products/databases/postgresql/how-to/monitor-databases/) with application timings. Watch p95 subscriber-query and write-commit latency, sustained CPU or disk-I/O saturation, memory pressure, connection-pool waits, lock waits, failed transactions, storage headroom, and queue age. Correlate query slowdowns with ingestion activity, and alert on sustained trends approaching latency or freshness limits. Track standby readiness and measured application recovery time after failover; use an independent database-access check alongside provider health signals.

**Read replicas:** Once added, watch replication lag, query latency, and resource headroom on each replica. Verify source-data age and coverage through the subscriber API, so fast queries returning old data are visible. Alert before lag consumes the freshness budget or a replica failure overloads the remaining read capacity.

**Backup and restore:** Track backup failures, the available recovery window, and the date and outcome of the latest restore test. Verify restored data and application access, and compare measured data loss and recovery time with the agreed limits. Alert when required recovery points are missing or restore tests fail.

### 4.3 Resilient Logging Subsystem

[Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) and [DigitalOcean Managed OpenSearch](https://docs.digitalocean.com/products/databases/opensearch/) are managed options for central log storage and search. DigitalOcean documents [forwarding logs](https://docs.digitalocean.com/products/databases/opensearch/how-to/forward-logs/) from resources such as Droplets and managed databases; application logs still need forwarding configured. Apply the [logging conventions](#3-logging) across every component and send them to your chosen destination. Managed hosting reduces operational work, but you still choose retention, access controls, capacity, and availability settings, and verify that every component's logs arrive.

#### 4.3.1 Monitoring/Alerting

Watch delivery delay, rejected or dropped records, collector backlogs, storage growth, and search availability. Send known test log events from each component and verify they arrive and can be found by correlation ID. Alert on missing events or stalled collection through an independent path that still works when the logging service is unavailable.

## License

[MIT](LICENSE) — use, adapt, and share this guide.

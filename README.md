# Web-Based ETL System Design

A practical guide to turning external API data into a service people pay a monthly subscription to use.

## Table of contents

- [1. Overview](#1-overview)
  - [1.1 High-Level Architecture Diagram](#11-high-level-architecture-diagram)
  - [1.2 Service Deployment Diagram](#12-service-deployment-diagram)
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
  - [4.4 Service Health Management](#44-service-health-management)
    - [4.4.1 Monitoring/Alerting](#441-monitoringalerting)
  - [4.5 Monitoring Platform](#45-monitoring-platform)
    - [4.5.1 Monitoring/Alerting](#451-monitoringalerting)
  - [4.6 Customer-Facing Edge](#46-customer-facing-edge)
    - [4.6.1 Managed DNS Service](#461-managed-dns-service)
    - [4.6.2 CDN](#462-cdn)
    - [4.6.3 Origin Server](#463-origin-server)
    - [4.6.4 Monitoring/Alerting](#464-monitoringalerting)
  - [4.7 Customer API](#47-customer-api)
    - [4.7.1 API Load Balancer](#471-api-load-balancer)
    - [4.7.2 API Service](#472-api-service)
    - [4.7.3 In-memory Cache](#473-in-memory-cache)
    - [4.7.4 Monitoring/Alerting](#474-monitoringalerting)
- [License](#license)

## 1. Overview

External data sources provide updates through REST polling, WebSocket push, or other supported methods. Our service collects them, adds value, and makes the results available to paying customers. The ETL pipeline **extracts** (fetches), **transforms** (prepares), and **loads** (stores) the data; customers reach the web service through its DNS and CDN edge.

### 1.1 High-Level Architecture Diagram

<!-- When the diagram changes, update this URL to the commit containing the new SVG to avoid stale cached images. -->
![System overview: external data sources 1 through N each connect to dedicated readers. Readers extract data into the queue; the DB writers validate, transform, and load it. A failed-message quarantine provides controlled replay. The shared Managed Secrets Store sits in the Management Plane with logging, metrics, monitoring, and alerting. The DB writers store observations in the service database. Customers use a DNS lookaside lookup, then HTTPS to the CDN; the CDN fetches landing-page content from the origin. Separate browser API requests go through a load balancer to the API Service, which reads the database, writes account and session changes, and uses an in-memory cache. The Logging Subsystem sits below the database. Shared metrics, monitoring, alerting, and service health management support operations, with the Service Ops Team outside the boundary.](https://raw.githubusercontent.com/SixbucksSolutions/web-based-etl-system-design/1b6c0b41f79c15ebfa9ab07adaddeae9e16337d3/docs/system-overview.svg)

The outer dotted box marks our sphere of control; data providers, customers, and the Service Ops Team are outside it. The lighter inner boxes group the **Data Plane — ingestion and serving** and **Management Plane — shared operational services**. These are logical responsibility groups, not guarantees of separate infrastructure or independent failure. The upper management row contains shared logging, metrics, and secrets; the lower row contains service health management, monitoring, and alerting for operators. These services support the data plane.

Readers perform extraction; the DB writers validate, transform, and load observations. Quarantine retains failed ingest messages for investigation and controlled replay through the queue. Short dotted corner arrows show shared-service interactions: logs and metrics flow inward, while secrets and connection profiles use requests and responses. Service Health Management exchanges control and status with workloads. Customers resolve `foobar.com` through Managed DNS, then send HTTPS requests directly to the CDN. The browser application calls `api.foobar.com` separately through an API load balancer to the API Service. The API Service queries the database and also uses an in-memory cache for frequently requested results and validated session state. The CDN serves the landing page and assets from cache or fetches/revalidates them against the origin; DNS is a lookaside service, not a web-traffic proxy.

### 1.2 Service Deployment Diagram

**Two copies of each application component is a good start.** This example puts multiple instances inside **one DigitalOcean deployment region**. It aims for recovery from supported process or node failures; it does **not** claim the service survives loss of that whole region. Add redundancy incrementally, size the surviving capacity, and test the actual failure modes.

<!-- Update this pinned URL when the deployment diagram changes. -->
![DigitalOcean single-region deployment: regional load balancer, OpenSearch, secrets, Valkey primary and standby, PostgreSQL primary and standby plus two read replicas, and two worker Droplets. A second provider operates a managed queue across three illustrative availability zones inside one region. Node redundancy does not provide region-outage protection.](https://raw.githubusercontent.com/SixbucksSolutions/web-based-etl-system-design/40ec5e6db0486580dc47ce19b0951fe29f13f08b/docs/service-deployment.svg)

**DigitalOcean baseline: 2 Droplets + (12 + 2 × [number of datasources]) self-hosted service deployments + 4 DB instances + 4 managed services.** The managed services are the regional API load balancer, logging, secrets, and Valkey cache; the third-party queue is additional. Six shared self-hosted roles run twice (12 deployments), plus one reader for each datasource on each Droplet. The single “Datasource 1..N Readers” row represents all those reader deployments; combined roles may share a process. Either worker should carry the essential workload alone. Redundant instances still need health checks, routing, replacement, and tested recovery; two VM names do not prove independent physical placement.

**Use DO's actual placement model.** DigitalOcean lists [datacenters and their API/CLI slugs](https://docs.digitalocean.com/platform/regional-availability/), such as `nyc3`; these are not interchangeable with AWS availability zones. The drawing uses `nyc3` as an example deployment region and places both workers and the managed resources there. It makes no claim about separate buildings or hidden provider fault domains.

| Managed component | Documented scope and resilience |
| --- | --- |
| API load balancer | [Regional Load Balancer](https://docs.digitalocean.com/products/networking/load-balancers/details/features/): managed routing to healthy backends within one datacenter region. Global Load Balancers are a different option for cross-region routing. |
| Logging | [Managed OpenSearch](https://docs.digitalocean.com/products/databases/opensearch/how-to/create/): choose one datacenter; use at least three nodes for automated node failover. This does not establish cross-region log availability. |
| Secrets | [Secrets Manager](https://docs.digitalocean.com/reference/doctl/reference/secrets/): regional secret containers. Do not assume automatic cross-region replication. |
| In-memory cache | [Managed Valkey](https://docs.digitalocean.com/products/databases/valkey/details/limits/): primary and selected standby remain in the same region. This provides node redundancy, not region-outage protection. |

**PostgreSQL has the same distinction.** Its [standby must remain in the primary's region](https://docs.digitalocean.com/products/databases/postgresql/details/limits/). The illustrated primary, standby, and two read replicas are all regional resources on a plan that supports them. That is useful node HA, but a whole-region outage can make all four unavailable and stop reads and writes. Verify the edition and [extension support](https://docs.digitalocean.com/products/databases/postgresql/details/supported-extensions/): TimescaleDB is listed for Standard Edition, not Advanced Edition; restricted new Standard plans cannot add standby/read-only nodes. The illustrated combination therefore requires an eligible existing configuration or another verified database option.

**Regional disaster recovery is separate work.** DO supports [read replicas in other regions and deliberate promotion into a new independent cluster](https://docs.digitalocean.com/products/databases/postgresql/how-to/add-read-only-nodes/). That is not automatic cross-region standby failover: plan application cutover, acceptable replication lag/data loss, and reconciliation with the former primary. [VPC peering](https://docs.digitalocean.com/products/networking/vpc/how-to/create-peering/) or public TLS connectivity can connect regions, but networking alone does not add database failover. Region-loss testing must also account for logging, secrets, and cache dependencies.

**The queue is operated by a third party.** The lower **CSP 2** boundary contains **one region with three availability zones**, illustrating provider-operated multi-AZ replication. This protects against an AZ failure within that region; it is not a claim of region-loss protection. The concrete example is [Amazon SQS standard queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html). The AWS, Azure, and Google names are alternatives, not three deployments; verify the selected product's guarantees. Readers and writers still need TLS connectivity, scoped authorization, retries, queue/DLQ monitoring, and correct acknowledgement handling. The external queue may remain available during a DO outage, but cannot make unavailable readers or database writers run.

**Budget conservatively for self-hosted metrics.** DO's [Monitoring agent collects system-level metrics](https://docs.digitalocean.com/products/monitoring/details/limits/); its newer [Insights public preview is explicitly unsuitable as the primary production observability system](https://docs.digitalocean.com/products/insights/details/limits/). That does not establish a production home for our custom ingest and data-freshness metrics. In the diagram, independent metrics replicas collect the same targets and retain their own persistent data. For example, [Prometheus local storage is not a replicated cluster](https://prometheus.io/docs/prometheus/latest/storage/): plan query failover, retention, and alert deduplication; use an additional backend if you need a durable, unified history. Logical rows can share a process—Prometheus can provide collection, storage, and rule evaluation.

**Hosted alternatives:** An external managed metrics service is **RECOMMENDED** when it meets the application's needs and budget, reducing the infrastructure this small team must operate. These non-DO examples show that the self-hosted footprint is a planning assumption, not an obligation. They are alternatives to evaluate, not a requirement to migrate everything to one vendor.

| Component | Hosted non-DO example |
| --- | --- |
| Reader fleets and DB writers | [Amazon ECS on Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html) runs your containerized workers; you still own extraction and transformation logic. |
| Message queue and DLQ | [Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html), or [Amazon MQ](https://docs.aws.amazon.com/amazon-mq/latest/developer-guide/welcome.html) for managed RabbitMQ. |
| PostgreSQL + TimescaleDB | [Tiger Cloud](https://www.tigerdata.com/docs), the managed TimescaleDB service. Verify the required HA plan and extensions. |
| Secrets and connection profiles | [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html). |
| Central logging | [Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) or [Grafana Cloud](https://grafana.com/docs/grafana/latest/introduction/grafana-cloud/). |
| Metrics Store and Monitoring Platform | [Grafana Cloud](https://grafana.com/docs/learning-paths/grafana-cloud-tour/explore-send-data/) supplies managed metrics storage; its dashboards and alerting cover additional monitoring roles. Custom data-gap detection and durable recovery requests still need application logic. |
| Service Health Management | [Amazon ECS services](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html) maintain desired worker counts and replace unhealthy tasks; you configure health checks and scaling policies. |
| Service Ops Team Alerting System | [PagerDuty](https://docs.pagerduty.com/incident-management/services/services-and-integrations) receives monitoring events and manages incident notification. |
| Managed DNS | [Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html). |
| CDN | [Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html). |
| Static origin | [Amazon S3](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) can hold the static assets behind CloudFront. |
| In-memory cache | [Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.html) offers managed Valkey and memcached; check connectivity and the chosen availability configuration. |
| API load balancer | [AWS Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html). |

Managed hosting shifts infrastructure work; it does not remove application ownership. Check network reachability, identity, data residency, retention, failure coverage, and transfer costs before combining providers. The API Service remains application-owned in this design.


## 2. Observability/Monitoring/Alerting

Design [monitoring and alerting](https://sre.google/sre-book/monitoring-distributed-systems/) from the start. The owner should learn about problems **no later than customers do**. At every layer, ask what can break, how we'll detect it, what the system should do, and when to notify the owner. A dashboard alone won't wake anyone up. Every component in section 4 closes with **Monitoring/Alerting**, covering what to watch and when to act as we design that component.

Metrics from every component reach a shared **Metrics Store**, whether pushed or collected by a scraper. The **[Monitoring Platform](#45-monitoring-platform)** queries those measurements for dashboards and alert conditions; **Service Health Management** uses native health checks and monitoring signals to right-size workloads and replace unhealthy instances; the **Service Ops Team Alerting System** groups and routes notifications to the **Service Ops Team**, who use Monitoring to investigate. These are [logical responsibilities](https://prometheus.io/docs/alerting/latest/overview/); one managed service may provide several of them. The diagram connects Alerting to Monitoring, with Monitoring accessing the Metrics Store.

Monitor external datasource data for both **freshness** (age of the newest source-timestamped observation committed to the database) and **coverage** (unacceptable gaps between observations). Evaluate each source/data stream against its expected cadence and active periods, allowing for publication delay and late or out-of-order arrivals. Configure the thresholds separately: one-minute updates do not necessarily imply a strict one-minute freshness limit. New arrivals may clear a stale-data alert while an earlier gap remains unresolved.

**Silence can be its own problem.** Send recurring test alerts through production detection, routing, and delivery; record arrival at the destination before filtering them out of routine human notifications. If that heartbeat stops, an independent watchdog must raise its **own alert**, potentially **out of band (OOB)** through a channel that doesn't depend on the primary alerting system. Occasionally test actual owner notifications too.

## 3. Logging

**Every component sends logs to exactly one shared, searchable destination: the Resilient Logging Subsystem.** **Prefer a managed logging service over self-hosting — RECOMMENDED**, to reduce infrastructure and maintenance work; [section 4.3](#43-resilient-logging-subsystem) gives examples. One logical destination can have redundant collectors and storage; it must not depend on one host. Correlated logs are essential for diagnosing failures across a distributed system. Use [structured fields](https://opentelemetry.io/docs/specs/otel/logs/data-model/): UTC timestamp, component, instance, severity, and an execution ID (for example, a GUID). Carry a shared correlation or trace ID across requests and queue messages, alongside each execution's own ID, so one search follows the work across readers, writers, and API calls. Nobody should have to guess where a component put its logs.

**Everything must log enough, at the right level, to reconstruct what happened.** Record meaningful starts, outcomes, durations, retries, and failures. Use consistent [levels](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging/overview#log-level): **Trace/Debug** for detailed diagnostics, **Information** for normal milestones, **Warning** for recoverable degradation, **Error** for failed operations, and **Critical** when a component cannot function. Filtering by component, execution, and severity should make an incident understandable without drowning it in noise. Expected duplicate writes are normal outcomes, not errors. Keep secrets and sensitive payloads out. Buffer and retry log delivery within limits; monitor dropped logs and missing test events, with independent alerting if the logging subsystem fails.

## 4. System Components

The diagram groups components by their primary responsibility. The **data plane** processes and serves customer data, including quarantined work awaiting recovery. The **management plane** supplies shared configuration, credentials, and operational visibility. These groups still depend on one another; the grouping itself does not provide deployment, security, or failure isolation.

### 4.1 Data Ingest

Move source updates through readers and a message queue into the database.

#### 4.1.1 Managed Secrets Store

A prototype may keep credentials in code or plaintext `.env`/deployment configs. A stolen key or bearer token may impersonate the customer within its permissions. Use an access-controlled [Managed Secrets Store](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html), limit reader access, and plan provider-supported rotation and revocation. **Startup-injected environment variables normally require reader restarts to pick up credential changes; fetching the current secret from the Managed Secrets Store for each collection cycle can pick up rotations without restarting.** Any caching needs an explicit refresh policy. A managed service can improve availability and reduce maintenance; DigitalOcean documents a [Secrets Manager CLI](https://docs.digitalocean.com/reference/doctl/reference/secrets/) and [API](https://docs.digitalocean.com/reference/api/reference/security/) for readers on Droplets. Protect credentials used to access the Managed Secrets Store, check regional resilience, and keep secrets out of code, images, browser bundles, and logs.

Named **connection profiles** can bundle a service's hostname or URL, port, TLS settings, and credentials in this same store; [AWS uses this pattern for database secrets](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_secret_json_structure.html). A separate configuration service is not required for these settings. Deployment supplies the store location, profile names, and a securely provisioned identity for accessing them. Limit access per workload and prefer provider-supported workload identity where available. Use bounded caching with an explicit refresh policy; stable DNS names and managed endpoints handle changing service nodes.

#### 4.1.2 Resilient Reader Fleet

Readers own **Extract (E)**: collect source payloads and publish them with source identity, source-effective time, receive time, and a correlation ID. For time-series observations, require an explicit effective instant or bounded interval from the source; missing or ambiguous timing is a source-contract/data-quality error, and polling time is not a substitute. The aim is to survive failures without customers noticing an interruption in data flow. Each data source has its own fleet; each reader independently collects updates using that source's supported protocol and publishes its copy to the queue. Spread readers across [independent failure domains](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/availability-zones.html)—preferably separate availability zones (AZs), or separate regions when regional outages are in scope. Healthy readers can keep collecting while a failed host or network link recovers. Budget redundant calls against the provider's shared rate limits, and handle duplicate or out-of-order updates safely downstream.

Each reader retrieves its named **queue-publisher connection profile** from the [Managed Secrets Store](#411-managed-secrets-store). For RabbitMQ, this includes the broker's stable hostname, port, TLS settings, and publisher credentials; for SQS, it includes the queue URL and region, with authorization through the workload's AWS identity. Readers publish through that endpoint without tracking individual queue nodes. Refresh cached settings and reconnect when needed.

#### 4.1.3 Resilient Message Queue

The queue gives us two key advantages:

- **Loose coupling:** Readers publish messages; DB writers consume them. That simple transfer contract lets either component be changed or replaced independently, provided the message format and delivery expectations stay compatible.
- **Backpressure:** When the database is heavily loaded, writers can slow consumption and keep in-flight work bounded. The [queue absorbs the backlog](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) while readers keep collecting and publishing; readers do not need to handle database write timeouts. This buffer has limits: monitor queue capacity and message age so a prolonged slowdown does not exhaust storage or exceed the freshness budget.

Message queuing is a well-solved problem. **Prefer a managed queue service over self-hosting — RECOMMENDED**, so the provider handles more of the infrastructure, maintenance, and availability work. [Amazon SQS](https://aws.amazon.com/sqs/) and hosted RabbitMQ are examples; choose a service that fits your environment and delivery requirements. The aim is for every reader's copy to reach a DB writer, with redundant copies protecting an observation if one is lost before queue acceptance. Check successful enqueueing, retry uncertain sends, and expect duplicate deliveries. For successful processing, writers acknowledge or delete messages only after database commit; failed work follows the [quarantine path](#415-failed-ingest-messages--quarantine).

**Set main ingest-queue retention to one hour.** This deliberately assumes each datasource can backfill the required missing observations—for example, by `(ticker, source timestamp)`—after a writer outage or backlog. Verify that the available history and request budget cover the recovery window, including any required revisions. [Expired SQS messages are deleted](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html); they are not automatically moved to the DLQ. Monitoring must detect missing coverage and, where revisions matter, reconcile source revisions, retaining recovery requests so readers can fetch the missing data and retry ingestion once processing recovers. If the source cannot provide that history, this policy can permanently lose data and needs longer retention or a different recovery plan. Retention limits age; it does not provide newest-first processing. The DLQ has its own, longer investigation/recovery retention.

The [deployment diagram](#12-service-deployment-diagram) uses a third-party managed queue, allowing two application worker VMs without operating a RabbitMQ cluster. [RabbitMQ](https://www.rabbitmq.com/docs/download) with [quorum queues](https://www.rabbitmq.com/docs/quorum-queues) remains an alternative. If self-hosting for single-zone-loss resilience, place three queue replicas on three hosts in three independent failure domains, with persistent storage; two replicas cannot lose either member and retain a majority. Check routing and require [publisher confirms](https://www.rabbitmq.com/docs/confirms). Test failover with whichever service you choose. Consider [Kafka](https://kafka.apache.org/intro/) when retained event history and independent replay become requirements.

#### 4.1.4 Resilient DB Writer

DB writers own **Transform and Load (T & L)**: validate the message and source contract, normalize fields, units, timestamps, and asset identifiers into the service's expected observation, then write it to the database. Preserve the source identity and observation time through transformation. Invalid or incompatible data follows the [quarantine path](#415-failed-ingest-messages--quarantine), so a successfully received payload cannot silently become incorrect stored data.

**All DB writer instances write to the current primary (master)** through its stable write endpoint; replicas and the failover standby are not write targets. Each DB writer retrieves its own **queue-consumer** and **database-writer connection profiles** from the [Managed Secrets Store](#411-managed-secrets-store). The first supplies the queue endpoint and receive/acknowledge authorization; the second supplies the database hostname, port, database name, TLS settings, and login credentials. Give readers publish permissions and writers consume permissions appropriate to the chosen queue. Keep credentials out of code and logs, and follow the database's [service discovery and credential refresh guidance](#421-service-discovery-and-credentials).

The goal is for every queued copy to be processed successfully, even when another writer has already stored the same observation revision. **Database uniqueness constraints are required** on a stable source-observation-and-revision key, so the first committed copy of that revision wins. Use an atomic [`INSERT … ON CONFLICT … DO NOTHING`](https://www.postgresql.org/docs/current/sql-insert.html) targeted at that key; an expected duplicate is a successful no-op, not a reason to crash or page the owner. Commit the database transaction before acknowledging or deleting the queue message. Back off and slow or pause consumption during database outages; quarantine persistent message-specific failures through the recovery process below. The [database section](#42-database) owns the observation key and its constraints.

#### 4.1.5 Failed Ingest Messages / Quarantine

Messages that cannot be processed after appropriate bounded retries belong in a durable **dead-letter queue (DLQ)** or equivalent quarantine. Preserve the payload, source identity and timestamps, correlation ID, and failure details so the owner can investigate, fix the cause, and deliberately replay the message through normal processing. Logs explain failures; quarantine retains the work. Confirm durable handoff before acknowledging or deleting the original, or use a verified provider-managed redrive mechanism; [dead-lettering guarantees depend on configuration](https://www.rabbitmq.com/docs/dlx). Set [retention](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) long enough for investigation and recovery. Replay must use the same duplicate-safe writer, without endless automatic retry loops.

#### 4.1.6 Monitoring/Alerting

**Managed Secrets Store:** Watch failed or slow secret retrievals, denied or unexpected access, credential expiry where applicable, and failed rotations. Distinguish Managed Secrets Store failures from upstream authentication errors; a healthy reader can still be locked out. Alert on suspected misuse and access failures that threaten ingestion, using audit metadata without logging secret values.

**Reader fleet:** Track healthy versus expected reader counts, defining health by timely collection and progress. For REST readers, track per-reader rolling-hour HTTPS connection counts, request attempts, `200` responses, non-`200` responses grouped by status, and socket failures. For WebSocket readers, watch connection and reconnect counts, socket failures, and time since the last expected event; an open connection alone does not prove progress. Track requests and connections separately because keep-alive can reuse a connection. Alert promptly on missing polls, expected events, or stalled progress; use hourly views to recognize a healthy pattern of life. Compare upstream results with reader progress and independent checks to locate failures, and keep watching response times and source-data age: a `200 OK` can still be stale, and freshness is unknown without reliable source metadata.

**Message queue:** Explicitly monitor and alert on **queue size (number of waiting messages)** and **maximum message time in queue (age of the oldest waiting message)**. Set thresholds for available capacity and the customers' freshness budget, so alerts arrive before the queue fills or data becomes too old. Alert well before the one-hour retention deadline, allowing time to restore processing and drain the backlog. Also watch availability, node and replica health, unconfirmed publishes, publish-confirm latency, and publish/consume errors. Alert when a node or replica failure reduces resilience, even if messages are still flowing. A growing backlog can mean writers are blocked or too slow, even when the queue is healthy.

**DB writer:** Watch validation and transformation failures, failed or slow writes, repeated retries, and time since the last successful database update. Count expected duplicates separately from failed writes, and keep freshness tied to source observation time. Fresh queued data with stalled writes points toward the writer or database. Check that fresh, complete data actually reaches customers; a successful write alone doesn't prove that. Run monitoring outside the application's host, notify the owner before freshness or coverage becomes unacceptable, and test that alerts arrive.

**Failed ingest messages / DLQ:** **Metrics and alerts are mandatory.** Monitor new failed-message arrivals, queue size, oldest unresolved message age, remaining retention time, failed handoffs, and replay failures. Alert the owner promptly on new quarantined messages and escalate when missing observations threaten coverage or freshness. These represent work the service has not completed; an empty main queue is not proof of health when the DLQ is growing. Verify recovery before resolving the alert.

### 4.2 Database

The database stores both ingested observations and the service's user-account/session records, supporting ingestion and customer queries. Our example uses [DigitalOcean Managed PostgreSQL](https://docs.digitalocean.com/products/databases/postgresql/) with TimescaleDB enabled, a useful foundation for observations organized around assets and time. Managed hosting lets the provider operate the infrastructure and backup facilities while we focus on schema, access permissions, queries, and retention. We still choose the availability configuration and test recovery; DigitalOcean documents this [shared responsibility](https://docs.digitalocean.com/products/databases/postgresql/concepts/responsibility-model/). Under its [plan rollout](https://docs.digitalocean.com/release-notes/upcoming/dbaas-plan-changes/), eligible existing Standard clusters retain both standby and read-only nodes; new Standard clusters subject to the restrictions support neither. Plan for Advanced when those features are needed, and verify [TimescaleDB extension compatibility](https://docs.digitalocean.com/products/databases/postgresql/details/supported-extensions/) before migration.

**Retain every received observation revision as append-only history**, including source-issued corrections or cancellations. Keep the source-effective instant or interval, source revision/version (and revision time when provided), and our receive time distinct. Effective time says when a value applies; revision metadata identifies which published version we received. A new revision adds a row; a repeated delivery of the same revision is a successful no-op. The current-value view follows the source's revision ordering, not arrival order. Preserving received revisions does not prove that the source delivered its entire revision history.

Define the stable observation-and-revision key and enforce first-committed-copy-wins per revision. For [TimescaleDB hypertables](https://www.tigerdata.com/docs/build/performance-optimization/hypertables-and-unique-indexes), unique keys must include every partitioning column, including time; use a source timestamp that stays the same across repeated deliveries. If the source provides no revision identifier or ordering, define that policy explicitly; a local fetch timestamp alone must not turn each duplicate into a new revision.

#### 4.2.1 Service Discovery and Credentials

DB writers retrieve a **database-writer profile** targeting the primary write endpoint from the [Managed Secrets Store](#411-managed-secrets-store); every API instance retrieves a **replica-read profile** with the configured read-replica endpoints. Profiles contain hostnames, ports, database name, TLS settings, and scoped credentials. Use stable DNS names instead of fixed IP addresses, so clients can follow [managed failover](https://docs.digitalocean.com/products/databases/postgresql/concepts/advanced-edition/). Give the replica pool a read-only database account; API operations that must write or read authoritative state use a separate, narrowly permitted primary profile. Use the provider's [TLS verification settings](https://docs.digitalocean.com/products/databases/postgresql/how-to/connect/). Define refresh and pool replacement after credential changes, and reconnect after interrupted connections; new settings do not update existing connections automatically.

#### 4.2.2 Primary DB and Failover Standby

**Start with a single Standard instance — RECOMMENDED.** Let ingestion and customer queries share it while measured performance meets their targets, with managed backups and tested restores. When the acceptable recovery window calls for it, plan a [failover standby](https://docs.digitalocean.com/products/databases/postgresql/how-to/add-standby-nodes/) ready to take over after primary failure; availability needs can justify this even at low load. Add one to an eligible existing cluster or plan a migration to a compatible HA offering. DigitalOcean describes eligible Standard deployments as offering [failover in minutes](https://docs.digitalocean.com/products/databases/postgresql/how-to/create/); test how long application reads and writes actually take to recover. A standby supplies failover capacity, while a dedicated read replica supplies query capacity. DO standbys remain in the primary's region, so describe this as node HA rather than region-outage protection. Cross-region recovery requires a separate [disaster-recovery plan](#12-service-deployment-diagram).

#### 4.2.3 Read Replicas

**In the replicated deployment, all ingest writers write to the primary, and every API instance sends customer-data reads to the read replicas.** Route queries across healthy, sufficiently current [read replicas](https://docs.digitalocean.com/products/databases/postgresql/how-to/add-read-only-nodes/) using bounded, separate connection pools. Customer queries get their own connection budgets instead of competing with ingestion for the primary's client slots. Separating the workloads also lets read nodes devote more CPU, PostgreSQL buffer cache, and OS page cache to the query working set, while the primary concentrates on ingestion. **Scale the primary for write demand; scale individual replicas or add replicas for read demand.** This separates read and write capacity planning, within provider limits: DigitalOcean requires a read-only node to be at least as large as the primary. Adding read replicas does not increase primary write capacity, and replicas still spend CPU and I/O replaying writes. Pool limits across all API instances must fit each replica's capacity.

The initial single-instance setup shares one endpoint until replicas are added. Once split, [replication lag](https://www.postgresql.org/docs/current/hot-standby.html) is part of the freshness budget: avoid routing to stale or unhealthy replicas, and define degraded behavior rather than silently flooding the primary. Operations requiring immediate read-after-write consistency or current authentication/revocation state need an explicit authoritative path; API writes—including registration, account updates, and session changes—also go to the primary through their separate restricted profile. A read replica is query capacity, not the primary's automatic failover standby.

#### 4.2.4 Backup and Restore

**Backups and tested restores are table stakes with paying customers.** DigitalOcean provides [daily backups and point-in-time recovery](https://docs.digitalocean.com/products/databases/postgresql/details/features/) over a seven-day window. Define acceptable data loss and recovery time, then regularly [restore to a separate cluster](https://docs.digitalocean.com/products/databases/postgresql/how-to/restore-from-backups/), verify TimescaleDB data and representative queries, and rehearse application cutover. Standbys improve availability; backups recover from unwanted changes that replicas also copy. Plan retention to meet business needs: deleting a DigitalOcean cluster also deletes its managed backups.

#### 4.2.5 Monitoring/Alerting

**Discovery and credentials:** Watch secret retrieval, DNS resolution, connection, TLS verification, and login failures. Distinguish a failed lookup or credential rotation from an unhealthy database, and alert when clients cannot establish usable connections.

**Primary and standby:** Combine [DigitalOcean's database metrics](https://docs.digitalocean.com/products/databases/postgresql/how-to/monitor-databases/) with application timings. Watch p95 customer-query and write-commit latency, sustained CPU or disk-I/O saturation, memory pressure, connection-pool waits, lock waits, failed transactions, storage headroom, and queue age. Correlate query slowdowns with ingestion activity, and alert on sustained trends approaching latency or freshness limits. Track standby readiness and measured application recovery time after failover; use an independent database-access check alongside provider health signals.

**Read replicas:** Once added, watch replication lag, query latency, and resource headroom on each replica. Verify source-data age and coverage through the customer API, so fast queries returning old data are visible. Watch per-replica connection usage, pool waits, and uneven query distribution. Alert before lag consumes the freshness budget or a replica failure overloads the remaining read capacity.

**Backup and restore:** Track backup failures, the available recovery window, and the date and outcome of the latest restore test. Verify restored data and application access, and compare measured data loss and recovery time with the agreed limits. Alert when required recovery points are missing or restore tests fail.

### 4.3 Resilient Logging Subsystem

[Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) and [DigitalOcean Managed OpenSearch](https://docs.digitalocean.com/products/databases/opensearch/) are managed options for central log storage and search. DigitalOcean documents [forwarding logs](https://docs.digitalocean.com/products/databases/opensearch/how-to/forward-logs/) from resources such as Droplets and managed databases; application logs still need forwarding configured. Apply the [logging conventions](#3-logging) across every component and send them to your chosen destination. Managed hosting reduces operational work, but you still choose retention, access controls, capacity, and availability settings, and verify that every component's logs arrive.

#### 4.3.1 Monitoring/Alerting

Watch delivery delay, rejected or dropped records, collector backlogs, storage growth, and search availability. Send known test log events from each component and verify they arrive and can be found by correlation ID. Alert on missing events or stalled collection through an independent path that still works when the logging service is unavailable.

### 4.4 Service Health Management

**Right-size service components and replace unhealthy instances.** Use the hosting platform or a suitable supervisor to maintain the intended healthy capacity, restart or replace failed workers, and confirm replacements are ready.

Keep health assessment separate from recovery decisions, even when one platform implements both. [Liveness checks](https://kubernetes.io/docs/concepts/workloads/pods/probes/) identify stuck or failed processes; readiness checks decide whether an instance should accept work. A readiness failure alone is not a restart instruction. Native checks drive local recovery; monitoring adds fleet-wide capacity and performance signals. Bound scaling and restart rates, and distinguish a failed worker from an unavailable dependency so an outage does not trigger a replacement storm. Keep policies versioned, and give Ops controlled, audited ways to pause automation or recover service. This is an operational capability in our Management Plane; its automatic [desired-state reconciliation](https://kubernetes.io/docs/concepts/architecture/controller/) is a control-plane function, not a requirement to build a custom service or use Kubernetes.

#### 4.4.1 Monitoring/Alerting

Monitor desired versus healthy instance counts, time to restore healthy capacity, failed replacements, repeated restarts, and blocked scaling actions. Log recovery decisions and alert when automation cannot restore service or repeatedly makes health worse. Test recovery and the ability to stop unsafe automation; keep an independent alert path for failures of the health-management mechanism itself.

### 4.5 Monitoring Platform

When Monitoring detects **stale or missing external datasource data**, record the affected source, data stream, and time interval, notify Ops through the Service Ops Team Alerting System, and request recovery where supported. Recovery requests are **durable, tracked work**, independent of logging and notifications. Source-specific readers perform the fetches with bounded retries and backoff, publishing recovered observations through the normal ingest pipeline. Keep the gap unresolved until database coverage is verified; escalate failed or impossible recovery rather than silently abandoning it.

#### 4.5.1 Monitoring/Alerting

Watch failed rule evaluations, missing or stale telemetry, failed condition/recovery-request publication, and the age of unresolved conditions. Test that a known data-health condition produces both an Ops notification and a durable recovery request where applicable. Monitor this path independently of the Monitoring Platform itself.

### 4.6 Customer-Facing Edge

The landing page at `foobar.com` uses three distinct responsibilities: name resolution, CDN delivery, and origin hosting.

#### 4.6.1 Managed DNS Service

Managed authoritative DNS publishes the records customers use to find the CDN entry point for `foobar.com`. The customer's resolver may answer from cache; otherwise it performs a [DNS lookup](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/welcome-dns-service.html). This is a lookaside exchange of names and addresses. The HTTP request itself goes to the CDN, not through the DNS service.

#### 4.6.2 CDN

Customers connect over HTTPS to the CDN for the landing page and its assets. The CDN [serves cached content or contacts the origin](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html) when content is missing or needs revalidation. Keep DNS and CDN responsibilities distinct even when one provider supplies both.

#### 4.6.3 Origin Server

The origin hosts the landing page and assets behind the CDN. It must remain available for cache misses and revalidation; a warm cache can temporarily hide an origin failure. Application API requests use the separate [Customer API](#47-customer-api) path.

#### 4.6.4 Monitoring/Alerting

Check DNS resolution and the landing page's HTTPS availability from outside the service. Watch certificate expiry, CDN and origin error rates and latency, and cache behavior. Test an uncached origin fetch as well as cached delivery, so the CDN does not conceal a broken origin until customers miss the cache.

### 4.7 Customer API

After loading the site, the browser application sends HTTPS requests to `api.foobar.com`. The API endpoint has its own load balancer and server fleet; serving the landing page does not grant access to protected operations.

#### 4.7.1 API Load Balancer

**Prefer a managed load balancer — RECOMMENDED**, such as [DigitalOcean Load Balancers](https://docs.digitalocean.com/products/networking/load-balancers/). It provides one API entry point and routes requests to healthy, ready server instances. Configure TLS and health checks deliberately; the load balancer's routing decision does not replace application authentication or authorization.

#### 4.7.2 API Service

The API Service can run multiple instances across independent failure domains as availability needs grow. In this design, the API application owns authentication and checks customer identity and subscription entitlements before protected operations. The service database also holds user accounts and session records: registration, account updates, and session changes write to the primary through a separate restricted API profile. Use established [authentication/session libraries](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html), and make session validation work across the fleet: a valid login must not depend on returning to one particular server. Every API instance uses the [read-replica pool for customer-data queries](#423-read-replicas), with the explicit authoritative-state exceptions described there. Instances retrieve required connection settings and secrets through their scoped profiles.

**Cache validated bearer-session lookups to reduce per-request database reads.** Use a non-reversible token fingerprint (for example, a server-keyed HMAC) as the cache key rather than the raw bearer token; store the user ID, permissions, required session variables, and validated absolute expiry. Set the [TTL no later than token/session expiry](https://www.rfc-editor.org/rfc/rfc7662#section-4), capped by the acceptable revalidation interval, and check expiry on every request without extending it on cache hits. On a miss, validate against authoritative state before populating the cache. Logout, revocation, and entitlement changes must invalidate or version cached state across the fleet; expiry alone is insufficient. For signed JWTs, [verify the signature, trusted issuer, intended audience, and time claims locally](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html#jwt)—those checks do not inherently require a database query. Cache failures must fall back to authoritative validation or deny access, never grant it. Restrict cache access and keep bearer tokens out of logs.

#### 4.7.3 In-memory Cache

Cache frequently requested results to reduce database work and response time. [Memcached](https://memcached.org/) is a focused in-memory key/value cache; [Valkey](https://valkey.io/docs/topics/introduction/) adds richer data structures and replication. Treat cached results as rebuildable: the database remains authoritative. Set expiry and invalidation rules to match the data-freshness budget, and scope keys to the customer or permissions when results differ by access. Authentication and authorization still apply on cache hits. Use bounded cache timeouts and controlled fallback to the database; avoid letting a cold or failed cache trigger an uncontrolled burst of identical queries.

**Prefer a managed cache when it meets your needs — RECOMMENDED.** [DigitalOcean Managed Valkey](https://docs.digitalocean.com/products/databases/valkey/) is an option within the existing provider; self-hosting is not required. Check [plan and availability limits](https://docs.digitalocean.com/products/databases/valkey/details/limits/): standby nodes must be in the same region, and the smallest plan cannot add one. A managed cache still needs the same expiry, fallback, and monitoring decisions.

#### 4.7.4 Monitoring/Alerting

Watch healthy versus expected API instances, request latency, error rates, and authentication failures. Use external checks for both successful authorized access and correctly denied access, and distinguish ordinary invalid logins from authentication-service failures. Alert on lost redundancy, broken login/session behavior, and customer-facing failures before they become widespread. For the cache, watch hit/miss rates, latency, timeouts, memory pressure, evictions, and the age of returned data. Test session expiry and revocation on cache hits as well as misses, and alert on failed invalidation. Test a cold or unavailable cache and monitor the resulting database load.

## License

[MIT](LICENSE) — use, adapt, and share this guide.

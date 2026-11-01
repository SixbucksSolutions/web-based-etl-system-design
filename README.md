# Web-Based ETL System Design

A practical guide to turning external API data into a service people pay a monthly subscription to use.

## Table of contents

- [1. Quick Problem Description](#1-quick-problem-description)
- [2. Observability/Monitoring/Alerting](#2-observabilitymonitoringalerting)
- [3. System Components](#3-system-components)
  - [3.1 Data Ingest](#31-data-ingest)
    - [3.1.1 External Credentials Vault/Storage](#311-external-credentials-vaultstorage)
    - [3.1.2 Resilient Reader Fleet](#312-resilient-reader-fleet)
    - [3.1.3 Resilient Message Queue](#313-resilient-message-queue)
    - [3.1.4 Resilient DB Writer](#314-resilient-db-writer)
    - [3.1.5 Monitoring/Alerting](#315-monitoringalerting)
  - [3.2 Database](#32-database)
    - [3.2.1 Monitoring/Alerting](#321-monitoringalerting)
- [License](#license)

## 1. Quick Problem Description

An authenticated REST API publishes data once per minute. Our service fetches it, adds value, and makes it available to paying subscribers. The ETL pipeline **extracts** (fetches), **transforms** (prepares), and **loads** (stores) the data; subscriber requests use our API to access the results.

```mermaid
flowchart TB
    upstream["Authenticated REST API<br/>Updates once per minute"]
    users["Subscribers<br/>Paid monthly access"]

    subgraph service["Web service: our sphere of control"]
        direction TB
        extract["Extract<br/>Fetch data from the external API"]
        queries["Web queries<br/>Receive subscriber requests"]

        transform["Transform<br/>Validate and prepare the data"]
        api["Service API<br/>Web service entry point"]

        load["Load<br/>Write the prepared data"]
        middleware["Middleware<br/>Request processing"]
        db[("Service database")]

        extract --> transform
        transform --> load
        load --> db
        api --> middleware
        middleware --> db
    end

    upstream --> extract
    users --> queries
    queries --> api

    style service fill:transparent,stroke:#64748b,stroke-width:2px,stroke-dasharray:2 5,stroke-linecap:round
```

The dotted box marks our sphere of control; the provider and subscribers are outside it. Arrows show the data and request paths. The API and middleware boxes show responsibilities; middleware typically runs before an endpoint handler and may also process its response.

## 2. Observability/Monitoring/Alerting

Design [monitoring and alerting](https://sre.google/sre-book/monitoring-distributed-systems/) from the start. The owner should learn about problems **no later than customers do**. At every layer, ask what can break, how we'll detect it, what the system should do, and when to notify the owner. A dashboard alone won't wake anyone up. Every component in section 3 closes with **Monitoring/Alerting**, covering what to watch and when to act as we design that component.

**Silence can be its own problem.** Send recurring test alerts through production detection, routing, and delivery; record arrival at the destination before filtering them out of routine human notifications. If that heartbeat stops, an independent watchdog must raise its **own alert**, potentially **out of band (OOB)** through a channel that doesn't depend on the primary alerting system. Occasionally test actual owner notifications too.

## 3. System Components

### 3.1 Data Ingest

Move source updates through readers and a message queue into the database.

#### 3.1.1 External Credentials Vault/Storage

A prototype may keep credentials in code or plaintext `.env`/deployment configs. A stolen key or bearer token may impersonate the customer within its permissions. Use an access-controlled [secrets store](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html), limit reader access, and plan provider-supported rotation and revocation. **Startup-injected environment variables normally require reader restarts to pick up credential changes; fetching the current secret from the store for each collection cycle can pick up rotations without restarting.** Any caching needs an explicit refresh policy. A hosted service can improve availability and reduce vault maintenance; DigitalOcean documents a [Secrets Manager CLI](https://docs.digitalocean.com/reference/doctl/reference/secrets/) and [API](https://docs.digitalocean.com/reference/api/reference/security/) for readers on Droplets. Protect store-access credentials, check regional resilience, and keep secrets out of code, images, browser bundles, and logs.

#### 3.1.2 Resilient Reader Fleet

The aim is to design a system that can survive failures without customers ever noticing an interruption in data flow. Each reader independently fetches and publishes its copy of the expected source update to the queue. Spread readers across [independent failure domains](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/availability-zones.html)—preferably separate availability zones (AZs), or separate regions when regional outages are in scope. Healthy readers can keep collecting while a failed host or network link recovers. Budget redundant calls against the provider's shared rate limits, and handle duplicate or out-of-order updates safely downstream.

#### 3.1.3 Resilient Message Queue

The aim is for every reader's copy to reach a DB writer, with redundant copies protecting an observation if one is lost before queue acceptance. **[RabbitMQ](https://www.rabbitmq.com/docs/download) with [quorum queues](https://www.rabbitmq.com/docs/quorum-queues) — RECOMMENDED** is a well-documented, Docker-friendly fit for this well-solved work-queue problem. Use a [three-node cluster](https://www.rabbitmq.com/docs/clustering) across separate hosts and, where possible, three independent failure domains, with three replicas per queue and persistent storage per node. Check routing and require [publisher confirms](https://www.rabbitmq.com/docs/confirms); a quorum queue confirms after majority replication. If a node dies before confirmation, acceptance is uncertain: reconnect and retry unconfirmed publishes, expecting duplicates. Writers acknowledge only after database commit. Test failover; consider [Kafka](https://kafka.apache.org/intro/) when retained event history and independent replay become requirements.

#### 3.1.4 Resilient DB Writer

The goal is for every queued copy to be processed successfully, even when another writer has already stored the same observation. **Database uniqueness constraints are required** on a stable source-observation key, so the first committed copy wins. Use an atomic [`INSERT … ON CONFLICT … DO NOTHING`](https://www.postgresql.org/docs/current/sql-insert.html) targeted at that key; an expected duplicate is a successful no-op, not a reason to crash or page the owner. Commit the database transaction before acknowledging RabbitMQ. Other database errors still need normal handling. The [database section](#32-database) owns the observation key and its constraints.

#### 3.1.5 Monitoring/Alerting

**Credentials store:** Watch failed or slow secret retrievals, denied or unexpected access, credential expiry where applicable, and failed rotations. Distinguish vault failures from upstream authentication errors; a healthy reader can still be locked out. Alert on suspected misuse and access failures that threaten ingestion, using audit metadata without logging secret values.

**Reader fleet:** Watch missed polls, missing heartbeats, request failures, response times, and source-data age against the provider's expected schedule. A `200 OK` can carry stale data; without reliable source metadata, freshness is unknown. Compare upstream responses with reader progress to separate source trouble from our own failures. A timeout alone doesn't locate the fault; an independent check can help.

**Message queue:** Watch availability, node and replica health, unconfirmed publishes, publish-confirm latency, publish/consume errors, backlog size, and the age of the oldest waiting message. Alert when a node or replica failure reduces resilience, even if messages are still flowing. A growing backlog can mean writers are blocked or too slow, even when the queue is healthy. Alert before queue delays consume the subscribers' freshness budget.

**DB writer:** Watch failed or slow writes, repeated retries, and time since the last successful database update. Count expected duplicates separately from failed writes, and keep freshness tied to source observation time. Fresh queued data with stalled writes points toward the writer or database. Check that fresh, complete data actually reaches subscribers; a successful write alone doesn't prove that. Run monitoring outside the application's host, notify the owner before freshness or coverage becomes unacceptable, and test that alerts arrive.

### 3.2 Database

The database supports both ingestion and subscriber queries. Our example uses [DigitalOcean Managed PostgreSQL](https://docs.digitalocean.com/products/databases/postgresql/) with TimescaleDB enabled, a useful foundation for observations organized around assets and time. Managed hosting lets the provider operate the infrastructure and backup facilities while we focus on schema, access permissions, queries, and retention. We still choose the availability configuration and test recovery; DigitalOcean documents this [shared responsibility](https://docs.digitalocean.com/products/databases/postgresql/concepts/responsibility-model/). Check [extension support](https://docs.digitalocean.com/products/databases/postgresql/details/supported-extensions/) for the selected edition and version.

Define the stable source-observation key and enforce first-committed-copy-wins here, so independent writers share the same storage contract. For [TimescaleDB hypertables](https://www.tigerdata.com/docs/build/performance-optimization/hypertables-and-unique-indexes), unique keys must include every partitioning column, including time; use a source timestamp that stays the same across repeated deliveries of that observation.

#### 3.2.1 Monitoring/Alerting

Watch database availability, read/write latency, failed transactions, connection saturation, lock waits, and storage headroom. Where replicas are configured, monitor replication lag and failover health. Track backup success and restore-test results so recovery is demonstrated. Alert before these conditions threaten ingestion freshness or subscriber queries, and use an independent check of database access alongside the provider's own health signals.

## License

[MIT](LICENSE) — use, adapt, and share this guide.

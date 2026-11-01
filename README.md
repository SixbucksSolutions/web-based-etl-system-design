# Web-Based ETL System Design

A practical guide for anyone building a web service that consumes data from an external API, adds value to it, and offers access through a paid monthly subscription.

The goal is to turn source data into useful information or tools that people will pay to keep using. As we work through the system, we'll add design notes at each layer and explain the problems they solve, with practical examples for developers building their first service.

## The starting picture

Two paths meet at one database. On the left, an ETL pipeline brings data into the system. On the right, subscribers query the web service to access its added value.

```mermaid
flowchart TB
    upstream["External API<br/>Source data"]
    users["Subscribers<br/>Paid monthly access"]

    extract["Extract<br/>Fetch data from the external API"]
    queries["Web queries<br/>Requests from subscribers"]

    transform["Transform<br/>Validate and prepare the data"]
    api["Service API<br/>Web service entry point"]

    load["Load<br/>Write the prepared data"]
    middleware["Middleware<br/>Request processing"]

    db[("Service database")]

    upstream --> extract
    extract --> transform
    transform --> load
    load --> db

    users --> queries
    queries --> api
    api --> middleware
    middleware --> db
```

**Left: bringing data in.** ETL means *Extract, Transform, Load*: fetch data from a source, prepare it for our needs, and write it to our database.

**Right: serving subscribers.** Subscribers send web requests to the service API. The service processes those requests and queries the stored data to deliver useful results. Results travel back to the subscribers; the diagram shows the downward request path to keep the starting picture simple.

The API and middleware boxes show responsibilities. In application code, middleware commonly runs before an endpoint handler and may also process the response on its way out.

This is a starting sketch. The framework, database engine, hosting, ingestion schedule, and behavior of the external API are still open. We'll fill those in as we learn what the system needs.

## First design notes

- **Give ingestion and user requests separate responsibilities.** The ETL path prepares stored data; the request path serves it. Keeping those responsibilities clear makes each easier to understand and change. They can still live in the same codebase and deployment.
- **Validate data at both boundaries.** Check incoming source data before storing it, and check user input before using it in a query. We'll define the rules once we know the data and the requests.
- **Make the database's job explicit.** Decide what each record represents, how it is identified, and which rules the database should enforce. Both paths need to agree on those meanings.
- **Define how fresh the data needs to be.** Users querying stored data see what has been ingested so far. The acceptable delay will help us choose when ingestion runs and how to show its status.

## What we'll fill in next

- What data does the external API provide, and what limits does it impose?
- What does the service do to that data before storing it?
- What value does the service add that makes a monthly subscription worthwhile?
- What questions do users ask, and who is allowed to see which data?
- What happens when an import or a user request fails?

We'll add detail to the diagram and notes as those answers arrive, with the reason for each design decision alongside it.

## License

[MIT](LICENSE) — use, adapt, and share this guide.

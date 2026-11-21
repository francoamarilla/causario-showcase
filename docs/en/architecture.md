# Architecture

[Versión en español](../arquitectura.md)

## Components

Three separate processes. If a PC is turned off or the portal changes, the application continues to work; only new portal updates are unavailable.

```mermaid
flowchart TB
    subgraph Client
        N[Browser<br/>Angular + Material]
        E[Desktop app<br/>Electron]
        AG[Java agent<br/>login + portal HTTP client]
        E --- AG
    end
    subgraph Cloud
        WEB[nginx<br/>static assets + /api proxy]
        API[API<br/>Spring Boot]
        WK[Worker<br/>email, Google Calendar,<br/>non-working days]
        DB[(PostgreSQL 16<br/>RLS + job queue)]
        WEB --> API
        API --> DB
        WK --> DB
    end
    PORTAL[Court portal<br/>read-only]
    N -->|HTTPS + JWT| WEB
    E -->|same web interface| WEB
    AG -->|HTTPS + PC token| API
    AG -->|firm IP| PORTAL
```

- **API:** authentication, CRUD operations, permissions and endpoints used by the PCs. It is the only process that runs migrations.
- **Worker:** queues synchronization jobs, sends emails, synchronizes Google Calendar and the non-working-day calendar. It does not access the court portal.
- **Agent:** runs on the firm's PCs. It logs in, keeps the session in memory and queries the portal. It has no database access: everything goes through the API using a revocable token.
- **Queue:** a PostgreSQL table rather than a message broker, sufficient for the workload of small law firms.

## Tenant isolation layers

1. Tenant filtering in the persistence layer (Hibernate discriminator).
2. Composite foreign keys (`id`, `estudio_id`), preventing data from being linked to another firm.
3. PostgreSQL Row-Level Security: the application connects using a role that does not own the tables, so the database enforces filtering even if a query omits the application filter.

The tenant is always obtained from the JWT validated on the server, never from a request parameter.

## Case synchronization

```mermaid
sequenceDiagram
    participant W as Worker
    participant DB as PostgreSQL
    participant A as Agent (firm PC)
    participant API as API
    participant P as Court portal

    W->>DB: enqueue jobs by account (every 2 h)
    loop approximately every 2 s
        A->>API: request job (PC token)
        API->>DB: claim job, one per account at a time
        API-->>A: job (or none)
    end
    A->>P: session still valid?
    alt expired session
        A->>P: login (single attempt)
    end
    A->>P: updates, entries, notices (1–2 s between requests)
    A->>API: save cases, entries, notices
    API->>DB: insert without duplicates
    A->>API: finish job (success or error)
    API->>DB: update account status
```

If a job stops reporting its heartbeat (for example, because the PC goes to sleep), the cloud returns it to the queue so another PC can claim it.

## Deployment

- **Cloud:** Docker Compose with four services (database, API, worker, nginx) and a backup service. Only nginx receives external traffic, behind the TLS-terminating proxy. The database and API remain on the internal network.
- **Law firm:** each PC installs the desktop app (Windows installer with automatic updates) and links it to its account using a token displayed only once.

```mermaid
flowchart LR
    subgraph Law firm
        PC1[PC 1<br/>desktop app]
        PC2[PC 2<br/>browser]
        PC3[PC 3<br/>browser]
    end
    PROXY[TLS proxy]
    subgraph Server
        WEB[nginx] --> API[API]
        API --> DB[(PostgreSQL)]
        WK[Worker] --> DB
        BK[Daily encrypted backup] --> DB
    end
    PC1 & PC2 & PC3 --> PROXY --> WEB
    BK -->|off-server copy| S3[(External storage)]
```

See also [Technical decisions](technical-decisions.md) and [CI/CD](ci-cd.md).

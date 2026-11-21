# Technical decisions

[Versión en español](../decisiones.md)

A record of design decisions, including context, choices, alternatives and consequences.

## 1. Database-level tenant isolation

**Context.** Multiple firms share a database. A query that omits a filter must not expose another firm's data.

**Decision.** Three layers: tenant filtering in Hibernate, composite foreign keys (`id`, `estudio_id`) and PostgreSQL Row-Level Security. The application connects with a role that does not own the tables and sets the tenant in every transaction. The tenant comes exclusively from the JWT.

**Alternatives.** Application-level filtering only (an omission can expose data); a separate database per firm (more isolation, but higher operational cost for firms of one to five people).

**Consequences.** Integration tests verify isolation using the same unprivileged role. Data access must go through transactional services; without a transaction the database returns no rows rather than another firm's data. Cross-tenant operations use narrowly scoped database functions instead of disabling security.

## 2. From local installation to desktop and cloud

**Context.** The court portal uses lawyers' credentials and may lock accounts after failed attempts. The original design was a local installation at the firm.

**Decision.** The system evolved into a hybrid model. The cloud stores data, distributes jobs and sends emails. An agent on the firm's PCs handles login and portal requests using the firm's IP. The web application remains accessible through browsers and mobile devices. The cloud never queries the portal.

**Alternatives.** A server-side worker that logs in directly (simpler, but depends on access from a data center IP); a fully local deployment (avoids the cloud, but each firm must operate its server and backups). The local version remains on a branch of the private repository.

**Consequences.** At least one PC must be running for the agent to fetch updates; the home screen reports when none are available. A desktop app with an installer and automatic updates was needed. Each PC is linked using a revocable token.

## 3. Read-only synchronization, endpoint allowlist and portal safeguards

**Context.** The system uses a real lawyer's account on an official portal. An error could lock the account or have procedural consequences.

**Decision.**

- The portal client exposes explicit methods for approved endpoints, rather than a generic URL request function.
- A test fails if a new public method is introduced.
- CAPTCHAs are never bypassed; if one appears, the account is flagged and a notification is generated.
- A login rejected due to invalid credentials is never automatically retried.
- One session per account, with pauses between requests and backoff on errors.
- Actions with effects on the portal, such as opening a notice, occur only on explicit user request and are audited.

**Alternatives.** Automatic login retries (risk of lockout); external CAPTCHA-solving services (rejected on principle).

**Consequences.** Missing a synchronization is preferable to risking the account. Accounts with rejected credentials are removed from the queue until a person corrects them.

## 4. Identifying records when portal IDs change

**Context.** Portal IDs for cases, entries and notices were observed to change between days. An early version that deduplicated by ID produced duplicates.

**Decision.** Cases are identified by case number; entries by entry number within the case; notices by the count for each combination of case number, date and type. The portal ID is retained as a daily value.

**Alternatives.** Deduplicate by full content (fragile when small changes occur); delete and reimport everything (loses read status and the firm's own data).

**Consequences.** Ambiguous cases, such as identical entries on the same day, are all shown with a warning rather than guessed. An unusually large batch of apparently new entries generates a warning about possible renumbering.

## 5. Job queue in a PostgreSQL table

**Context.** Synchronization jobs must be distributed across PCs without two agents processing the same account simultaneously.

**Decision.** The queue is a table. Claiming a job locks the account row; PCs send a heartbeat while processing. A database function returns abandoned jobs to the queue.

**Alternatives.** Kafka or RabbitMQ: additional operational complexity unnecessary at this workload.

**Consequences.** Less infrastructure and transactional consistency with the rest of the database. The design targets a small number of firms and would need reconsideration at significantly larger scale. A real incident in which a sleeping PC left an account stuck for hours led to adding heartbeats and a per-job duration limit.

## 6. Handling sensitive data

**Context.** The system stores portal credentials and information about cases and people.

**Decision.**

- Credentials are encrypted with AES-256-GCM, bound to their row and field, with the key held only in environment variables.
- The API never returns credentials to a person; they are delivered only to a linked PC with a job and each delivery is audited.
- Logs exclude credentials and case content.
- Documents are stored as structured editor JSON, never arbitrary HTML.
- Attachments are always downloaded as files and deleted after seven days.
- Administrator, lawyer and assistant roles, with per-person visibility enforced in the database.
- Append-only audit records.

**Alternatives.** Store credentials only on individual PCs, as in the previous design. This was changed so any linked PC can synchronize any account without depending on the PC that registered it.

**Consequences.** The cloud now holds encrypted secrets, making key management critical. Key rotation by version is planned.

See [Architecture](architecture.md) for the component overview.

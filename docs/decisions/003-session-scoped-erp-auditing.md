# ADR 003: Session-Scoped ERP Auditing & Regulatory Traceability

[← ADR Index](README.md) · [← Main README](../../README.md)

- **Status:** Accepted
- **Context:** Under Danish statutory accounting regulations (*Bogføringsloven*), any automated software modifying double-entry ledger entries or staging reconciliations must provide immutable audit attribution identifying the exact automated process or agent responsible. Standard database connection pools using generic user credentials violate compliance rules if transactions cannot be tied to a verified identity.
- **Decision:** Enforce session-level context injection by executing `SET session.username = 'ai-service@accounting.internal'` immediately prior to executing any write or reconciliation mutation against the Core ERP PostgreSQL/MSSQL database.
- **Benefits:**
  - Automated ledger entries trigger internal database audit logs tied directly to the AI service account.
  - Ensures compliance with Danish auditing authorities with zero application-level logging overhead.
  - Granular rollback capability: if an automated batch is quarantined, all writes tagged with the session token can be isolated.
- **Trade-offs:** Every transaction must execute the session injection command; pooling middleware must ensure session variables do not bleed into subsequent connections (addressed by session resets on transaction close).
- **Alternatives:**
  - Application-level logging: Rejected because raw database modifications bypassing the app wouldn't be attributed.
  - Individual database credentials per worker: Rejected due to connection pooling limits and credential sprawl.

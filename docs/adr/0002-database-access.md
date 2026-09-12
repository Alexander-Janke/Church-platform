# ADR 0002: Database Access

## Status

Accepted

## Context

The Church Platform begins as a TypeScript/NestJS modular monolith, as recorded in [ADR 0001](0001-backend-framework.md). Its multi-tenant modules need predictable queries, transactions, constraints, and safe schema evolution. Sensitive personal and child data make reviewable database behavior especially important.

Database access must support explicit tenant scoping and PostgreSQL Row Level Security (RLS), alongside application authorization. [ADR 0006](0006-tenancy-and-authorization.md) defines that security model. Maintainability and clear conventions for human and Codex-assisted development take priority over hiding SQL behind implicit behavior.

## Decision

- PostgreSQL remains the authoritative relational database.
- Drizzle ORM is the application database-access technology.
- `node-postgres` is the initial PostgreSQL driver.
- Drizzle Kit supplies migration generation and tooling.
- SQL and database behavior must remain visible and reviewable.

This choice does not authorize bypassing repositories, trusted tenant context, or authorization. Drizzle is a data-access tool, not a security boundary by itself. Package versions and the exact TypeScript repository API remain implementation decisions.

## Database Access Rules

- Controllers never access the database directly.
- Application services must not receive an unrestricted global database connection that makes tenant bypass easy.
- Tenant-owned access belongs in module-owned repositories/data-access components exposed through intentional module interfaces.
- Repository APIs make trusted tenant context explicit. Conceptually prefer `findEvent(tenantContext, eventId)` over `findEvent(eventId)` for a tenant-owned event. Exact interfaces are deferred until implementation.
- Explicit tenant scoping applies to reads, inserts, updates, deletes, bulk operations, joins, aggregates, and search queries. It is not only a read concern. Derive insert ownership from the validated context and constrain mutation targets and joined relationships.
- Non-tenant resources use explicit account, participant, or platform policies rather than a fake tenant context.
- Raw SQL must be parameterized, reviewed, and preserve tenant isolation and RLS expectations. Dynamic identifiers must come from controlled definitions, not arbitrary request input.
- Return explicit, authorized response DTOs rather than exposing database rows indiscriminately.

## Transaction Rules

Multi-step invariant changes use transactions. Future examples include Primary Owner transfer, membership transitions, capacity/waitlist changes, role changes, and child/guardian relationship changes.

All repository calls participating in one transaction must use the same transaction handle/context and therefore the same database connection. Never fall back to a global connection inside an active transaction.

Every tenant-owned database operation, including a single read, runs within a transaction whose tenant context is established locally for that transaction. Context setup and protected queries must use the same handle. Missing context fails closed. Reusable connection-global tenant state is prohibited: pool reuse must not leak context between requests, after commit, or after rollback.

Keep transactions as short as reasonably possible. Do not hold them open for email delivery, remote API calls, or other slow external work. Transaction rollback and concurrency behavior must preserve business invariants; the exact locking strategy belongs to the relevant use case.

## Migration Rules

- Version-control migrations. Review generated SQL before application, including constraints, data changes, indexes, grants, and RLS policies where relevant.
- Production schema changes use migrations, never automatic schema synchronization or direct schema-push workflows.
- Test migrations against a clean PostgreSQL database; test upgrades from a representative earlier schema for important changes.
- Deployment has one controlled migration step, not competing migration attempts from every API or worker instance.
- Destructive changes require deliberate review. Use expand/migrate/contract when application versions may overlap.
- Consider rollback or forward-recovery strategy for risky migrations; a reverse migration cannot automatically restore deleted data.
- Use separate migration/schema-owner credentials and runtime credentials in staging and production. Runtime roles must not need schema-owner privileges.

Drizzle Kit generation does not remove the need for reviewed PostgreSQL-specific migration SQL. RLS and privilege changes must be represented in version-controlled migrations and verified, not depend on undocumented manual database state.

## Alternatives Considered

### Prisma

Prisma offers a strong type-safe developer experience and mature schema, migration, and client tooling. It could support this platform, but this project prioritizes direct visibility into PostgreSQL queries and explicit transaction/RLS integration. Drizzle's schema and SQL-oriented query model is a closer fit for those conventions. Prisma is not rejected as insecure or incapable of multi-tenancy.

### Kysely

Kysely offers an explicit typed SQL approach and would fit complex PostgreSQL queries. Drizzle provides the preferred combined schema definition, query, and migration-generation/tooling approach, reducing the conventions the project must assemble separately.

### TypeORM

TypeORM has an established ecosystem and NestJS integration. Its entity-oriented abstractions and optional implicit behaviors, such as relationship loading and cascades, are less desirable for this project's emphasis on explicit query scope and reviewable database effects. Those behaviors can be controlled, but Drizzle better matches the chosen conventions.

No decision here depends on unsupported performance benchmarks.

## Security Considerations

Application authorization establishes entitlement; explicit repository scoping, RLS, and database constraints provide additional protection. RLS does not replace tenant, object-level, or sensitive-field authorization.

Runtime roles must not be superusers, have `BYPASSRLS`, or own protected tenant tables. Apply `FORCE ROW LEVEL SECURITY` where appropriate, as defined in ADR 0006. Restrict credentials and database privileges to their purpose, including workers. Do not log credentials or sensitive query parameters.

## Testing Considerations

Support unit/service tests and integration/API tests. Use real PostgreSQL whenever constraints, SQL semantics, RLS, locking, transactions, or pooling behavior matter; mocks or another database cannot prove these guarantees.

Test reviewed migrations, transaction rollback and atomicity, cross-tenant foreign keys, and the full isolation matrix in ADR 0006. RLS integration tests execute protected operations as the actual runtime database role, not the migration owner. Verify that repository calls do not escape their transaction and that pooled/concurrent operations cannot reuse another tenant's context.

Authorization and tenant-isolation tests are release-blocking from the first tenant-owned tables/services. The concrete testing stack is finalized separately under [TESTING.md](../TESTING.md).

## Consequences

The project gains a consistent schema/query/migration workflow, typed queries, explicit PostgreSQL behavior, and repositories that can be tested independently of HTTP controllers. These conventions support onboarding and focused Codex implementation sessions.

Costs include learning SQL and Drizzle, writing PostgreSQL-specific migrations where needed, and enforcing repository and transaction discipline. Types alone cannot prove correct permissions or tenant predicates. Mitigate these costs with small module interfaces, reviewed SQL, shared transaction conventions, real database tests, and mandatory negative authorization tests. Avoid both unrestricted convenience helpers and a large generic repository abstraction that hides query behavior.

## Future Review Triggers

Revisit this choice only for concrete maintenance/security problems, unsupported required PostgreSQL behavior, migration reliability problems, or measured limitations that cannot reasonably be addressed within the chosen stack. Newer libraries or synthetic benchmark differences alone are not reasons to change it.

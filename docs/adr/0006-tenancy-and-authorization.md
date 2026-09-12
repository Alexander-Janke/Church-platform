# ADR 0006: Tenancy and Authorization

## Status

Accepted

## Context

One church equals one tenant, but users can have legitimate relationships with multiple churches and some church content is public. Platform identities, private communication, and personal Bible notes also exist outside church ownership. A blanket membership requirement would contradict these product behaviors.

The NestJS modular monolith must protect tenant boundaries, object scope, privacy, and highly sensitive child data across every access path. [ADR 0001](0001-backend-framework.md) defines the backend structure; [ADR 0002](0002-database-access.md) defines PostgreSQL, Drizzle, and transaction discipline.

The accepted defense consists of server-side application authorization, trusted server-created tenant context, explicitly scoped repositories/queries, PostgreSQL RLS for tenant-owned tables, constraints preventing cross-tenant relationships, and mandatory isolation tests. RLS is an additional security boundary, not a replacement for application authorization.

## Access Classes

Church-related access is not synonymous with church membership. Use these conceptual access/relationship classes:

| Class | Example and required policy |
|---|---|
| PUBLIC | An explicitly published public church profile; no account or membership required. |
| AUTHENTICATED_USER | Following a church requires an authenticated platform user, not prior membership. |
| FOLLOWER | Follower-targeted content requires the permitted follower/audience relationship. |
| MEMBER | Member directory access requires membership plus applicable permission and privacy rules. |
| OBJECT_AUTHORIZED | A private group resource requires authorization for that specific group. |
| TENANT_ADMIN | An administrative operation requires the applicable tenant administrative permission. |
| PRIMARY_OWNER | Church deletion or ownership transfer requires the protected ownership capability and additional security requirements. |

These are not necessarily database roles, a universal ordered hierarchy, or hard-coded role-name checks. Effective audience rules follow [PERMISSIONS.md](../PERMISSIONS.md), including member access to follower content where specified. Administrative rank never grants automatic access to all private content.

## Trusted Tenant Context

A client may request `/api/v1/churches/{churchId}/...`. The supplied `churchId` is a requested resource scope, not authorization.

The server must:

1. Authenticate the actor when required.
2. Resolve the requested church.
3. Determine the actor's relationship/access class.
4. Determine effective permissions.
5. Evaluate object/context-specific authorization.
6. Construct a trusted server-side tenant context.
7. Pass that context to tenant-aware application/database operations.

Never trust a tenant ID because it appears in a route, body, header, or frontend state. Created resources take tenant ownership from validated server context. Reject unknown fields and protected ownership fields outside the accepted request DTO rather than silently accepting them.

The context represents a validated operation scope; it is not an unrestricted grant to every resource in the church. Public contexts permit only explicitly public operations and filtered fields. Service-level authorization still applies when a context is reused for another use case.

Resolving relationships or resource metadata may itself require database reads before final authorization. Keep these preliminary reads narrowly scoped inside trusted authorization/data-access components, with no resource disclosure or mutation before entitlement is established. Tenant-owned reads still use explicit scope and transaction-local RLS. This internal lookup scope must not be exposed as an authorized application context or implemented with a general RLS bypass. Final transaction-bound operations must recheck authorization-sensitive state where concurrency can invalidate an earlier decision.

## Authorization Flow

```text
Session/authentication
→ requested tenant/resource
→ relationship/access class
→ effective permissions
→ object/context authorization
→ trusted tenant context
→ transaction-local database tenant context
→ explicitly scoped repository query
→ RLS
→ filtered response DTO
```

Not every endpoint requires every stage. PUBLIC resources do not require authentication or membership; global account resources require their own ownership policy. No protected operation may omit a stage relevant to its security requirements. NestJS guards can handle coarse prerequisites, but application/services must also enforce tenant, object, contextual, and sensitive-data rules.

## PostgreSQL Row Level Security

Enable RLS from the first real tenant-owned tables. Tenant ownership must be explicit, normally a non-null `church_id`.

- Protect reads and writes, using policy visibility conditions and write checks as appropriate. Policies must constrain existing rows and new/changed ownership.
- Missing, invalid, or mismatched tenant context fails closed; never fall back to all tenants.
- Establish database tenant context transaction-locally, using the same transaction handle for context setup and protected queries, including reads.
- Never use reusable connection-global tenant state. Commit, rollback, errors, and pooled connection reuse must not carry another request's scope.
- Runtime roles must not be superusers, have `BYPASSRLS`, or own protected tenant tables.
- Use `FORCE ROW LEVEL SECURITY` where appropriate to subject table-owner access to policies as well; it does not neutralize superuser or `BYPASSRLS` privileges.
- Keep migration/schema-owner credentials separate from runtime credentials. Workers also use restricted runtime roles.

RLS does not prove that the user is entitled to the tenant. Application authorization establishes entitlement. RLS primarily protects against accidental or incorrect database access after the application has established trusted tenant context. Both application authorization AND RLS are required.

A tenant RLS policy does not by itself enforce group membership, participant privacy, sensitive fields, or public-only visibility within that tenant. It also cannot detect an application that wrongly authorizes Tenant B and consistently supplies B as its trusted scope. Tests must independently exercise entitlement and database isolation. RLS is not a defense against a fully compromised application or privileged database operator.

This ADR defines the policy requirements, not SQL implementations. Review concrete policies, grants, and context plumbing with the first relevant schema migrations.

## Database Constraints

Represent isolation structurally where practical. For tenant-owned relationships, consider composite ownership keys such as `(church_id, resource_id)` and corresponding foreign keys. Prevent a row in Church A from referencing an incompatible tenant-owned row in Church B, including join tables and bulk inserts.

Use PostgreSQL constraints where they can safely enforce an invariant rather than relying only on application checks. Exact schemas are deferred to each module; legitimate references to global identities require their own relationship rules.

## Non-Tenant-Owned Resources

Do not force every resource into a church tenant. Examples include:

- platform user identity;
- personal/private Bible notes;
- direct-message conversations with participant-based ownership;
- some account, session, and security records;
- platform administration records.

These resources need explicit ownership, participant, or platform authorization policies. Do not invent a synthetic church ID. Whether additional resource-specific RLS policies are useful must be decided with their actual ownership model; this is not permission to expose them through unrestricted repositories.

Child/family identity needs an explicit ownership and sharing design, including guardian relationships and any church operational access. Arbitrary tenant assignment must not substitute for that design.

Platform superadmin access is a separate, explicitly authorized and audited capability. It must not use an ordinary runtime RLS bypass or imply access to private messages, private prayers, Bible notes, or child medical information.

Cross-church calendar/search aggregation and public discovery must select only authorized/public resources. Implement a reviewed scoped aggregation or public projection when those modules are built; do not grant general cross-tenant access to private tables to support discovery.

## Membership and Roles

Follower, Member, Inactive, and Left Church are relationship states. Member is fundamentally a church relationship state, not an ordinary manually assignable administrative role. Basic member capabilities derive from the current relationship and applicable policy.

Administrative/contextual roles may include Group Leader, Area Leader, Event Administrator, Children's Worker, and Main Church Administrator. Their assignments have explicit tenant/object scope. Primary Owner is a protected ownership relationship/capability. Custom roles must not bypass protected privilege or delegation boundaries.

Permission representation remains governed by [PERMISSIONS.md](../PERMISSIONS.md) and future implementation work. Use centralized permission and resource policies rather than scattered role-name comparisons. Granting a leadership role must not silently change the person's membership state.

## Primary Owner Security

V1 requires exactly one Primary Owner per church. Church deletion is Primary-Owner-only; ordinary custom roles cannot receive deletion authority.

Ownership transfer requires mandatory 2FA, recent step-up authentication, and an audit event. The transfer must preserve the exactly-one-owner invariant transactionally, including failures and concurrent attempts. Do not enable transfer before these security dependencies exist.

Normal transfer is controlled by the current owner. Exceptional support recovery remains subject to the separate security procedures in [SECURITY.md](../SECURITY.md); this ADR does not grant support a general bypass or define that recovery workflow.

## Sensitive Child Data

A medical-data read permission alone is insufficient. Highly sensitive child information requires explicit permission, valid operational context, current need-to-know, and appropriate audit logging. This applies to administrators and the Primary Owner as well as workers.

Guardian access derives from an explicit, authorized child/guardian relationship and its allowed actions; it is not inferred from membership, surname, or address. That relationship supplies the relevant parent-care context without requiring a church worker role.

Detailed child identity, sharing, and sensitive-access modeling remains deferred. Child-specific registrations, calendar behavior, and check-in/out must wait until the child/guardian/sensitive-access foundation exists. Phase 3 must not bypass this dependency through generic guest fields or temporary insecure models.

## Background Jobs

Jobs operating on tenant data must carry explicit identifiers/context and use the same scoped application/repository rules as requests. Prefer IDs over unnecessary private content in payloads.

Serialized permission decisions are not permanent grants. User-requested sensitive jobs must revalidate current authorization at execution where revocation or changed relationships affect entitlement. Scheduled system workflows require a narrowly defined system authorization policy, not an arbitrary user impersonation or database bypass. Establish transaction-local context for each tenant operation; never inherit the previous job's connection scope.

## Realtime

Authenticate connections where required, authorize subscriptions, and derive rooms/channels server-side. Tenant IDs supplied by a realtime client are not authorization. Revalidate sensitive operations and respond to revoked sessions, relationships, or permissions by removing or denying access as appropriate.

Event delivery and payload filtering must honor current resource/audience policy. Realtime remains inside the modular backend initially; a transport or scaling decision cannot bypass these requirements.

## Files and Object Storage

Object keys and bucket paths are not authorization. Before private file access, authorize the metadata/resource relationship. Tenant-owned file metadata follows repository, RLS, and constraint rules; global/private attachments use their own ownership policies.

Signed downloads must be issued only after authorization and have appropriate short validity. Highly sensitive downloads may require authenticated streaming and current authorization rather than relying solely on a longer-lived signed URL. Storage/provider details belong to a separate ADR.

## Search and Exports

Apply authorization during selection, including joins, counts, snippets, and suggestions. Never retrieve broad cross-tenant data and filter it only afterward in application memory.

Exports require the same or stronger tenant and field-level authorization as normal views, including explicit export permission where specified. Protect generated artifacts and downloads, audit sensitive exports, and revalidate background execution as needed. A broad church export must not include unrelated personal or participant-owned private content.

## Caching

Do not introduce permission or private-data caching now. If introduced later, keys must include appropriate tenant, resource, and authorization context; authorization changes require a defined invalidation strategy. Cached data must never expand access. Tenant-isolation tests must cover caching behavior, including revocation and cross-tenant key collisions.

## Testing Requirements

From the first tenant-owned tables/services, release-blocking tests must prove:

- Tenant A can access its permitted data.
- Tenant A cannot read, update, or delete Tenant B's protected data.
- Tenant A cannot insert cross-tenant relationships.
- Missing tenant context fails closed.
- Incorrect RLS context fails closed: for example, a repository scoped to A under database context B must disclose/mutate neither tenant's protected data.
- Bulk operations, joins, aggregates, search, and exports preserve isolation.
- Pooled and concurrent database connections do not leak tenant context, including after commit, rollback, or errors.

Use real PostgreSQL and the actual runtime database role for RLS integration tests. Migration-owner test connections may prepare fixtures but must not execute the operations asserted to be protected. Verify role restrictions and policy enforcement independently of repository filters, including attempted ownership changes and cross-tenant write checks.

Initial fixtures must exercise the foundational read/write, relationship, missing/mismatched context, and pooling guarantees immediately. Add feature-specific search/export, file, worker, realtime, and cache tests in the same changes that introduce those paths; do not postpone foundational isolation tests to later product phases.

Also test application entitlement independently: requested Tenant B must not become trusted context merely because the client supplies its ID. Public/follower positive tests must prove legitimate access without membership while denying internal content. Cover object scope, protected request fields, response filtering, permission revocation, nondelegable owner powers, and sensitive-child context as implemented. Authorization and isolation failures block release under [TESTING.md](../TESTING.md).

## Security Consequences

Independent application, query, RLS, constraint, and test layers reduce accidental cross-tenant access across entry points. Their cost is explicit transaction/context handling, database-role discipline, and broader negative/concurrency tests. Keep the implementation centralized and reviewable without hiding individual resource policies.

These controls do not replace secure authentication/sessions, parameterized queries, least privilege, audit protections, or privacy-aware responses. Deny ambiguous protected access rather than silently broadening permissions. No RLS policy, repository, schema, or permission implementation is created by this record.

## Future Review Triggers

Review concrete policies as modules introduce new ownership models, sharing, aggregation, or sensitive operational contexts. Revisit the architecture only for a demonstrated isolation defect, material PostgreSQL/runtime limitation, or fundamentally changed tenancy requirements. Scaling work must preserve all security layers unless an explicit reviewed replacement provides equivalent safeguards. Convenience, framework changes, or benchmark results alone do not justify removing application authorization or RLS.

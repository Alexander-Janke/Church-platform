# AGENTS.md

## Project Mission

This repository contains the Church Platform:
a modern, multi-tenant SaaS platform for Christian churches.

The platform must be:
- simple and intuitive
- mobile-first
- privacy-focused
- secure
- scalable
- cost-efficient
- accessible
- visually modern and high quality

The product specification in `/docs` is the authoritative source for product behavior.

Codex must not invent major product features that are not described in the product specification.

---

## Core Technology

Mobile:
- Flutter

Web:
- React
- Next.js

Backend:
- API-first architecture
- PostgreSQL
- Redis or equivalent cache when necessary
- background job queue/workers
- object storage for files and audio

Infrastructure:
- containerized deployment
- Docker
- EU hosting preferred
- initial deployment should be cost-efficient
- netcup is the preferred initial hosting provider

Architecture:
- start with a modular monolith
- avoid unnecessary microservices
- keep clear module boundaries so components can later be separated if scaling requires it

---

## Multi-Tenancy

The application is multi-tenant.

One church equals one tenant.

Every request involving church-specific data must verify:
1. the authenticated user
2. tenant membership
3. the user's permissions inside that tenant

Never trust tenant IDs sent by the client without validating access.

A user belonging to Church A must never be able to access data belonging to Church B unless the user also has legitimate access to Church B.

Tenant isolation must be enforced server-side.

---

## Security Rules

Security is mandatory.

Never:
- hardcode secrets
- commit passwords
- commit API keys
- commit private keys
- log passwords or authentication tokens
- expose sensitive personal data in logs
- trust client-side authorization
- bypass tenant checks

Use environment variables or secure secret storage.

Critical operations must be auditable.

2FA is mandatory for:
- Primary Owner
- main church administrators
- platform superadmins

Normal users may enable 2FA optionally.

Critical actions require step-up authentication where specified.

---

## Privacy Rules

Privacy by default.

Optional personal data must be private unless the user explicitly shares it.

Sensitive data includes:
- addresses
- phone numbers
- birthdays
- child information
- medical information
- emergency contacts
- pickup permissions

Sensitive information must only be accessible to users with explicit authorization.

Access to highly sensitive data should be logged where required.

Private direct messages must not be readable by platform administrators during normal operation.

Private prayer requests and private Bible notes must also remain protected.

---

## Product Scope Rules

Do not implement features outside the defined scope without explicit instruction.

The following are NOT part of V1:

- AI sermon summarization
- AI Bible features
- AI duty scheduling
- campus or branch hierarchy
- direct video uploads
- in-app payments
- donation campaigns
- coupon systems
- passkeys
- audio/video calling
- recurring absence patterns
- personal project management
- appointment booking
- automatic translation of church-created content
- public general-purpose API

Do not introduce these features unless explicitly requested.

---

## UX Rules

The application must not feel like old administrative software.

The UI should be:
- modern
- stylish
- calm
- high quality
- easy to understand
- accessible

Prefer:
- clear cards
- simple navigation
- understandable icons
- short workflows
- helpful empty states
- clear error messages

Avoid:
- overloaded dashboards
- deeply nested menus
- unnecessary configuration
- showing inaccessible features as disabled

If a user does not have permission for a feature, hide it when appropriate.

Support:
- light mode
- dark mode
- system theme

Accessibility must include:
- screen reader support
- scalable text
- sufficient contrast
- keyboard accessibility on web
- adequately sized touch targets

---

## Internationalization

The platform must support:

- German
- English
- Portuguese

UI strings must never be hardcoded directly into components if they should be translated.

Use the translation/i18n system.

Church-created content does not require automatic translation in V1.

---

## Development Workflow

Before implementing a significant feature:

1. inspect existing code
2. inspect relevant product documentation
3. identify affected modules
4. identify permission requirements
5. identify database changes
6. identify required tests
7. create a concise implementation plan
8. implement the feature
9. run tests
10. fix failures
11. re-run tests

Do not stop after writing code if tests can be executed.

---

## Testing Requirements

Every feature must include appropriate tests.

At minimum consider:

- unit tests
- integration tests
- authorization tests
- tenant isolation tests
- validation tests

For critical user flows, add end-to-end tests where appropriate.

### Mandatory Tenant Test

For every API or service that accesses tenant-specific data, add or maintain a test proving:

A user from Tenant A cannot access protected resources from Tenant B.

This is a mandatory security requirement.

---

## Definition of Done

A task is not complete unless:

- code builds successfully
- formatter passes
- linter passes
- relevant tests pass
- permission checks are implemented
- tenant isolation is preserved
- errors are handled
- loading states are handled where applicable
- empty states are handled where applicable
- no secrets were introduced
- no sensitive information is exposed in logs
- documentation is updated if behavior or architecture changed

---

## Git Rules

Keep commits focused.

Do not mix unrelated changes.

Use meaningful commit messages.

Before completing work:
- inspect git diff
- run required checks
- check git status

Do not delete unrelated user changes.

---

## Database Rules

PostgreSQL is the primary relational database.

Database changes must use migrations.

Never manually depend on undocumented database state.

Important tables should include timestamps where appropriate.

Prefer explicit foreign keys and database constraints.

Do not rely only on application validation for data integrity.

Tenant-owned entities must have a clear tenant relationship.

---

## API Rules

All protected API endpoints must enforce authentication and authorization server-side.

Validate inputs.

Return consistent error structures.

Do not expose internal stack traces to clients.

Use pagination for potentially large result sets.

Prevent N+1 query behavior.

Use transactions for operations that must succeed atomically.

---

## Performance Rules

Do not prematurely optimize, but avoid obviously inefficient architecture.

Use:
- pagination
- appropriate indexes
- efficient queries
- background workers for slow operations

Cache only where safe.

Never allow cached data from one tenant or permission context to leak into another.

---

## Background Jobs

Use background workers for work such as:

- email
- push notifications
- imports
- media processing
- scheduled publishing

Jobs should be:
- retry-safe
- idempotent where possible
- observable
- failure-aware

---

## Documentation

Important architectural decisions must be documented.

Relevant documentation belongs under:

`/docs`

Prefer updating existing documentation instead of creating duplicate descriptions.

The product documentation is authoritative for intended application behavior.

---

## Decision Rule

When choosing between multiple valid solutions, prefer:

1. correctness
2. security
3. simplicity
4. maintainability
5. good UX
6. low operating cost
7. scalability

Avoid clever complexity when a simpler solution is sufficient.
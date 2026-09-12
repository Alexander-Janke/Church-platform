# ADR 0003: Authentication and Sessions

## Status

Accepted

## Context

The Church Platform needs one authentication boundary for Flutter, Next.js web, and Next.js platform-admin clients. Sensitive personal and child data, private communication, multiple devices, and protected administrative capabilities require explicit revocation, authentication assurance, and recovery policies.

The backend is a NestJS/TypeScript modular monolith using Express initially ([ADR 0001](0001-backend-framework.md)). PostgreSQL and Drizzle are accepted ([ADR 0002](0002-database-access.md)); tenant and object authorization remain application responsibilities ([ADR 0006](0006-tenancy-and-authorization.md)). The initial deployment favors cost-efficient, self-hosted EU infrastructure. Authentication must remain maintainable without building every security-critical primitive from scratch.

## Decision

Use Better Auth as the initial authentication library, encapsulated behind the application's own NestJS `AuthModule`. Use PostgreSQL-backed server-side sessions with opaque credentials for both browser and mobile clients. Do not introduce an application JWT access-token/refresh-token architecture initially.

Support email/password, Google, Apple, email verification, explicit secure account linking, TOTP, recovery codes, privileged assurance, and recent-authentication/step-up checks. Application-owned security policy takes precedence over library defaults. Better Auth is an implementation component, not the application's security architecture.

This record accepts the architecture and policy, not an installed package version, adapter, schema, or implementation. Supported integration behavior must be verified against the actual versions selected later.

## Authentication Methods

### Email and password

Email/password registration requires email verification before full account use. Unverified identity must not be treated as fully trusted for security-sensitive capabilities. Any limited verification/onboarding state must not grant protected capabilities.

Use a modern memory-hard password hashing algorithm. Argon2id is the preferred application choice where the integration permits application-controlled hashing. Select and test parameters during implementation, including resource usage and safe verification/rehashing behavior; no parameters are prescribed here. Allow long passwords and password managers, avoid silent truncation, and define a strong length/abuse policy without arbitrary weak rules. Never store plaintext or reversibly encrypted passwords.

### Social authentication

V1 supports Google and Apple. Social-login-only users need no additional local password to use the platform, enroll TOTP, or satisfy security policy. Validate provider responses and identity claims server-side. Provider-verified email may satisfy verification only under an explicitly validated provider trust policy; an unverified or merely supplied address is insufficient.

Use supported OAuth/OIDC flows with appropriate state, nonce, PKCE, redirect validation, and replay protections for the provider/client combination. Flutter uses a secure system-browser callback flow; mobile apps must not contain provider client secrets. Exact callback topology is deferred. Provider tokens are not Church Platform session credentials, and social login alone does not establish privileged 2FA assurance.

## Account Linking

Never silently merge accounts because different authentication methods present the same email address. Require an explicit, secure linking flow proving control of the existing account and the new provider identity. Match provider identities using validated provider identifiers, not email alone.

Linking and unlinking require recent authentication within the five-minute window and the account's required assurance. Prevent CSRF, replay, and linking a provider identity already owned by another account. Do not allow unlinking to leave the account without a usable approved sign-in/recovery path. Notify the user of provider link/unlink changes where practical.

Disable or constrain any library automatic email-based linking behavior that violates this policy. The exact linking UX is deferred; email equality, even for a verified address, does not replace explicit proof and consent.

## Authentication Boundary

```text
Client
→ NestJS API
→ application AuthModule
→ Better Auth integration
→ PostgreSQL/session storage
```

Other modules depend on application-owned actor/session/assurance abstractions. Do not import Better Auth throughout the codebase or use its session objects as the universal domain authorization model. Tenant relationships, permissions, contextual authorization, sensitive-field filtering, and Primary Owner invariants remain application-owned.

All exposed library routes, callbacks, server APIs, and alternate entry points must satisfy application policy. Merely wrapping normal controllers is insufficient if raw authentication endpoints can bypass linking, assurance, recovery, or revocation rules. Services must enforce relevant security requirements independently of client UI and coarse route guards.

Authentication identity/session records are account-scoped, not assigned synthetic church ownership. Keep library data access inside the authentication boundary, reconcile its PostgreSQL/Drizzle integration with ADR 0002, and use reviewed version-controlled migrations later. Library schema tooling must not become an automatic production synchronization path or require runtime schema-owner privileges.

## Session Model

The client receives an unpredictable opaque session credential; authoritative validity, expiry, revocation, and assurance state remain server-side in PostgreSQL. Each browser/device session can be revoked independently. Both transports use the same application session model, not separate identity systems.

Server-side sessions fit immediate invalidation, session/device management, and privileged-session control with less complexity than custom JWT rotation and revocation. Do not enable stateless/cookie-cached acceptance that allows a revoked session to remain valid. Every protected operation must use authoritative current session state; fail closed if it cannot be verified. Realtime connections must also respond to revocation under ADR 0006.

Prevent fixation: establish fresh credentials on successful authentication and define safe credential rotation at assurance/security transitions during implementation. Rotation must invalidate replaced credentials and must not silently restart absolute lifetime or manufacture recent authentication. Store credentials/verifiers with least privilege and never expose them through ordinary session metadata responses.

## Web Session Transport

Next.js web and platform-admin clients use cookies with `HttpOnly`, `Secure` in HTTPS environments, appropriate `SameSite`, and narrowly appropriate host/domain/path scope. The session credential must not be available to ordinary frontend JavaScript, including through authentication JSON responses, exposed headers, or session-list APIs. Do not store authentication credentials in localStorage.

Next.js may forward requests within an approved browser/API topology, but the NestJS AuthModule remains authoritative. Automatic cookie attachment requires CSRF and origin protections. Exact cookie names, domain scope, and application-domain topology are deferred; do not broadly share privileged cookies across subdomains by default.

## Mobile Session Transport

Flutter receives an opaque bearer-style session credential from the backend authentication system and stores it in operating-system secure storage. Ordinary preferences, insecure backups, URLs, logs, and analytics must not contain the credential. Send it over HTTPS in the intended authorization transport.

The credential is server-revocable and password-equivalent for its valid lifetime if stolen. Mobile does not justify a separate JWT architecture. Verify supported bearer transport and secure OAuth-to-app credential delivery during integration; do not place a reusable session secret in a redirect URL. Keep browser responses from exposing mobile bearer credentials.

## Session Lifetime Policy

Initial product-security defaults:

| Scope | Inactivity timeout | Absolute limit / freshness |
|---|---|---|
| Normal web session | 7 days | 30 days absolute lifetime |
| Normal mobile session | 30 days | 90 days absolute lifetime |
| Privileged administrative elevation | 15 minutes | 8 hours maximum elevated duration |
| Critical-operation step-up | Not a sliding activity timer | Authentication within the preceding 5 minutes |

A normal authenticated session identifies the actor. It does not by itself authorize privileged operations. Elevated assurance permits protected privileged actions only while the required factors, current capabilities, and elevation timers are valid. Recent step-up is a separate fresh proof for critical actions; it is not satisfied merely because an eight-hour elevation is still active.

Enforce all applicable clocks server-side, using the earliest expiry. Inactivity is measured from qualifying activity; absolute session lifetime runs from the original session establishment and is not extended by activity, rolling renewal, or token rotation. Elevated duration runs from explicit elevation. Expired elevation requires new proof while an otherwise valid normal session may remain usable for permitted non-privileged actions. Background polling must not indefinitely extend privileged inactivity. Define qualifying activity precisely during implementation.

Record factor/reauthentication times explicitly, bound to the current session and relevant assurance scope. A client timestamp, user-agent label, recently refreshed credential, or library session creation timestamp alone is not sufficient proof. Use trusted issuance/transport policy for web versus mobile lifetimes; a client-supplied label must not extend a browser session's policy.

These values are initial defaults. Later configurability must not weaken required platform security and requires deliberate review. Library defaults or rolling expiry must not silently override them.

## Two-Factor Authentication

Use TOTP as the initial second factor, with backup/recovery codes. Normal users may enable 2FA optionally; choosing it must provide real protection across supported sign-in paths. Verify enrollment before treating TOTP as active. Protect TOTP seeds as secrets requiring secure storage/encryption; they cannot be handled like non-recoverable password hashes because verification needs the seed. Define key handling during implementation.

Accounts currently possessing protected privileged capabilities require 2FA. At minimum this covers Primary Owner, Main Church Administrator, and Platform Superadmin, plus custom roles containing capabilities classified as requiring 2FA. Enrollment alone is not evidence that the current session satisfied the second factor.

Explicitly distinguish authenticated, 2FA-satisfied where required, and recently step-up-authenticated states. Email/password, Google, Apple, and future authentication methods must pass the same application assurance gates. A successful social/OAuth callback alone must never activate privileged access. Provider-side MFA claims or remembered-device behavior must not silently substitute for the project's TOTP requirement. Approved recovery-code use is handled through the controlled recovery policy below.

Do not assume Better Auth challenges every sign-in method. Its current documentation distinguishes credential and social sign-in enforcement; actual integration behavior requires explicit tests. See [Better Auth 2FA documentation](https://better-auth.com/docs/plugins/2fa). Incomplete challenges may permit only narrowly defined completion/recovery operations, never protected privilege.

## Privileged Access

Protected capabilities require both authorization and current authentication assurance. Permission/capability metadata must express elevated-assurance requirements; do not encode the rule solely as `role === "admin"`. Custom roles cannot edit away protected metadata. Member remains a church relationship state, not an administrative role.

Safe transitions must be designed and tested before enabling privileged features:

- **Grant to an account without 2FA:** reject activation or keep the grant pending until verified enrollment and required assurance; never provide an active unprotected capability.
- **Enable 2FA:** confirm factor possession before activation; do not automatically elevate unrelated existing sessions.
- **Disable/reset 2FA:** require recent authentication and existing required assurance or the approved recovery flow. Prevent disabling while protected privileges remain usable, or perform a safe coordinated transition that first makes those privileges unavailable.
- **Lose/recover a factor:** retain restrictions until sufficient recovery proof is established; no email/password reset shortcut to privilege.
- **Revoke privilege:** immediately stop authorizing the capability and invalidate affected elevation/step-up state across sessions as appropriate.

Exactly-one-Primary-Owner must remain intact. Locking privileged use during recovery is not permission to delete the ownership relationship or create a second owner. Recipient eligibility for transfer and initial church-owner creation must include the 2FA prerequisite. Detailed transition UX is deferred.

## Step-Up Authentication

Critical operations require authentication within five minutes, checked server-side at the operation, not only on entry to a settings screen. Examples include ownership transfer, church deletion, disabling/resetting 2FA, linking/unlinking providers, changing highly sensitive security settings, email changes, and high-impact platform-superadmin actions.

Use proof appropriate to the account and operation. Social-only accounts must have a supported recent provider reauthentication/factor flow without being forced to create a password. Silent provider SSO or an existing platform cookie must not automatically count as fresh proof. For privileged accounts, step-up cannot downgrade mandatory 2FA or revive expired/revoked elevation without the required proof. Bind challenges and their completion to the initiating actor/session and intended security scope.

Exact reauthentication mechanisms and operation-specific factor rules must be verified during integration. If the required proof cannot be established, deny the critical action rather than treating library success as equivalent assurance.

## Recovery

Generate backup codes securely during implementation, show them only in an appropriately authorized flow, store them safely (prefer non-recoverable verifiers/hashes where technically appropriate), consume them atomically once, and support revocation/regeneration. Regeneration invalidates the previous set and requires appropriate recent authentication. Never log the codes.

A recovery code is an approved recovery proof, not a universal support bypass. Verify its binding to the account and the other proof required by the recovery flow. Do not silently remove mandatory privileged 2FA or grant unrestricted elevation after factor loss. Until sufficient proof and required factor restoration are complete, protected capabilities may need to remain unavailable. Exact privileged recovery/elevation handling must be specified and tested before release.

No master password, universal recovery code, informal database edit, or generic support impersonation is permitted. Exceptional Primary Owner recovery is intentionally deferred and requires its own explicit security design before implementation.

## Password and Email Changes

Password reset uses cryptographically secure, time-limited, single-use tokens and non-enumerating responses. A successful reset revokes **all existing sessions** for that account, including browser/mobile sessions and elevation/step-up state. Do not report success while old sessions remain usable. Reset does not satisfy or disable privileged 2FA. Audit the reset and notify the user by email.

Authenticated password changes require appropriate current/recent proof and security notification/auditing. Define their exact session-revocation behavior before implementation; the all-session reset rule above is mandatory and must not be weakened by library defaults. Adding a local password to a social-only account must be an explicitly protected security operation, never an implicit registration requirement.

Email changes require recent authentication and verification of the new address before transferring identity trust. Notify the old address of the change/attempt under SECURITY.md and audit the event. Provider linking remains a separate explicit flow; changing email must not merge identities automatically.

## Session and Device Management

Provide an account security page listing active sessions with device/client description, creation time, last activity, coarse location only when safely and lawfully available, and current-session indication. Users can revoke one session or all other sessions; logout invalidates the current session server-side.

Use non-secret session identifiers for management requests and verify ownership on every list/revoke operation. Never return reusable session credentials merely to enable a device list or revocation UI. Device descriptions are display metadata, not proof of identity or factor assurance. Do not collect precise location for this page, and minimize retention of IP/device metadata.

## CSRF and Origin Protection

Cookie-authenticated state-changing requests require suitable SameSite behavior, Origin validation where applicable, and CSRF protection appropriate to the final browser/API topology. CORS is not CSRF protection. Reject unsafe cross-origin credential use, restrict trusted origins and callbacks, and do not disable protections globally to accommodate a mobile client or social callback.

Bearer-only mobile credentials are not automatically attached by browsers like cookies, but still require authorization, TLS, secure storage, and callback protections. Mixed cookie/bearer routes must not accidentally bypass browser protections. Exact domain-specific mechanisms are deferred until topology is known.

## Rate Limiting and Abuse Protection

Protect sign-in, sign-up, verification requests, password-reset requests, TOTP attempts, recovery-code attempts, provider linking, and security-sensitive recovery flows. Combine appropriate account, source, and endpoint controls without easy account-lockout denial of service. Avoid unnecessary account-existence disclosure and avoid logging submitted secrets on failures.

Rate-limit storage, thresholds, and eventual multi-instance coordination are implementation decisions. Redis is not introduced solely to record this requirement.

## Audit and Security Events

Audit password changes/resets, email changes, provider links/unlinks, 2FA enable/disable/reset, recovery-code regeneration/use where appropriate, privileged grants/revocations, ownership transfer, session revocation, and suspicious/repeated authentication failures where appropriate. Record actor/session identifiers, outcome, time, and necessary context with limited personal metadata; notify users of relevant security changes.

Logs, audit records, analytics, error reports, and traces must never contain passwords, session credentials, TOTP seeds/codes, backup codes, reset/verification tokens, or OAuth tokens. Authentication/session evidence must be server-controlled and unavailable for arbitrary client updates.

## Alternatives Considered

### Fully custom authentication

Offers maximum control, but imposes unnecessary security-critical implementation and maintenance burden. Password, session, provider, and recovery flows are easy to implement incorrectly. Use a library for supported mechanisms and own the application policy around it.

### JWT access and refresh tokens

A common API approach useful in some distributed systems. It adds issuance, refresh rotation, replay, and revocation complexity here. Server-side sessions more directly satisfy immediate revocation and device management in the modular monolith. Reconsider only for a concrete requirement.

### Auth.js

Auth.js has a strong web/Next.js ecosystem. This platform's authoritative authentication boundary is the NestJS API serving both browser and Flutter clients, so Better Auth behind that boundary is the accepted fit. This is a boundary/maintenance choice, not a claim that Auth.js cannot support other integrations.

### External managed identity provider

Managed providers can offer operational support and security capabilities that reduce local maintenance. They are not selected initially because of cost, external dependency, control requirements, and the current self-hosted architecture. Reconsider if operational/security needs justify the tradeoff; EU data handling and cost must then be reviewed explicitly.

## Security Considerations

Better Auth does not establish tenant entitlement, permissions, private-content access, or ownership invariants. Application policy must enforce those boundaries even where library authentication succeeds. Session theft remains serious despite immediate revocation support; HttpOnly does not eliminate XSS-driven actions, and opaque credentials still require protection at rest and in transit.

Verify supported APIs and actual installed versions for social-login 2FA, passwordless-account enrollment, explicit linking, session transport, cookie behavior, revocation, and NestJS integration. Keep any community adapter inside the application AuthModule so it can be replaced without rewriting domain authorization; adapter-specific decorators/objects must not spread through application modules. The [Better Auth NestJS integration documentation](https://better-auth.com/docs/integrations/nestjs) identifies its documented adapter as community maintained.

Library freshness and rolling session renewal are not substitutes for the separate application clocks. Cookie caching can delay revocation; this project requires authoritative checks instead. Verify these behaviors against [Better Auth session documentation](https://better-auth.com/docs/concepts/session-management). Review bearer issuance and browser credential exposure against the [bearer transport documentation](https://better-auth.com/docs/plugins/bearer), and linking behavior against [account documentation](https://better-auth.com/docs/concepts/users-accounts).

Where supported extension points cannot enforce policy, keep the affected flow unavailable and resolve the integration gap explicitly. Do not weaken policy or patch undocumented internals silently. No library compatibility or security behavior is claimed tested by this documentation task.

## Testing Requirements

Authentication tests are release-critical. Add positive and negative paths with each flow; privileged functionality cannot ship before assurance tests pass. Cover at least:

- Email/password registration, email verification, valid/invalid password sign-in, and unverified-account restrictions.
- Google and Apple sign-in, provider-response/callback validation, and social-only accounts without local passwords.
- Safe explicit linking/unlinking, rejected automatic email-based merging, wrong-account/replayed linking, and recent-authentication requirements.
- Session creation, fixation/rotation behavior, normal expiration, separate inactivity and absolute limits, and web/mobile policy enforcement.
- Revocation of current/other sessions, wrong-user revocation denial, all-session revocation after password reset, and rejected old mobile credentials.
- HttpOnly/Secure/SameSite/scope behavior and absence of reusable browser credentials in JavaScript-facing responses or session lists.
- Mandatory 2FA for protected standard and custom capabilities, verified enrollment, invalid/replayed TOTP, and incomplete challenge restrictions.
- Email/password, Google, and Apple cannot bypass privileged 2FA; optional enabled 2FA also cannot be silently bypassed through another sign-in path.
- Safe privilege-grant, factor-disable/reset, factor-loss/recovery, and privilege-revocation transitions; recovery-code single-use under concurrency and invalidation after regeneration.
- Fifteen-minute elevated inactivity, eight-hour elevation maximum, five-minute step-up expiration, and critical Primary Owner operations requiring current assurance and authorization.
- CSRF/Origin protections, callback exceptions, unsafe cross-origin credential rejection, and authentication abuse limits.
- Sensitive values absent from logs, audits, errors, and analytics; expected security events/notifications present.

Use controlled clocks rather than real waits. Use real PostgreSQL when session persistence, transaction atomicity, revocation races, or database behavior matters. Test the installed Better Auth/adapter boundary rather than mocking all authentication behavior. Use controlled provider fakes/test environments for automated OAuth flows and approved non-production end-to-end provider checks during integration. No production credentials or bypasses belong in tests. Continue the independent authorization and tenant-isolation coverage in [TESTING.md](../TESTING.md) and ADR 0006.

## Consequences

One authentication boundary and server-side session model simplify revocation, device management, and client consistency while preserving application control. Encapsulation supports replacing an adapter/library and focused Codex-assisted work without distributing provider-specific behavior through domain modules.

Costs include operating authentication storage and secrets, maintaining library updates, database reads for authoritative checks, and implementing assurance/clock/transition policies beyond defaults. Native callback integration and social-only reauthentication require deliberate validation. Mitigate with narrow supported integrations, explicit contracts, least privilege, and release-blocking boundary tests rather than a second general authentication framework.

## Future Review Triggers

Review for unsupported or unmaintained integrations, unresolved security issues, inability to enforce approved policy through supported APIs, measured session-storage limitations, or fundamentally changed deployment/client requirements. A managed provider or different session architecture may be reconsidered with evidence and explicit approval. Newer libraries, synthetic benchmarks, or convenience alone do not justify weakening revocation, tenant authorization, mandatory 2FA, or step-up policy.

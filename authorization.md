# Authorization Matrix — social-middleware

_Last updated: 2026-09-09 (feat/sep9-middleware, after auth fixes)_

## Auth model

Single flat model — no RBAC/roles/scopes. Access = valid `app_session` cookie
(JWT, `JWT_SECRET`, 4h TTL, Redis `jti` blacklist) verified by `SessionAuthGuard`
(src/auth/session-auth.guard.ts:16), attaching `{sub (BCSC card ID), email, name,
userId (Mongo _id)}` to `req.user`. Data isolation via per-call ownership scoping.
External form-builder routes use one-time **form access tokens** (UUID, 30-min
expiry default, `FORM_ACCESS_TOKEN_EXPIRY_MINUTES`). `USE_KONG_OIDC` (default true)
selects Kong `X-Userinfo` vs direct BCSC OAuth for the login flow.

**Legend:** `S` = SessionAuthGuard · `S+own` = session + caller-ownership check ·
`T` = form-access token · `P` = public · `DEV` = mounted only when
`NODE_ENV` ∈ {dev, development, local}

## Auth — auth.controller.ts

| Route                   | Auth | Additional checks                   | Notes                                                                                                 |
| ----------------------- | ---- | ----------------------------------- | ----------------------------------------------------------------------------------------------------- |
| GET /auth/login         | P    | —                                   | Kong mode trusts `X-Userinfo` header (spoofable if reachable without gateway)                         |
| GET/POST /auth/callback | P    | —                                   | POST variant takes `code`+`redirect_uri` from body                                                    |
| GET /auth/status        | P    | inline `jwt.verify` of cookie → 401 |                                                                                                       |
| GET /auth/logout        | P    | —                                   |                                                                                                       |
| GET /auth/profile       | S    | user doc must exist (404)           | `TEST_RESOURCE_CASE === 'true'` gates `resource_case_active_date` + `non_key_player_caregiver` fields |

## Application packages — application-package.controller.ts (class-level S)

| Route                                                                                                                                                                                                                   | Auth  | Additional checks                                     |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- | ----------------------------------------------------- |
| POST/GET/PATCH/DELETE /application-package(/:id)                                                                                                                                                                        | S+own | service scoping by `userId` (404 if not owner)        |
| GET …/:id/application-form                                                                                                                                                                                              | S+own | —                                                     |
| POST …/submit, /request-info-session, /save-referral-contact, /lock-application, /submit-documents-to-icm, /submit-training-certificates, /upload-medical-assessments, /validate-household, /in-service-training/submit | S+own | —                                                     |
| POST /application-package/access-code/redeem                                                                                                                                                                            | S     | code must match session user's lastName + dateOfBirth |

## Attachments — attachments.controller.ts (class-level S)

| Route                                     | Auth  | Additional checks                            |
| ----------------------------------------- | ----- | -------------------------------------------- |
| POST /attachments                         | S     | scoped to `userId`                           |
| POST/GET /attachments/in-service-training | S     | requires `user.resource_case_id` else 400    |
| GET …/application-package/:id             | S+own | userId-scoped                                |
| GET …/household-member/:id                | S+own | `verifyUserOwnsHouseholdMemberPackage` → 403 |
| GET/DELETE /attachments/:id               | S+own | `findByIdAndUser`/`delete(id, userId)` → 404 |

## Forms (external form-builder) — forms.controller.ts

| Route                                     | Auth | Additional checks                                   | Notes                               |
| ----------------------------------------- | ---- | --------------------------------------------------- | ----------------------------------- |
| POST /forms/validateTokenAndGetParameters | T    | token lookup + expiry → 404/400                     |                                     |
| POST /forms/validateTokenAndGetSavedJson  | T    | ⚠ expiry check disabled (accepted risk)             | stale tokens return saved form data |
| POST /forms/tombstone-data                | T    | token must be most recent for the form; returns PII |                                     |

GET /forms/token — **removed 2026-09-09** (was unowned duplicate); token minting
is exclusively via GET /application-forms/token.

## Application forms — application-form.controller.ts (no class guard)

| Route                                          | Auth                                           | Additional checks                    |
| ---------------------------------------------- | ---------------------------------------------- | ------------------------------------ |
| GET /application-forms/token                   | S+own                                          | `confirmOwnership` → 401             |
| GET /application-forms/:id                     | S+own                                          | `confirmOwnership` → 401             |
| POST /application-forms/submit, /saveDraft     | T                                              | token lookup; no expiry on this path |
| GET /application-forms                         | S (in-method `extractUserIdFromRequest` → 401) | caller's forms only                  |
| GET …/household/:memberId                      | S+own                                          | `verifyHouseholdMemberAccess` → 401  |
| POST …/:id/clone, /submit-to-icm; DELETE …/:id | S+own                                          | `confirmOwnership` → 401             |

## Household — household.controller.ts (class-level S, base application-package/:pkgId/household-members)

| Route                                                                | Auth  | Additional checks                                                                                            | Notes                                                           |
| -------------------------------------------------------------------- | ----- | ------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------- |
| POST/GET list, GET/PATCH/DELETE :memberId, GET :memberId/access-code | S+own | `verifyUserOwnsPackage` / `verifyUserOwnsHouseholdMemberPackage` → 401                                       | PATCH: member must not have redeemed code / submitted screening |
| POST :memberId/confirm-screening-package                             | S     | member must be the caller themself → 401                                                                     |                                                                 |
| POST :memberId/access-code/resend                                    | S+own | in-method rate limit (cooldown + daily cap) — only rate limiting in the app                                  |                                                                 |
| POST :memberId/mark-screening-documents-attached                     | S+own | caller must be package owner; **blocked for Spouse / Common law / Partner** (co-applicants submit their own) | fixed 2026-09-09 (was: no caller-to-member check)               |

## Household access — household-access.controller.ts

| Route                                 | Auth | Additional checks                      |
| ------------------------------------- | ---- | -------------------------------------- |
| POST /household/access-code/associate | S    | code must match lastName + dateOfBirth |
| GET /household/members                | S    | returns caller's members only          |

## Admin / infrastructure surfaces

| Surface                                                                   | Auth                    | Status                                                                                        |
| ------------------------------------------------------------------------- | ----------------------- | --------------------------------------------------------------------------------------------- |
| /admin/queues (Bull Board)                                                | DEV-gated               | fixed 2026-09-09: module + mount only in development/local                                    |
| /dev-tools/\* (clear-user-data, reset-application-package, trigger-stage) | DEV-gated + runtime 403 | fixed 2026-09-09: registration dev-only; `isDev()` check now actually invokes (was no-op bug) |
| /api (Swagger)                                                            | NONE                    | unauthenticated docs (gateway-protected); accepted risk                                       |
| GET /health                                                               | P                       | intentional                                                                                   |
| /siebel/auth/\*                                                           | n/a                     | controller registration commented out — dead code                                             |

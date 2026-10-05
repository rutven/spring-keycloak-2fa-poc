# PoC: Keycloak Step-Up Authentication for Sensitive REST API Operations

## Goal

Implement a **Standalone PoC** for step-up authentication using Keycloak and Spring Boot. Establish an independent, reproducible baseline.

Select and document explicit versions, prerequisites, and build/run tooling. Provide reproducible Keycloak realm configuration and application configuration so developers can run the demonstration without access to existing environments.

Sensitive REST operations must require stronger authentication for **human/UI users**, while explicitly trusted **machine-to-machine integration clients** using OAuth2 `client_credentials` may invoke the same operations without OTP.

Business authorization must remain the same for both types of callers.

The authorization rule should conceptually be:

```text
has required business permission

AND

(
    caller is an explicitly trusted service-account client
    OR
    user authentication satisfies required LoA/ACR
    AND valid single-use step-up authorization is consumed at execution start
)
```

Keycloak must remain responsible for authentication and MFA verification. The PoC API must never receive or validate OTP values itself.

---

## Expected UI flow

Include a minimal browser UI demonstrating normal login, redirect to Keycloak
for OTP verification, and sensitive execution. The UI must also support manual
checks of session binding, pending-authorization replacement across tabs, and
logout invalidation. OTP entry remains exclusively in Keycloak.

Serve the minimal browser UI and API from the same application and origin.
A separately deployed frontend is outside the PoC scope.

Browser API calls use Keycloak bearer access tokens, as integration API calls
do. Do not replace API bearer authentication with an application session cookie.
The backend still owns the step-up challenge, callback, and code exchange;
callback correlation does not independently authorize sensitive API calls.
Keep browser tokens and authorization handles only in page memory, without
persistent browser storage. Reloading the page loses that state and requires
obtaining it again. Use a popup for OTP so the main page retains its in-memory
state during verification. Delivery after the backend code exchange remains to
be selected. Losing a handle in the browser does not itself consume or revoke
the corresponding API authorization; a new successful flow can replace it.

Normal login also uses a backend-owned authorization-code flow in a popup.
Use the same callback pattern with distinct, correlated flow purposes: normal
login obtains tokens but does not issue single-use step-up authorization;
successful step-up additionally issues authorization for the requested operation.

After successful callback validation and authorization issuance, require an
explicit **Execute** click to invoke the sensitive operation. Do not execute it
automatically from the callback. The UI should expose whether authorization is
ready and its expiry so issuance and consumption can be demonstrated separately.

Allow the user to start **Verify OTP** directly, without first submitting a
sensitive request that is rejected. Check business permission before starting
the flow; after successful issuance, enable the explicit Execute action. The
API must still reject direct sensitive calls without valid authorization. The
rejection-first flow below is an alternative demonstration, not a prerequisite.

```text
User
  │
  │ normal authentication
  ▼
Keycloak
  │
  │ access token (LoA 1)
  ▼
UI
  │
  │ regular REST calls
  ▼
PoC API ──────────────► allowed

UI
  │
  │ sensitive operation
  ▼
PoC API
  │
  └──────────────► rejected: step-up required

UI
  │
  │ request stronger authentication
  ▼
Keycloak
  │
  │ built-in OTP/TOTP
  │
  ▼
access token (LoA 2)
  │
  ▼
PoC API establishes single-use authorization
  │ bound to user, session, and operation
  ▼
UI requests sensitive execution
  │
  │ sensitive operation
  ▼
PoC API checks permission and active session
  │ atomically consumes authorization at execution start
  ▼
allowed once
```

The built-in Keycloak OTP/TOTP mechanism should be reused.

Do not implement OTP verification in the PoC API.

Each human-user invocation of a sensitive operation requires a fresh OTP verification.
An earlier successful OTP verification must not authorize a subsequent invocation,
even if the caller still holds a valid LoA 2 access token. Reusing stronger
authentication for a time window is not sufficient. Each successful OTP verification
may authorize at most one sensitive invocation: if two concurrent requests attempt
to reuse the same authorization, at most one may be allowed.

Consume the single-use step-up authorization when execution begins, even if
execution subsequently fails. A retry requires fresh OTP verification. Requests
rejected before execution, including requests missing the required business
permission, do not consume the authorization. The enforcement mechanism remains
to be decided.

Require the operation's business permission before starting its OTP flow, and
check that permission again before sensitive execution begins. Successful OTP
verification never grants business permission. Use business permissions from the
validated access token until it expires; permission revocation need not take
effect immediately while that token remains valid. This does not relax the
required session-status checks and logout-notification invalidation described below.

Bind the single-use step-up authorization to the requested sensitive method.
Authorization obtained for one method must not authorize a different method.
Identify the sensitive operation using a stable operation identifier declared
by the protected method, rather than its Java method name or HTTP route. Use this
identifier for authorization binding and the per-user pending-authorization limit.
Do not bind authorization to individual request data in this PoC. Request data
may change after OTP verification provided the operation identifier stays the
same; normal business authorization and validation still apply at execution.
Payment-specific dynamic linking remains outside the current PoC scope.

Unused single-use step-up authorization expires two minutes after successful OTP
verification by default. Make this lifetime configurable. Expiry requires fresh
OTP verification; requests rejected before execution do not restart or extend
the lifetime.

If the selected built-in Keycloak flow cannot provide a trustworthy OTP-completion
timestamp, measure the configured lifetime from when the API initiates the
correlated OTP flow instead. Time spent completing OTP reduces the remaining
usable lifetime. Callback arrival must never restart the clock; reject issuance
if that deadline has already passed. Document which time origin the baseline uses.

Each human-user sensitive invocation requires a new OTP verification, and the
same OTP code must not authorize multiple invocations, including within the same
TOTP interval. Keycloak must enforce OTP replay prevention; the API must never
receive or compare OTP codes. Verify support against the selected Keycloak
version and realm configuration before implementation. This requirement is
separate from preventing reuse of a single-use step-up authorization.

Allow at most one pending, unused single-use step-up authorization per user and
sensitive method, including across browser tabs. Behaviour when another OTP
verification succeeds while an authorization is pending is atomic replacement:
the old authorization becomes unusable, and the new authorization receives its
own configured lifetime (two minutes by default), measured from the new successful
OTP verification or the new flow's initiation time under the fallback above.
Replacement and consumption must preserve the single-use rule
under concurrent requests.

Starting, failing, or cancelling a new OTP flow does not invalidate or extend an
existing pending authorization. Replace it only after the new OTP verification
succeeds, its correlated result is validated, and issuance is otherwise eligible
(including the applicable expiry deadline). Until then, the existing authorization
remains usable subject to its original expiry and all normal execution checks.

Track OTP flow initiation order per user and stable operation identifier. Once
a newer flow successfully issues authorization, an older flow's delayed callback
must not issue or replace authorization, even if the newer authorization has
since been consumed or expired. Enforce this ordering atomically with issuance.
Merely starting a newer flow does not invalidate an existing authorization.

Bind each single-use step-up authorization to the login session that completed
OTP verification. Another session belonging to the same user must not use it.
The one-pending-authorization limit remains per user and sensitive method across
sessions; a new successful verification in another session replaces the previous
pending authorization under the same atomic replacement rule.

A refreshed, validated access token may use an existing pending authorization
when it represents the same user and login session and carries the required
business permission. Token refresh does not extend the authorization's expiry
or restore a consumed authorization.

Check the bound session's status immediately before sensitive execution, even
if its access token has not expired. Reject inactive or unknown status without
using cached positive results. Validated logout notifications invalidate unused
authorizations for the affected session; local invalidation and consumption must
be serialized. The race where remote logout occurs after the final active-session
check but before local execution is explicitly accepted. This replaces the
earlier strict immediate-logout guarantee. Verify the session-status and
logout-notification mechanisms against the selected Keycloak baseline.

If the API cannot establish that the bound session is still active, reject the
sensitive invocation before execution (fail closed). Leave the authorization
unconsumed and retain its original expiry; an unavailable session-status check
must not extend the authorization lifetime.

For this PoC, run a single API instance and keep pending single-use step-up
authorizations in memory. Application restart invalidates all pending
authorizations and requires fresh OTP verification. Multi-instance coordination
and durable authorization storage are outside scope. In-memory replacement and
consumption must still be atomic under concurrent requests.

The API manages a separate single-use step-up authorization in addition to
validating the access token. It enforces operation binding, session binding,
expiry, atomic replacement, and atomic consumption. Keycloak remains responsible
for OTP verification. A valid LoA 2 token alone must not grant sensitive execution.
The API owns the step-up flow: it creates a challenge bound to the initiating
user, login session, and stable operation identifier, and handles the Keycloak
callback and authorization-code exchange. Issue authorization only after
validating the correlated authentication result and completing the challenge
atomically once. A UI-submitted LoA 2 token alone is not issuance evidence.
Replayed callbacks or authentication results must not issue another authorization,
including after consumption, replacement, expiry, or application restart.
The exact validation and correlation configuration requires baseline verification.

If the step-up result represents a different user or login session from the one
that initiated the challenge, reject it without issuing authorization or
replacing the main page's tokens. Account switching requires an explicit normal
login flow. A rejected mismatch does not replace existing pending authorization.

Human sensitive requests explicitly present an opaque single-use authorization
handle alongside the access token. The API resolves the handle to its own state
and checks the authenticated user, login session, stable operation identifier,
expiry, and pending status before atomic consumption. The handle alone does not
authenticate the caller or grant business permission. Replacement invalidates
the old handle; a request carrying it must not consume the newer authorization.
Carry this application-issued handle in the dedicated `Step-Up-Authorization`
HTTP request header. This is an application protocol choice, not a Keycloak
claim or standard OIDC header. The header cannot establish identity, integration
trust, or business permission; it references API state bound to validated identity.

Use only built-in Keycloak capabilities; custom Keycloak extensions remain out
of scope. If research shows that built-in capabilities cannot satisfy fresh OTP
verification, OTP replay prevention, the agreed logout checks, or another
agreed requirement, document the gap and revisit the requirement explicitly
before implementation. Do not silently weaken it or introduce a custom provider.

Explicitly trusted service-account integration clients remain exempt from this
per-invocation OTP requirement. They must still have the required business
permission; the exemption does not extend to untrusted service accounts.

---

## Expected integration flow

Integrations use OAuth2 `client_credentials` through Keycloak service accounts.

```text
Integration
    │
    │ client_id + client authentication
    ▼
Keycloak
    │
    │ client_credentials
    ▼
service-account access token
    │
    ▼
PoC API
    │
    │ sensitive operation
    ▼
allowed
```

No OTP/step-up authentication is required because there is no human user involved.

However, **not every service-account/client_credentials token should automatically bypass MFA**.

Only explicitly configured trusted integration clients may bypass the step-up requirement.

---

## Investigation

Before application implementation, select and document the baseline versions of Keycloak, Spring Boot, Spring Security, and the language/runtime, along with build and local-run tooling. Research the selected versions and verify authentication behaviour against the PoC realm configuration.

Determine:

- Selected baseline versions.
- Authentication flows required for the PoC realm.
- Built-in OTP/TOTP configuration required for the PoC.
- How UI users authenticate.
- How integration clients authenticate.
- How the PoC API will validate Keycloak JWTs.
- PoC business roles/authorities and authorization design.
- Claims present in UI access tokens.
- Claims present in `client_credentials` access tokens.
- Whether `acr`, `amr`, `client_id`, `azp`, or another suitable standard/Keycloak claim can reliably distinguish the authentication context.
- Keycloak-native support for Authentication Context / Level of Authentication.
- How the UI should request LoA 2.
- Whether Authentication Flow changes are required.

Prefer standard OIDC and built-in Keycloak functionality over custom claims, custom authenticators, or Keycloak extensions.

Document the findings before implementing the PoC.

---

## Authentication levels

Use at least two authentication levels for human users.

Conceptually:

```text
LoA 1
    normal user authentication

LoA 2
    normal authentication + built-in OTP/TOTP
```

Prefer the OIDC `acr` claim and Keycloak's native Authentication Context / Level of Authentication functionality.

For example:

```json
{
  "sub": "...",
  "acr": "1"
}
```

versus:

```json
{
  "sub": "...",
  "acr": "2"
}
```

Do not assume these exact values without verifying how the selected Keycloak version and PoC realm configuration represents LoA.

The API must verify that MFA was actually performed for the authentication represented by the token.

Merely having OTP configured on the Keycloak user is not sufficient.

---

## Trusted integration clients

Introduce configuration containing the Keycloak clients that may bypass the step-up requirement.

For example:

```yaml
security:
  step-up:
    trusted-clients:
      - some-integration-client
      - another-integration-client
```

Document the configuration naming conventions established for the Standalone PoC.

Do not hardcode client IDs in authorization code.

A service account must be considered trusted only when:

1. the token can reliably be identified as representing an OAuth2 service-account/client principal;
2. its Keycloak client identity can be reliably determined from validated JWT claims;
3. that client is present in the configured trusted-client allowlist.

Do **not** infer that a token is trusted merely because:

- `acr` is absent;
- `sub` looks different;
- a custom HTTP header says the caller is an integration;
- the caller claims a particular client ID.

All decisions must be based on validated JWT claims issued by Keycloak.

---

## Business authorization

UI users and integrations should use the **same business permissions**.

For example, if approving an operation requires:

```text
PAYMENT_APPROVE
```

then both:

```text
UI user → PAYMENT_APPROVE
integration service account → PAYMENT_APPROVE
```

must have that permission.

Do not introduce a special business permission such as:

```text
PAYMENT_APPROVE_WITHOUT_MFA
```

just to bypass step-up authentication.

Keep these concerns separate:

```text
Business authorization:
    Is this principal allowed to perform the operation?

Authentication assurance:
    Does this invocation require stronger human authentication?
```

The service-account exception applies only to the second question.

---

## Payment demonstration API

Use seeded fictional payments to demonstrate realistic business authorization
and state transitions. No funds move and no external payment system is involved.

| Endpoint | Required permission | Human step-up | Stable operation identifier |
|---|---|---|---|
| `GET /api/payments` | `PAYMENT_READ` | No | — |
| `GET /api/payments/{id}` | `PAYMENT_READ` | No | — |
| `POST /api/payments/{id}/approve` | `PAYMENT_APPROVE` | Single-use OTP authorization | `payment.approve` |
| `POST /api/payments/{id}/reject` | `PAYMENT_APPROVE` | Single-use OTP authorization | `payment.reject` |

Approve changes a payment from `PENDING` to `APPROVED`; reject changes it from
`PENDING` to `REJECTED`. Both sensitive operations use the same business
permission for humans and integrations, with only explicitly trusted integrations
exempt from step-up. Demonstrate that approval authorization cannot authorize
rejection. Success returns the payment identifier and resulting status.

Authorization binds to `payment.approve` or `payment.reject`, not the payment ID,
amount, beneficiary, or request payload. A user may choose a different pending
payment after OTP verification for the same operation. State this limitation in
the UI and documentation.

Validate payment state before sensitive execution begins. Approving or rejecting
an already `APPROVED` or `REJECTED` payment returns `409 Conflict` without
consuming single-use authorization. Serialize the payment state check, authorization
consumption, and transition so concurrent approve/reject requests allow at most
one transition from `PENDING`; a losing request must not consume authorization.
Execution failure after consumption still requires fresh OTP for retry.

Conceptually:

```text
required business permission

AND

(
    trustedServiceAccount
    OR
    authenticationLevel >= 2
    AND valid single-use step-up authorization is consumed at execution start
)
```

---

## Spring Security design

Implement the policy as a reusable Spring Security authorization component.

Do not put ad-hoc JWT parsing into controllers.

The resulting application code should preferably be expressive, for example conceptually:

```kotlin
@PreAuthorize("hasAuthority('PAYMENT_APPROVE') && satisfiesStepUp(2)")
```

or:

```kotlin
@RequiresStepUp(level = 2)
```

or an equivalent approach that fits the selected Spring Security version and PoC architecture.

Do not introduce either API literally if Spring Security provides a cleaner mechanism.

The reusable policy should implement:

```text
satisfiesStepUp(requiredLoA):

    if trusted service-account client:
        true

    else:
        authentication ACR/LoA >= requiredLoA
        AND valid single-use step-up authorization for this user, session, and operation
```

Business permission checking remains separate.

The policy check alone does not consume authorization. Consumption must be
atomic at execution start after pre-execution checks pass; annotation examples
above are illustrative and must accommodate this lifecycle in the final design.

---

## Failure behaviour

Differentiate authentication/authorization failures appropriately.

At minimum consider:

```text
Unauthenticated
    → 401

Authenticated but missing business permission
    → 403

UI user has business permission but insufficient LoA
    → reject and provide enough information for the client
      to know that step-up authentication is required
```

Investigate the most appropriate standards-based response for the last case rather than inventing a custom protocol unnecessarily.

Document the selected behaviour.

---

## Test scenarios

### 1. Normal UI authentication

```text
normal login
→ LoA 1 token
→ regular API
→ allowed
```

---

### 2. Sensitive operation without step-up

```text
normal login
→ LoA 1 token
→ user has required business permission
→ sensitive API
→ rejected because LoA is insufficient
```

---

### 3. Sensitive operation after OTP

```text
normal login
→ request LoA 2
→ Keycloak OTP/TOTP challenge
→ successful OTP
→ LoA 2 token
→ API establishes a session-bound authorization for the requested operation
→ sensitive API
→ authorization consumed at execution start
→ allowed once
→ reuse rejected even while the LoA 2 token remains valid
```

---

### 4. Trusted integration

```text
trusted integration client
→ client_credentials
→ service-account token
→ service account has required business permission
→ client is present in trusted-client allowlist
→ sensitive API
→ allowed without OTP
```

---

### 5. Untrusted integration

```text
different service-account client
→ client_credentials
→ service account has required business permission
→ client NOT present in trusted-client allowlist
→ sensitive API
→ rejected
```

It must not automatically receive the MFA bypass simply because it uses `client_credentials`.

---

### 6. Integration without business permission

```text
trusted integration
→ client_credentials
→ client is allowlisted
→ required business permission is missing
→ sensitive API
→ rejected
```

Being trusted for the step-up exemption must not bypass business authorization.

---

### 7. User with OTP configured but LoA 1 token

```text
user has OTP configured in Keycloak
→ normal LoA 1 authentication
→ sensitive API
→ rejected
```

This proves that OTP enrollment alone does not satisfy the requirement.

---

### 8. Missing/malformed authentication context

For a human/user token:

```text
ACR/LoA missing or invalid
→ sensitive API
→ fail closed
```

Do not interpret missing ACR as an integration token.

---

## Automated tests

Add automated tests covering at least:

- unauthenticated request;
- UI user with LoA 1;
- UI user with LoA 2;
- user with malformed/missing ACR;
- trusted service account;
- untrusted service account;
- trusted service account without required business permission;
- UI user without required business permission.

Also cover the agreed single-use authorization lifecycle:

- LoA 2 token without a pending authorization is rejected;
- concurrent reuse allows at most one execution;
- execution failure consumes authorization and retry requires fresh OTP;
- rejection before execution leaves authorization unconsumed without extending expiry;
- wrong operation or wrong session is rejected;
- payload changes for the same operation are permitted subject to normal validation;
- configurable expiry defaults to two minutes from OTP verification, with flow-initiation time as the agreed conservative fallback;
- delayed or expired callbacks cannot restart the clock or issue expired authorization;
- a stale authorization handle cannot consume a newer replacement;
- failed or cancelled OTP flows preserve existing authorization without extending it;
- an older callback cannot issue authorization after a newer flow successfully issued, even after consumption or expiry;
- successful OTP callback does not execute the sensitive operation before an explicit Execute click;
- new verification atomically replaces the previous authorization across tabs and sessions;
- inactive session status or a validated logout notification blocks unused authorization, including with an unexpired token;
- the accepted race between the final session-status check and execution is documented and tested;
- unknown session status rejects execution without consumption;
- refreshed tokens for the same user and session can use pending authorization without extending it;
- application restart invalidates all pending authorizations;
- business permission is checked before the OTP flow and before execution using validated token permissions.

Payment demonstration tests must also cover read access with `PAYMENT_READ`,
both permitted transitions from `PENDING`, operation-binding mismatch between
approval and rejection, `409 Conflict` on finalized payments without consumption,
and concurrent approve/reject allowing at most one transition without consuming
the losing caller's authorization.

Keycloak feasibility/E2E checks must demonstrate a new OTP verification for each
authorization, rejection of OTP code reuse within the same TOTP interval, and
that replaying verification evidence cannot recreate an authorization after
consumption, replacement, expiry, or application restart.

Keep Keycloak E2E testing lightweight for this PoC.

Separate unit tests of the authorization policy from integration/E2E tests where practical.

---

## Keycloak configuration

Add or document the minimum Keycloak configuration necessary for:

- LoA 1 normal authentication;
- LoA 2 authentication using the built-in Keycloak OTP mechanism;
- requesting LoA 2 from the UI;
- exposing the resulting authentication context in the access token;
- `client_credentials` authentication for integrations;
- service-account business roles.

Provide version-controlled realm configuration or setup automation that reproduces the required Keycloak baseline. Document any unavoidable manual steps, including OTP enrollment. Never commit credentials, OTP secrets, or live tokens.

Avoid undocumented manual configuration.

---

## Documentation

Add short developer documentation containing:

1. selected architecture, explicit baseline versions, prerequisites, and exact build/run commands;
2. Keycloak configuration;
3. normal UI authentication;
4. triggering step-up authentication;
5. completing OTP;
6. obtaining the LoA 2 token;
7. obtaining a `client_credentials` token;
8. configuring trusted integration clients;
9. example calls to the sensitive endpoint;
10. example decoded JWT claims for:
    - normal UI token;
    - stepped-up UI token;
    - integration/service-account token.

Include `curl` examples where practical.

---

## Security constraints

- The PoC API must never receive OTP/TOTP values.
- The PoC API must never store OTP secrets.
- Do not implement custom OTP validation.
- Do not make MFA mandatory for all API calls.
- Use built-in Keycloak MFA.
- Prefer OIDC standards and built-in Keycloak functionality.
- Do not trust caller-supplied HTTP headers for identifying integrations.
- Do not automatically exempt all `client_credentials` clients.
- Trusted integration clients must be explicitly configured.
- An MFA exemption must never bypass normal business authorization.
- Missing or ambiguous authentication information must fail closed.
- Keep the implementation small and isolated because this is a PoC.

---

## Out of scope

Do not implement yet:

- OTP confirmation bound to an individual payment;
- actual funds movement or external payment-system integration;
- dynamic linking of OTP to payment ID, amount, beneficiary, etc.;
- PSD2/SCA transaction signing;
- custom Keycloak providers;
- production-grade management UI for trusted integrations.

These can be addressed separately after the basic step-up mechanism has been validated.

---

## Deliverables

Provide:

- working payment read, approve, and reject demonstration endpoints;
- minimal browser UI for login, Keycloak OTP, sensitive execution, and lifecycle checks;
- reusable Spring Security step-up authorization mechanism;
- configurable trusted-integration allowlist;
- reproducible Keycloak realm and application configuration;
- explicit baseline versions, prerequisites, and build/run instructions;
- automated authorization tests;
- manual E2E test instructions;
- example tokens/claims;
- short explanation of architectural decisions and limitations.

**Before application implementation, document the selected baseline versions and configuration and present a short implementation plan. Verify claim names and Keycloak behaviour against the selected version and reproducible PoC realm;**

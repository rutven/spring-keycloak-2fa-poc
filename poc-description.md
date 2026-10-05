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
)
```

Keycloak must remain responsible for authentication and MFA verification. The PoC API must never receive or validate OTP values itself.

---

## Expected UI flow

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
PoC API
  │
  │ sensitive operation
  ▼
allowed
```

The built-in Keycloak OTP/TOTP mechanism should be reused.

Do not implement OTP verification in the PoC API.

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

## PoC endpoint

Add a minimal test endpoint such as:

```http
POST /api/poc/sensitive-action
Authorization: Bearer <access-token>
```

No real business operation is required.

On success:

```json
{
  "status": "executed"
}
```

The endpoint should require a PoC business permission plus the step-up policy.

Conceptually:

```text
required business permission

AND

(
    trustedServiceAccount
    OR
    authenticationLevel >= 2
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
```

Business permission checking remains separate.

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
→ sensitive API
→ allowed
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

- payment-specific confirmation;
- dynamic linking of OTP to payment ID, amount, beneficiary, etc.;
- PSD2/SCA transaction signing;
- custom Keycloak providers;
- production-grade management UI for trusted integrations.

These can be addressed separately after the basic step-up mechanism has been validated.

---

## Deliverables

Provide:

- working PoC sensitive endpoint;
- reusable Spring Security step-up authorization mechanism;
- configurable trusted-integration allowlist;
- reproducible Keycloak realm and application configuration;
- explicit baseline versions, prerequisites, and build/run instructions;
- automated authorization tests;
- manual E2E test instructions;
- example tokens/claims;
- short explanation of architectural decisions and limitations.

**Before application implementation, document the selected baseline versions and configuration and present a short implementation plan. Verify claim names and Keycloak behaviour against the selected version and reproducible PoC realm;**

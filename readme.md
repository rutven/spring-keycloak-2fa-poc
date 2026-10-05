# PoC: Keycloak Step-Up Authentication for Sensitive REST API Operations

Implement a proof of concept for step-up authentication using the existing Keycloak 2FA/MFA configuration.

## Goal

Demonstrate that a sensitive REST API operation can require a stronger authentication level than normal authenticated API requests.

Expected flow:

1. User authenticates normally through Keycloak.
2. User receives an access token sufficient for regular API operations.
3. User calls a normal endpoint - it responds.
4. User calls a sensitive endpoint.
5. The endpoint requires step-up authentication.
6. User performs the existing Keycloak second factor (prefer the currently configured TOTP/OTP mechanism).
7. Keycloak issues an access token indicating the stronger authentication context.
8. The sensitive endpoint accepts the request with the stepped-up token.

Do not implement OTP verification inside the CBS application. Keycloak must remain responsible for authentication and MFA verification.

## Investigation

Before implementation, inspect the existing Keycloak and Spring Security configuration and determine:

- Keycloak version and current authentication flow.
- How 2FA/TOTP is currently configured.
- How the application obtains and validates access tokens.
- Whether Keycloak already exposes useful `acr`, `amr`, authentication method, or authentication-level information in the access token.
- The most appropriate Keycloak-native mechanism for requesting step-up authentication.
- Whether an Authentication Flow / LoA configuration is required.

Prefer standard OIDC mechanisms and Keycloak-supported features over custom token claims or custom Keycloak extensions.

Document briefly what was found and the selected approach.

## PoC API

Add a minimal test endpoint, for example:

```http
POST /api/poc/sensitive-action
Authorization: Bearer <access-token>
```

The endpoint does not need to perform any real business operation.

Return:

```json
{
  "status": "executed"
}
```

when the required authentication level is satisfied.

If the supplied access token represents only normal authentication, reject the operation with an appropriate HTTP response.

Do not require MFA globally. Existing endpoints must continue working with normal authentication.

## Authentication levels

Use two distinguishable authentication levels:

```text
Normal authentication
    → regular access token
    → regular APIs allowed
    → sensitive API rejected

Step-up authentication / MFA
    → stronger authentication context
    → sensitive API allowed
```

Prefer using the OIDC `acr` claim / Keycloak Authentication Context / Level of Authentication if supported by the current Keycloak setup.

Do not simply check whether the user has OTP configured. The API must verify that stronger authentication was actually performed for the authentication session represented by the token.

## Spring Security

Implement the check using the existing Spring Security infrastructure.

Prefer a reusable mechanism rather than putting ad-hoc JWT parsing directly into the controller.

For example, the final usage could conceptually look like:

```kotlin
@PreAuthorize("hasAuthenticationLevel(2)")
@PostMapping("/api/poc/sensitive-action")
fun sensitiveAction(): ...
```

or an equivalent Spring Security authorization mechanism appropriate for the existing project.

Do not introduce this exact API if Spring Security provides a cleaner solution.

## Keycloak configuration

Add/document the minimum Keycloak configuration required for the PoC.

The configuration must allow the client to explicitly request stronger authentication when necessary.

Where possible, keep configuration reproducible through the project's existing Keycloak realm/configuration mechanism rather than requiring undocumented manual changes.

## Test scenario

Demonstrate at least these cases:

### Case 1 — normal authentication

```text
normal login
→ access token
→ regular API: allowed
→ sensitive API: rejected
```

### Case 2 — step-up authentication

```text
normal login
→ request stronger authentication from Keycloak
→ complete TOTP/2FA
→ obtain stepped-up access token
→ sensitive API: allowed
```

### Case 3 — invalid assumption

Verify that merely having TOTP configured for the user does not grant access to the sensitive endpoint unless the token represents the required authentication level.

## Tests

Add automated tests for the authorization logic where practical.

At minimum cover:

- missing authentication;
- normal authentication level;
- required step-up authentication level;
- malformed/missing relevant token claim.

Keep Keycloak integration/E2E testing lightweight for this PoC.

## Documentation

Add a short README or developer documentation containing:

1. selected architecture;
2. required Keycloak configuration;
3. how to obtain a normal token;
4. how to trigger step-up authentication;
5. how to obtain the stepped-up token;
6. example `curl` requests against the PoC endpoint;
7. example decoded JWTs showing the relevant difference (`acr`, `amr`, or selected equivalent).

Include a simple sequence diagram of the final flow:

```text
Client          CBS API             Keycloak
  |                |                    |
  |-- normal auth --------------------->|
  |<------------- token ----------------|
  |                |                    |
  |-- sensitive -->|                    |
  |<-- step-up required                 |
  |                                     |
  |-- step-up auth + TOTP ------------->|
  |<---------- stronger token -----------|
  |                                     |
  |-- sensitive -->|                    |
  |<-- success ----|                    |
```

## Constraints

- Do not implement or store OTP secrets in CBS.
- Do not send OTP values to CBS REST endpoints.
- Do not build a custom MFA implementation.
- Do not make MFA mandatory for all API calls.
- Reuse the existing Keycloak 2FA mechanism.
- Prefer OIDC standards and built-in Keycloak capabilities.
- Keep the implementation small and isolated: this is a PoC, not the final payment-approval implementation.
- Do not introduce payment-specific logic yet.

## Deliverable

Provide:

- working PoC endpoint;
- required Spring Security changes;
- required Keycloak configuration;
- automated authorization tests;
- manual end-to-end test instructions;
- short explanation of the selected approach and any Keycloak limitations discovered.

Before making changes, inspect the existing implementation and present a short implementation plan. Then proceed with the PoC.

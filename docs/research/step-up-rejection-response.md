# Step-up rejection response standard

Research for issue #3 (map #1). Input to a later grilling decision; nothing here is decided. Not runtime-verified.

Terms per `CONTEXT.md`. Spec: `poc-description.md` "Failure behaviour" ("investigate ... standards-based response ... rather than inventing a custom protocol").

## 1. RFC 9470 facts

Source: <https://www.rfc-editor.org/rfc/rfc9470.html> (sections 3-7).

- Challenge = `WWW-Authenticate: Bearer error="insufficient_user_authentication"` on a resource server response. New error code (extends RFC 6750 Bearer errors).
- Optional params: `acr_values` (space-separated ACR values, in order of preference), `max_age` (allowed seconds since last active authentication event).
- Status: examples in sec. 3 show `401 Unauthorized`. Text does not discuss 401 vs 403 (no normative rule found). Contrast RFC 6750 `insufficient_scope` = 403.
- `error_description` = optional human-readable (RFC 6750 sec. 3). RFC 9470 examples: "A different authentication level is required", "More recent authentication is required".
- Example:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="insufficient_user_authentication",
  error_description="A different authentication level is required",
  acr_values="myACR"
```

- Intended client reaction: parse header, send new OIDC authorization request with `acr_values`/`max_age`, get new token carrying `acr` (+ `auth_time`). RS inspects `acr`/`auth_time` from JWT or introspection (RFC 7662).
- RS may issue challenge before or after normal token validation. Security note: challenge leaks required ACR to unauthenticated actors if sent before validation; `acr_values` may disclose info about resource/AS.
- RFC is silent on retry-with-same-token, multiple challenges, and what to do for an unmet `acr` beyond the challenge.

Fit with this PoC (differences):

- RFC 9470 assumes client re-runs authorization-code flow itself. Here API (backend) owns challenge/callback/code exchange (ADR 0001) and a LoA 2 token is not enough anyway. The RFC header is therefore a standards-shaped *signal* ("human step-up required"), not a directly actionable client protocol; UI must call the API's step-up initiation endpoint instead.
- `max_age` does not model single-use (event-count, not time). Omit; `acr_values="2"` suffices. Keycloak LoA 2 Max Age = 0 already forces fresh OTP.
- RFC 9470 says nothing about single-use or handles. Case "LoA 2 token present, no valid Authorization handle" has no RFC code.

## 2. RFC 6750 / RFC 9457

- RFC 6750 sec. 3/3.1 (<https://www.rfc-editor.org/rfc/rfc6750.html#section-3>): `error` codes `invalid_request` (400), `invalid_token` (401), `insufficient_scope` (403); `error_description`, `error_uri`, `realm`, `scope` optional; no error code when request carried no credentials. Other codes via IANA "OAuth Extensions Error" registry.
- RFC 9457 (<https://www.rfc-editor.org/rfc/rfc9457.html>): `application/problem+json`; members `type`, `title`, `status`, `detail`, `instance` + extension members; generators MUST use same status in response as `status`. Silent on `WWW-Authenticate`; no conflict - body and header are orthogonal.
- Fit: Problem Details is a good *body* (machine `type` URI + extension member), header carries the OAuth-standard challenge. Header remains authoritative for generic OAuth clients; body is for the PoC UI and humans. Body optional for correctness - header alone is RFC 9470-compliant.
- Do not put OTP/handle/token values in body (RFC 9457 security considerations: avoid exposing implementation details).

## 3. Spring support (current)

Context7 against Spring Security 7.0 reference + Spring Framework reference. Versions not selected in repo yet.

- No native RFC 9470 support found: no `insufficient_user_authentication` constant. `BearerTokenErrorCodes` lists RFC 6750 codes (e.g. `INSUFFICIENT_SCOPE`). Needs custom code (verify against chosen version; absence-of-evidence).
- `BearerTokenAuthenticationEntryPoint` (401) and `BearerTokenAccessDeniedHandler` (403) format `error`, `error_description`, `error_uri`, `scope` into `WWW-Authenticate` per RFC 6750 sec. 3. Sources: <https://docs.spring.io/spring-security/reference/7.0/servlet/oauth2/resource-server/index.html>, <https://docs.spring.io/spring-security/reference/7.0/api/java/index-all.html>.
- Custom path: throw custom exception from authorization layer; custom `AuthenticationEntryPoint`/`AccessDeniedHandler` (or `@ExceptionHandler`) writes status + header (+ optional body) by hand. Spring Security 7 has `DelegatingMissingAuthorityAccessDeniedHandler` (missing authority -> entry point, adaptive/MFA) - aimed at login-page factors, not OAuth bearer APIs.
- Problem Details: Spring Framework `ProblemDetail`, `ErrorResponse`, `ErrorResponseException`, `ResponseEntityExceptionHandler` (RFC 9457; `application/problem+json`; non-standard fields via `properties` map). Boot toggle `spring.mvc.problemdetails.enabled` (WebFlux: `spring.webflux.problemdetails.enabled`). Source: <https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html>. Caveat (general Spring knowledge, not verified here): `@ControllerAdvice` does not see filter-level errors; entry point/denied handler must write body themselves.
- Spring Security does not emit Problem Details bodies for 401/403 by default (no doc evidence of it).

## 4. Case matrix and options

Existing rules: unauthenticated 401; no business permission 403 (spec). Order matters: check business permission first so step-up requirement is not disclosed to callers lacking permission (input for grilling).

| Case | Option A (RFC-max) | Option B (minimal custom) |
|---|---|---|
| Human, LoA 1 (acr < 2), has permission | 401 + `Bearer error="insufficient_user_authentication", acr_values="2"` | same |
| LoA 2 token, no/invalid/expired/consumed/wrong-operation handle | 401 + `error="insufficient_user_authentication"`, `error_description`, optional `acr_values` + Problem Details `type` `.../step-up-authorization-required` | 403 + Problem Details only (no OAuth error; `type` tells UI to start step-up flow) |
| Inactive/unknown session (introspection) | 401 `Bearer error="invalid_token", error_description="session inactive"` (RFC 6750); handle not consumed | same |
| Missing/malformed ACR on human token | 401 `invalid_token` (fail closed; never integration path) | 403 Problem Details (`type` = invalid-authentication-context) |

Notes on the handle case (the non-standard part):

- Presenting a handle is an application precondition, not an OAuth token property; `acr_values` cannot express it, and asking for `acr_values=2` is misleading when the token already has acr 2 (a generic RFC 9470 client would re-run Keycloak auth, get a fresh LoA 2 token, still no handle).
- Option A reuses the 9470 code as "fresh user authentication needed" (semantically close). Risk: generic clients auto-redirect to Keycloak, bypassing API-owned flow. Harmless per ADR (browser-submitted token is never issuance evidence) but useless; UI must not use generic handling.
- Option B avoids misusing OAuth semantics but forgoes a standards signal for this case; only Problem Details `type` is custom (RFC 9457 supports custom `type` URIs/extensions, so not a new protocol).
- Hybrid: 401 + RFC 9470 header when token lacks acr 2; for "LoA 2 but no handle" either option above. In all cases Problem Details body with stable `type` + extension members (operation identifier, step-up initiation hint) lets UI branch without parsing prose.
- 401 = "authenticate (again)"; 403 = "known, refused". Rejection before execution leaves handle unconsumed, so inactive session vs invalid handle should have different `type`s.
- Disclosure: distinguishing expired / consumed / other-user handle externally helps UX but leaks state; grilling input.

## 5. Inputs for grilling (unresolved)

1. 401 (RFC 9470 examples) vs 403 for LoA-insufficient?
2. Reuse `insufficient_user_authentication` for "LoA 2 but no valid handle", or Problem Details only?
3. Always include `acr_values="2"`? Include `max_age`? (leaning omit max_age.)
4. Problem Details body on all step-up rejections? Required `type` URIs / extension fields?
5. Distinguish handle missing / expired / consumed / wrong-operation in `type`/`detail`?
6. Missing/malformed ACR: 401 `invalid_token` vs 403 vs 9470?
7. Permission check before step-up check (avoid disclosure)? Spec implies yes.
8. Filter-level handlers write body, or checks run in method-security layer with `@ControllerAdvice`? (depends on unselected Spring versions.)

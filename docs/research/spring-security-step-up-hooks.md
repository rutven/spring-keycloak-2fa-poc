# Spring Security hooks for step-up policy

Research date: 2026-10-17. Ticket: #5 (map #1). Status: source/doc reading, not runtime-verified. No code written beyond snippets.

Baseline (given): Spring Boot 4.1.1, Spring Security 7.1.1, Java 25, Maven; OAuth2 resource server (JWT), Keycloak 26.8.1. Framework version not independently checked.

## Sources

Primary: Spring Security source at tag `7.1.1` (cloned and read; paths below relative to repo root) and its in-tree reference docs. Rendered docs: <https://docs.spring.io/spring-security/reference/> (unversioned URLs redirect; pages match `docs/modules/ROOT/pages/` at the tag).

- Method security: <https://github.com/spring-projects/spring-security/blob/7.1.1/docs/modules/ROOT/pages/servlet/authorization/method-security.adoc>
- MFA: <https://github.com/spring-projects/spring-security/blob/7.1.1/docs/modules/ROOT/pages/servlet/authentication/mfa.adoc>
- JWT resource server: <https://github.com/spring-projects/spring-security/blob/7.1.1/docs/modules/ROOT/pages/servlet/oauth2/resource-server/jwt.adoc>
- Opaque token / RestClient introspection: <https://github.com/spring-projects/spring-security/blob/7.1.1/docs/modules/ROOT/pages/servlet/oauth2/resource-server/opaque-token.adoc>
- RFC 9470 (step-up challenge): <https://www.rfc-editor.org/rfc/rfc9470.html>
- `@Transactional` order: <https://github.com/spring-projects/spring-framework/blob/main/spring-tx/src/main/java/org/springframework/transaction/annotation/EnableTransactionManagement.java> (`main`, not tag-pinned)

Context7 not used; tag source is more authoritative for 7.1.1.

## Summary

- Built-in Spring Security 7 MFA does not fit: it checks factors accumulated on the `Authentication`, not per-invocation fresh OTP or server-side single-use state (see "Built-in MFA").
- The extension point that fits is method security with a custom `AuthorizationManager<MethodInvocation>` (or a `@PreAuthorize` bean expression), plus a custom annotation carrying the stable operation id.
- The policy check is naturally non-consuming. Consumption needs a separate hook after all pre-checks. The payment state check (409 without consumption, serialized with consume and transition) makes a purely interceptor-based consumption awkward. This is the main open design tension.
- A distinct "step-up required" outcome can be carried as a custom `AuthorizationResult` inside `AuthorizationDeniedException`, then mapped to a custom `WWW-Authenticate` by an `AccessDeniedHandler` or `@ExceptionHandler`.

## Verified facts from 7.1.1 source

### Method interceptor mechanics

- `AuthorizationManagerBeforeMethodInterceptor` has a public ctor `(Pointcut, AuthorizationManager<MethodInvocation>)`. Its pointcut helper (`AuthorizationMethodPointcuts`) is package-private, so supply your own `Pointcut` (e.g. Spring `AnnotationMatchingPointcut`). `core/.../authorization/method/AuthorizationManagerBeforeMethodInterceptor.java`.
- Flow in `attemptAuthorization`: `authorizationManager.authorize(...)`; if result not granted, delegate to handler; else `mi.proceed()`. If the manager itself implements `MethodAuthorizationDeniedHandler`, it is used as the handler; default handler rethrows the `AuthorizationDeniedException` (wrapping the result as `new AuthorizationDeniedException("Access Denied", result)`, so a custom result type is preserved via `getAuthorizationResult()`). `ThrowingMethodAuthorizationDeniedHandler.java`.
- `AuthorizationDeniedException extends AccessDeniedException implements AuthorizationResult`; wraps an `AuthorizationResult`; refuses granted results.
- `AuthorizationResult` is a plain interface (`boolean isGranted()`), so a custom subtype can carry fields (operation id, required ACR, reason). `AuthorizationManager.authorize` returns `@Nullable AuthorizationResult` (null = abstain).
- Docs endorse the pattern: publish the manager, then an `Advisor` bean with `@Role(ROLE_INFRASTRUCTURE)`; "You can place your interceptor in between Spring Security method interceptors using the order constants specified in `AuthorizationInterceptorsOrder`"; manager may implement `MethodAuthorizationDeniedHandler`. method-security.adoc, section "Using a Custom Authorization Manager". Can be added alongside normal `@PreAuthorize` (no need for `prePostEnabled=false` if pointcut/annotation is distinct; not runtime-checked).
- `authorize` may be called more than once per invocation by other code paths (e.g. `verify`, event publishing hooks); consuming inside it is unsafe by contract. Not documented as a guarantee either way: treat the manager as side-effect-free.

### Annotation / templating

- `AnnotationTemplateExpressionDefaults` bean (static `@Bean`) enables `{attr}` placeholders in meta-annotations; docs example `@PreAuthorize("hasRole('{value}')")` on `@HasRole`. Quotes must be handled by author (string attr substituted raw; for arrays include quotes in the values). method-security.adoc "Templating Meta-Annotation Expressions". Class has one property `ignoreUnknown` (default true).
- Bean-in-SpEL: `@PreAuthorize("@authz.decide(#root)")` where bean method takes `MethodSecurityExpressionOperations`; may return `Boolean`, `null` (abstain) or custom `AuthorizationDecision`. `#root` exposes `getThis()`/authentication, but not `Method` directly, so the operation id must be passed in as an expression argument (via template), e.g. `@stepUp.satisfied(#root, '{value}')`.
- Inside a custom manager the id can be read from the `MethodInvocation` with `SecurityAnnotationScanners.requireUnique(RequiresStepUp.class).scan(method, targetClass)` (public API, handles meta-annotations; `core/.../annotation/SecurityAnnotationScanner.java`). Plain Spring `AnnotatedElementUtils` is the alternative.

### Interceptor ordering

- `AuthorizationInterceptorsOrder`: PRE_FILTER 100, PRE_AUTHORIZE 200, SECURED 300, JSR250 400, SECURE_RESULT 450, POST_AUTHORIZE 500, POST_FILTER 600 (ordinal*100 rule; explicit for 450/500/600). `@EnableMethodSecurity(offset=n)` shifts the built-ins (since 6.3).
- `@Transactional` advisor default order is `Ordered.LOWEST_PRECEDENCE` (`@EnableTransactionManagement.order`), so security interceptors run outside the transaction by default. A custom consuming interceptor at e.g. 250 also runs before the transaction opens. Consumption state is in-memory so not rolled back by a tx rollback either way (consistent with "consumed even if execution fails"). Spring Boot's auto-config does not obviously override the order (unverified).

### Built-in MFA (7.x): fit and conflicts

- `@EnableMultiFactorAuthentication(authorities=..., when=...)` (`@since 7.0`, `when` `@since 7.1`) publishes an `AuthorizationManagerFactory` that AND-s required `FACTOR_*` authorities into `hasRole/hasAuthority/...` rules, for both HTTP and method security (`SpEL` `hasAuthority` in `@PreAuthorize` goes through the factory). Also installs a bean post-processor setting `mfaEnabled` on authentication filters. `config/.../authorization/EnableMultiFactorAuthentication.java`, `EnableMfaFiltersConfiguration.java`.
- `FactorGrantedAuthority` (`FACTOR_PASSWORD`, `FACTOR_OTT`, `FACTOR_WEBAUTHN`, `FACTOR_BEARER`, ...) carries `issuedAt`. `RequiredFactor.withAuthority(..).validDuration(Duration)` + `AllRequiredFactorsAuthorizationManager` give "factor issued within N" checks; denial yields `FactorAuthorizationDecision` with `RequiredFactorError` (missing/expired).
- Resource server JWT: `JwtAuthenticationConverter` adds `FACTOR_BEARER` only (`oauth2-resource-server/.../JwtAuthenticationConverter.java`). No OTP/ACR factor is derived from the token. A custom converter could map `acr`/`amr` to a `FACTOR_*` authority, but this only restates the LoA 2 token claim, which the ADR/spec says is insufficient.
- Misfit with requirements: (1) validity is a time window on the authority, not single use; ADR rejects reusable assurance windows; (2) no server-side consumption, session-binding, op-binding or replacement; (3) failure handling (`DelegatingMissingAuthorityAccessDeniedHandler`) is built around redirecting to login entry points per missing `FACTOR_*`, not API challenges; (4) the factor is established by Spring's own login mechanisms, whereas here the API runs a Keycloak OIDC flow and issues its own handle.
- Possible reuse only: `FactorAuthorizationDecision`/`RequiredFactorError` types as a result vocabulary (stretches the type: it is factor-authority based). Recommendation as input: do not enable `@EnableMultiFactorAuthentication`; confirm it is not accidentally active (it changes global `hasAuthority` semantics, which would silently AND factors into the business-permission checks). `AuthorizationManagerFactory` (core, `@since 7.0`) is still a legitimate seam if a design wants `hasAuthority` customised, but is not needed.

### Surfacing a distinct "step-up required" failure

- `ExceptionTranslationFilter` catches `AccessDeniedException` and calls the `AccessDeniedHandler` (non-anonymous). With resource server defaults, `BearerTokenAccessDeniedHandler` answers 403 with `WWW-Authenticate: Bearer error="insufficient_scope"` for JWT-authenticated requests. That mislabels a step-up failure, so a custom handler or advice is needed. `oauth2-resource-server/.../web/access/BearerTokenAccessDeniedHandler.java`.
- Mapping options: (a) custom `AccessDeniedHandler` registered via `exceptionHandling(...)` / `defaultAccessDeniedHandlerFor(handler, matcher)`, that unwraps `AuthorizationDeniedException.getAuthorizationResult()` and checks `instanceof StepUpRequiredResult`, else delegates to `BearerTokenAccessDeniedHandler`; (b) `@RestControllerAdvice` `@ExceptionHandler(AuthorizationDeniedException.class)` producing a `ProblemDetail` and header (runs inside MVC, earlier than the filter; method security on a controller bean propagates the exception through `DispatcherServlet`, behaviour not runtime-checked). The unwrap helper pattern is visible in `DelegatingMissingAuthorityAccessDeniedHandler.authorityErrors`.
- Standards shape: RFC 9470 defines `HTTP 401` + `WWW-Authenticate: Bearer error="insufficient_user_authentication", error_description="...", acr_values="..."` (and `max_age`). Section 3. It assumes the client re-runs authorization and returns a new token; the PoC's API-owned flow plus handle differs, so only the error shape/status is reusable (acr_values would announce LoA 2; the OTP initiation URL/handle protocol is application-defined). Whether to use 401 vs spec's 403 wording is a grilling item. Distinguish sub-cases (`missing_handle`, `expired`, `stale`, `session_inactive`) in `error_description` or a `ProblemDetail` body; they must not reveal other users' state.
- Distinct from 401 (no/invalid token: the bearer filter's entry point) and 403 (business permission missing: ordinary `hasAuthority` denial).

### JwtAuthenticationConverter for `resource_access.<client>.roles`

- `JwtGrantedAuthoritiesConverter` only reads flat claims (`scope`, `scp` by default, or one configured claim name; `JwtGrantedAuthoritiesConverter.java`), so nested Keycloak `resource_access.<client>.roles` needs a custom `Converter<Jwt, Collection<GrantedAuthority>>`.
- Wire via `JwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(...)`. In 7.x `convert()` is `final` and always adds `FACTOR_BEARER`; do not subclass. Resulting authorities are the union of the converter's output and `FACTOR_BEARER`.
- Authority naming: map roles to business-permission strings such as `PAYMENT_APPROVE` directly (no `ROLE_` prefix), or keep `ROLE_` and use `hasRole`. Decide whether realm roles / scopes are ignored (should be, to keep a single permission source).
- Keycloak claim shape (`resource_access.<client_id>.roles` array) is from Keycloak docs, not re-fetched here: unverified. Use the resource server's own client id; a token for another audience should fail via audience validation (separate `JwtDecoder` validator).
- ACR/`sid` read from `Jwt` claims (`acr`, `sid`); missing/malformed `acr` on a human token must fail closed per spec. Trusted-service-account detection (client id claim, `azp`, allowlist) belongs in the policy, not the converter.

### RestClient-based introspection for the session-status check

- 7.1 adds `RestClientOpaqueTokenIntrospector(String introspectionUri, RestClient)` (`@since 7.1`; `OpaqueTokenIntrospector.introspect(String token)`). It POSTs `token=...` form body, requires `active=true` else throws `BadOpaqueTokenException` (subclass of `OAuth2IntrospectionException`); any transport/HTTP error becomes `OAuth2IntrospectionException`. It trusts the server and does not check `exp`. Client authentication must be configured on the supplied `RestClient` (Basic via interceptor/default header; Spring Boot docs TIP: `OAuth2ResourceServerProperties` offers URI and credentials). Opaque-token docs section "Using `RestClientOpaqueTokenIntrospector`", including timeouts via `RequestFactory`.
- Usable standalone as a session-liveness probe on the JWT's raw value (`jwt.getTokenValue()`): `BadOpaqueTokenException` => inactive (reject, no consume); other `OAuth2IntrospectionException`/timeouts => unknown (reject, no consume, fail closed). The result is a principal with attributes (`OAuth2IntrospectionAuthenticatedPrincipal`) so `sid`/`sub`/`client_id` can be cross-checked if Keycloak returns them (unverified).
- It maps `scope` to `SCOPE_*`; irrelevant for this use. A direct `RestClient` call would be the alternative if the response must be inspected beyond `active`, or if one wants to tell 4xx from 5xx explicitly.
- Whether Keycloak introspection reports `active=false` once the SSO session is terminated (vs token merely unexpired) belongs to the Keycloak feasibility ticket; not asserted here.

## Candidate designs

All three keep business permission as separate `@PreAuthorize("hasAuthority('PAYMENT_APPROVE')")` (runs at order 200 alongside). Illustrative annotation:

```java
@Retention(RUNTIME) @Target(METHOD)
@interface RequiresStepUp { String operation(); }   // e.g. "payment.approve"
```

### Design 1: meta-annotation + `@PreAuthorize` bean expression

```java
@Retention(RUNTIME) @Target(METHOD)
@PreAuthorize("@stepUp.satisfied(#root, '{operation}')")
@interface RequiresStepUp { String operation(); }
// plus: @Bean static AnnotationTemplateExpressionDefaults ...
```

- `StepUp.satisfied` returns a custom `AuthorizationDecision` subtype (non-consuming): trusted service account => granted; else ACR check + peek handle (from `Step-Up-Authorization` header via `RequestContextHolder`) + session introspection.
- Consumption: not in SpEL (side effect inside expression evaluation; ordering vs other expressions on the same method unclear). Use either an explicit call inside the service (`stepUpAuthorizations.consume(...)` after state validation) or a companion interceptor (Design 3 style).
- Pros: least infrastructure; id declared once, readable; uses documented features only.
- Cons: policy lives behind a string expression (weaker typing/IDE support, errors at runtime); no access to `Method` so id must be threaded as a literal; unwrap of custom decision depends on the handler path; reading the request header requires thread-bound request access inside the policy (couples policy to servlet); harder to put the denied-handler on the manager.

### Design 2: dedicated annotation + custom `AuthorizationManager<MethodInvocation>` + `AuthorizationManagerBeforeMethodInterceptor`

```java
@Bean @Role(ROLE_INFRASTRUCTURE)
Advisor stepUpAdvisor(StepUpAuthorizationManager m) {
  var i = new AuthorizationManagerBeforeMethodInterceptor(
      new AnnotationMatchingPointcut(null, RequiresStepUp.class, true), m);
  i.setOrder(AuthorizationInterceptorsOrder.PRE_AUTHORIZE.getOrder() + 10); // after business permission
  return i;
}
```

- Manager scans the annotation for the operation id, returns granted / `StepUpRequiredResult` (custom `AuthorizationResult`) / plain denial; implements `MethodAuthorizationDeniedHandler` if special handling is wanted. Pure, unit-testable policy class independent of controllers (matches spec "reusable component").
- Consumption options with this design:
  - 2a: second interceptor/manager chained after it (order +20) that consumes then proceeds, only reached if all earlier checks passed.
  - 2b: explicit `consume` call in the service method body (declared id still from annotation, consumption by calling code).
- Pros: typed, testable; distinct failure result; id read from the declared annotation (stable, route/method-name independent); explicit ordering control; can compose with `@PreAuthorize` business check without interference.
- Cons: more custom infra (advisor bean, pointcut, annotation scanning); need to ensure proxying happens for the annotated bean (controller vs service layer; self-invocation bypass); custom order choice must be kept coherent with `offset`.

### Design 3: single gate interceptor (check + introspect + consume + proceed), not an `AuthorizationManager`

- A custom `MethodInterceptor`/`PointcutAdvisor` (like Design 2's advisor but with its own `invoke`) does: policy -> session check -> atomic consume -> `proceed()`. One class owns the full lifecycle, with consumption strictly after earlier interceptors (business permission at 200) and before `@Transactional`.
- Pros: declarative and hard to forget; consumption provably tied to "execution begins" (immediately before `proceed()`); no policy/consume split to coordinate.
- Cons: collapses policy and effect into one unit, weakening "policy check must not consume" separation/testing (can still keep policy as an inner pure collaborator); loses `AuthorizationEventPublisher`/denied-handler plumbing unless re-implemented; bypasses the standard `AuthorizationManager` abstraction.

### Cross-cutting consumption trade-off (input for grilling)

The spec requires: payment state check (409), consume, and transition serialized, and a loser or a 409 must not consume (poc-description.md lines ~455-465, 521-523). An interceptor that consumes before the method body cannot see payment state unless (i) the state check is performed inside the gate (e.g. a pre-execution precondition callback or a lock around state check + consume + body), or (ii) consumption moves into the method body after validation (explicit call, Design 2b / 1). Options to compare:

| Option | Atomic consume | 409-without-consume | Forgettable | Declarative |
|---|---|---|---|---|
| Interceptor consumes before body (2a/3) | yes | only with precondition hook or global lock + in-gate state check | no | yes |
| Explicit consume in service after state check, under a lock | yes | yes, natural | yes (mitigate by test / wrapper API) | partial |
| Consume inside `AuthorizationManager` | possible | no | no | yes, but violates non-consuming policy and unsafe if invoked repeatedly |

Single instance, in-memory: a coarse lock or per-operation lock is available as the simplest serialization (decision for grilling).

## Open questions (for the grilling round)

- Layer: put the annotation on controller methods or service methods? (proxying, exception propagation path, test seams)
- 401 vs 403 for "step-up required": follow RFC 9470 `401` + `insufficient_user_authentication` or keep 403?
- Consumption hook: interceptor vs explicit call? How to fit the payment 409 pre-check and serialization?
- Where does the session introspection run: in the policy (peek) and again at consumption, or only once immediately before consumption?
- Does the service-account allowlist check live in the policy class or in the converter (extra authority)?
- Introspect with `RestClientOpaqueTokenIntrospector` or a thin direct `RestClient` call (finer 4xx/5xx distinction)?
- Is `@EnableMultiFactorAuthentication` ruled out (recommended input: yes)? Add a test asserting no `AuthorizationManagerFactory` bean is present.

## Not verified

- Runtime behaviour: exception propagation from controller-level method security to `ExceptionTranslationFilter` vs MVC resolvers; `Step-Up-Authorization` header access in policy; advisor ordering in a real context.
- Spring Boot 4.1.1 auto-configuration effects on `@Transactional` order and on MFA/method security wiring.
- Keycloak token shape (`resource_access`, `sid`, `acr`) and introspection semantics for terminated sessions.
- That Spring Security 7.1.1 is the version managed by Boot 4.1.1 (taken from the ticket).

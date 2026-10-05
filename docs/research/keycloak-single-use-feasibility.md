# Keycloak single-use step-up feasibility

Research date: 2026-10-05. Status: documentation research, not runtime verification.

## Conclusion

Built-in Keycloak documents fresh step-up authentication and TOTP replay prevention. These support proceeding to a reproducible feasibility experiment, but do not establish that the entire agreed lifecycle is achievable. The user accepted checking session status immediately before execution, rejecting inactive or unknown status, and invalidating unused authorization on validated logout notifications, with the race between the final check and execution explicitly accepted. API flow-initiation time is an accepted conservative expiry fallback if factor-specific timing is unavailable. Replay-safe authorization issuance and the agreed lifecycle still need runtime verification before implementation.

No baseline version, architecture, or runtime tooling is selected here. No Keycloak server was started and no token claims or concurrent behavior were observed.

## Sources and versions examined

Context7 resolved `Keycloak` to `/keycloak/keycloak`, the official repository with high source reputation, and was queried separately for step-up freshness, OTP replay, and session termination. Its responses referenced moving `main` source. The available tool does not expose the requested `researchMode` option, so gaps were investigated with official documentation and first-party source instead.

The live [Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/index.html) identifies itself as **26.8.0** when retrieved. This is the documentation examined, not a selected dependency. [OIDC application guidance](https://www.keycloak.org/securing-apps/oidc-layers) and GitHub `main` are moving sources. Attempts to retrieve 26.5.2 source through the browsing tool failed; no claim below is represented as verified against that release. Freeze and recheck the eventual baseline rather than assuming moving documentation describes an older version.

## Fresh OTP for each operation

The documented browser step-up flow has LoA 1 password authentication and LoA 2 with a required OTP Form. Its LoA 2 condition uses **Max Age = 0**, which prevents reuse of that level on later authentication requests explicitly requesting it. Request and validate the expected ACR; a browser can alter request parameters. [Keycloak step-up flow and flow logic](https://www.keycloak.org/docs/latest/server_admin/index.html#_step-up-flow).

This is the LoA condition's configuration, not interchangeable evidence that OIDC `max_age=0` alone guarantees OTP. Every sensitive invocation must initiate a new correlated authorization flow. A required OTP execution and coherent LoA mapping are essential: ACR describes the configured authentication context, not an intrinsic guarantee of a particular factor. Audit alternate flow paths, enrollment, and client overrides in the actual realm. Existing LoA 2 tokens remain insufficient for execution under the agreed API policy.

## OTP replay prevention

Current documentation provides **Reusable code** under TOTP policy; reuse is disabled by default. Set it explicitly to false in reproducible configuration. [Keycloak OTP policy](https://www.keycloak.org/docs/latest/server_admin/index.html#_otp_policies).

First-party `main` source validates TOTP and, unless reuse is enabled, invokes `SingleUseObjectProvider.putIfAbsent`. The key combines credential ID and entered code; retention equals OTP period multiplied by `(2 × look-around-window + 1)`. This supports a replay barrier shared across authentication sessions for the same credential, subject to the selected provider's behavior. [OTPCredentialProvider source](https://github.com/keycloak/keycloak/blob/main/services/src/main/java/org/keycloak/credential/OTPCredentialProvider.java).

This is not permanent prohibition of a numeric string: digits can recur in later TOTP intervals, and separate credentials have separate keys. Interpret replay prevention over the code's accepted validity window. If the requirement literally forbids reuse of the same digits forever or across credentials, this evidence does not satisfy it. A user may need to wait for the next code before another sensitive call. Verify two concurrent submissions, separate sessions, neighboring accepted intervals, multiple enrolled credentials, and Keycloak restart behavior; documentation alone does not prove these outcomes.

## Evidence for API authorization issuance

Keycloak supports Authorization Code flow returning access, refresh, and ID tokens after code exchange. [Keycloak Authorization Code guidance](https://www.keycloak.org/securing-apps/oidc-layers#_authorization_code).

OIDC defines nonce correlation and ID-token validation, including issuer, audience, signature, and expiry. `auth_time` identifies end-user authentication, whereas `iat` identifies token issuance; refresh preserves the original authentication time. Neither standard claim by itself identifies a unique fresh OTP event. [OIDC Core ID Token claims](https://openid.net/specs/openid-connect-core-1_0.html#IDToken), [ID Token validation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation), and [refresh response](https://openid.net/specs/openid-connect-core-1_0.html#RefreshTokenResponse).

**Accepted architectural direction:** the API owns the operation-bound challenge, callback, and authorization-code exchange, and completes each challenge atomically once before issuing authorization. An arbitrary browser-submitted LoA 2 token must not serve as issuance evidence. Protocol details remain design recommendations requiring baseline validation: an unpredictable challenge binding user, issuer, session, stable operation ID, state, nonce, and PKCE verifier; permission checking before redirect; validation of authentication evidence and required ACR; and atomic replacement of pending authorization.

Replay barriers must cover both stages: callback completion and sensitive execution. Replayed state, code, ID token, access token, or refresh output must not create another authorization after consumption, replacement, expiry, or application restart. Keeping challenges only in process memory makes callbacks for pre-restart challenges unknown and rejectable. Authorization-code single use alone does not replace atomic callback handling. [OAuth authorization-code single-use requirement](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.1.2).

The configured two-minute lifetime starts at **successful OTP verification** when trustworthy factor-completion evidence is available, not callback arrival. Verify whether the selected Keycloak flow exposes that timestamp; `auth_time` may reflect other authentication activity and token `iat` may be later. The user accepted API initiation of the correlated OTP flow as a conservative fallback if that evidence is unavailable. Time spent completing OTP reduces the remaining usable lifetime. Delayed callbacks cannot extend the deadline, and an expired flow must not issue authorization. Document the baseline's chosen time origin.

## Session binding and logout

OIDC back-channel logout uses a signed Logout Token containing subject and/or session identity. Session IDs are scoped to their issuer. Validated logout events allow the API to invalidate state for the affected session; a subject-only event targets all that subject's sessions. [Back-channel Logout Token](https://openid.net/specs/openid-connect-backchannel-1_0.html#LogoutToken), [validation](https://openid.net/specs/openid-connect-backchannel-1_0.html#Validation), and [logout actions](https://openid.net/specs/openid-connect-backchannel-1_0.html#BCLogoutActions).

Verify the actual ID-token/access-token/introspection `sid` representation, its persistence across refresh, and its agreement with Logout Tokens. Key by validated issuer and subject, with the validated session identity bound to each authorization. A client ID or an unvalidated browser session value is insufficient.

Keycloak introspection source checks token and user-session validity before returning active status. Introspection requires authenticated client configuration; configure the API audience correctly and verify the selected version's audience behavior. [AccessTokenIntrospectionProvider source](https://github.com/keycloak/keycloak/blob/main/services/src/main/java/org/keycloak/protocol/oidc/AccessTokenIntrospectionProvider.java), [introspection endpoint guidance](https://www.keycloak.org/securing-apps/oidc-layers#_introspection_endpoint), and [RFC 7662 response semantics](https://www.rfc-editor.org/rfc/rfc7662.html#section-2.2).

Candidate behavior: introspect on every sensitive human call without positive-result caching, reject inactive or unknown status, and leave authorization unconsumed on rejection. Back-channel logout can proactively remove authorization and serialize local invalidation with consumption. These are design recommendations, not runtime findings.

**Strict immediate logout remains a feasibility gap.** An active introspection result is a point-in-time observation: logout can occur after that response and before local execution starts. A callback is also delivered over a network and can be delayed or fail. Local locking cannot serialize a remote Keycloak logout event with local execution. No cited built-in protocol supplies an atomic cross-service session-check-and-execution transaction. This logical race remains even if normal logout tests pass. Defining logout only after local invalidation, narrowing the guarantee to logout observed before the final check, or coordinating logout through the API would change the current requirement and needs an explicit decision.

### Decision following research

The user accepted narrowing the logout guarantee to a session-status check immediately before execution, rejection of inactive or unknown status, and logout notifications invalidating unused authorization. The race described above is accepted; strict cross-service atomic ordering is no longer a requirement. The mechanisms still need runtime validation. See [ADR 0001](../adr/0001-single-use-step-up-authorization.md).

## Required feasibility checks

Before application implementation, select a baseline and reproduce a realm with required OTP, coherent ACR mappings, LoA 2 Max Age zero, and non-reusable TOTP policy. Record sanitized claims and evidence; never store codes, OTP secrets, credentials, or live tokens in the repository.

1. Repeated explicit step-up requests challenge OTP despite an existing LoA 2 token; modified ACR requests cannot issue authorization.
2. Repeated and concurrent use of one valid code accepts at most one verification within its validity window, including across login sessions.
3. Correlated callbacks issue once; replay cannot reissue after consumption, replacement, expiry, or API restart. Old challenges cannot bind to a new session.
4. Establish a trusted factor timestamp or use the agreed flow-initiation fallback. Delayed callbacks cannot extend expiry, and callbacks after the deadline cannot issue authorization.
5. Refresh preserves session binding and cannot issue, revive, or extend authorization. Another session cannot consume it.
6. Logout with an unexpired access token causes introspection rejection and local invalidation; introspection outage fails closed without consumption. Test admin logout and browser logout separately.
7. Pause between a positive introspection response and execution, then log out. Document the accepted race and verify that a logout notification processed before consumption blocks execution.

Proceed with focused feasibility experiments, not full implementation. Fresh authentication and code replay controls have documentation support; replay-safe issuance remains a gate. The agreed flow-initiation fallback removes exact OTP-completion-time evidence as a blocker. Runtime validation of expiry and the narrowed logout guarantee is still required.

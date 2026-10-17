# Keycloak 26.8.1 token claims

Research date: 2026-10-17. Baseline: Keycloak 26.8.1 (`quay.io/keycloak/keycloak:26.8.1`). Builds on [keycloak-single-use-feasibility.md](keycloak-single-use-feasibility.md); not repeated here. Closes the "claims / ACR / `sid`" gates of issue #4.

Evidence labels: **[doc]** = keycloak.org 26.8.1 docs; **[src]** = first-party source at tag `26.8.1`; **[spec]** = OpenID/IETF spec; **[unverified]** = inference, needs runtime check. Nothing here was run; all example claims are *expected, to verify at feasibility baseline*.

Doc bases: [Server Admin 26.8.1](https://www.keycloak.org/docs/26.8.1/server_admin/index.html), [Upgrading 26.8.1](https://www.keycloak.org/docs/26.8.1/upgrading/index.html), [Securing apps / OIDC](https://www.keycloak.org/securing-apps/oidc-layers). Source base: `https://github.com/keycloak/keycloak/blob/26.8.1/` (paths below relative to it).

## Summary / decisions this changes

1. `acr` is **not** an integration discriminator. Service-account tokens most likely carry `acr` = `1` (or its mapped name), not an absent claim [src]. Classify by `azp` allowlist + service-account markers, never by ACR absence. Human tokens: `acr` must map to LoA 2 else fail closed.
2. `acr` reflects the *request* (`acr_values`/`claims`) capped by the achieved LoA, so a browser-tampered request yields a lower `acr`, not an error. API must compare `acr` to the expected LoA 2 value [src, doc].
3. `sid` is present in access, ID, refresh tokens, introspection responses; stable across refresh; equals Logout Token `sid` **only if** poc-app has *Backchannel logout session required* = true [src]. Absence of `sid` marks default (transient) `client_credentials` tokens but must not be the sole discriminator.
4. Introspection validates `aud` since 26.6.2: introspecting client must be in the token `aud`, else `{"active":false}` [doc, src]. Audience Resolve **never adds the token's own client** [src], so tokens issued to poc-app with poc-app roles lack `aud=poc-app` unless an Audience mapper adds it. Realm design must handle this.
5. No OTP-completion timestamp claim exists. Best built-in evidence: `auth_time` (updated on each non-SSO completion) [src]. Agreed API-initiation fallback stands.
6. `amr` has a built-in mapper but is not default, not in introspection, and has no timestamp [src].

## ACR / LoA

- Feature `step-up-authentication` is `Type.DEFAULT` (enabled) in 26.8.1 [src `common/.../Profile.java`]. Realm default client scope `acr` holds mapper `oidc-acr-mapper` (access, ID, introspection all on) [src `OIDCLoginProtocolFactory.java`].
- ACR-to-LoA mapping: realm attribute `acr.loa.map` (JSON `{"<acr>": <int LoA>}`), overridable per client attribute of the same name; client map wins if non-empty, else realm [src `AcrUtils.getAcrLoaMap`, `Constants.ACR_LOA_MAP`]. Admin UI: Realm settings > General (realm, "recommended"), Client > Advanced (client) [doc: [ACR to LoA mapping](https://www.keycloak.org/docs/26.8.1/server_admin/index.html#_mapping-acr-to-loa-realm)]. ACR can be any string; LoA numeric.
- Request LoA 2 [doc: [step-up flow](https://www.keycloak.org/docs/26.8.1/server_admin/index.html#_step-up-flow)]:
  - `claims={"id_token":{"acr":{"essential":true,"values":["2"]}}}` (URL-encoded), or
  - `acr_values=2` (treated as non-essential).
  - Client `Default ACR Values` (attr `default.acr.values`) apply only when neither is sent, and "cannot be reliably used to enforce" a level [doc]. Client `minimum.acr.value` is documented for SAML only [doc]; `AcrUtils.enforceMinimumAcr` also applies to OIDC code path [src], but do not rely on it without a test.
  - Docs: params can be altered in the browser; mitigate with PAR/request object and verify `acr` in the token [doc]. API-owned flow (ADR 0001) fits: API verifies `acr` in the ID token from its own code exchange.
- Mapper algorithm [src `AcrProtocolMapper.getAcr`]: `loa` = client-session LoA note (`NO_LOA=-1` if absent); if `< 0`, `loa = SSO-authentication ? 0 : 1`. Then pick, in order: highest *essential* requested value with mapped LoA <= `loa`; else highest requested (`acr_values`/`claims`/client default) <= `loa`; else highest map key <= `loa`; else `String.valueOf(loa)`. Unmapped numeric strings parse as LoA themselves. Consequences:
  - Achieved LoA 2, request `acr_values=1` -> `acr` = `1` [src, inferred].
  - SSO-cookie-only auth with expired level -> `acr` = `0` [doc].
  - Refresh re-runs the mapper from stored client-session notes (original request params, LoA) -> `acr` stable across refresh [src, unverified at runtime].
- Max Age = 0 on the LoA 2 condition: that level is valid only for the current authentication; every later request for it re-authenticates [doc]. Config key in flow: *Conditional - Level Of Authentication*, `Level of Authentication`=2, `Max Age`=0; LoA 1 subflow first [doc]. Docs example uses OTP Form as the second step. Not interchangeable with OIDC `max_age` [feasibility doc]; do not use `max_age` or `prompt=login` unverified. `prompt=login` is suspect: `AuthenticationManager.redirectAfterSuccessfulFlow` removes a prior user session whose id differs from the cookie session [src], which could change `sid` and kill the UI session [unverified].
- Session continuity: step-up from an existing SSO session (cookie present in popup) re-uses the user session -> same `sid` [unverified]; popup without SSO cookie does full LoA 1+2 login -> new session, new `sid` -> API must reject (binding to original `sid`).

## `auth_time`

- Built-in mapper `auth_time` lives in default client scope **`basic`** (with `sub`), emitted in access, ID, introspection (not userinfo) [src `OIDCLoginProtocolFactory.java`; doc [25.0.0 migration](https://www.keycloak.org/docs/26.8.1/upgrading/index.html#new-default-client-scope-basic)]. If a realm already had a scope named `basic`, it was not auto-added [doc]; irrelevant for fresh PoC realm.
- Value is the user-session note `AUTH_TIME`, written in `redirectAfterSuccessfulFlow` **unless the authentication was SSO-cookie only** (then skipped) [src]. A step-up that ran the OTP form is non-SSO -> `auth_time` = completion time of the step-up flow (~OTP success, plus form-submit latency) [src, unverified at runtime].
- Caveats: note is per *user session* (shared by all clients/tabs); a later interactive login in the same session rewrites it, and tokens minted afterwards (incl. refresh) show the newer value. It is not OTP-specific (password-only re-login also updates it). Not present on service-account tokens (no `AUTH_TIME` note on `client_credentials` path) [src].
- Usable as corroboration: require `auth_time >= challenge-initiation time - skew` (proves an interactive auth after API initiation; SSO reuse would leave it stale). Do not use alone as OTP proof. OIDC Core: `auth_time` = end-user authentication time vs `iat` = issuance [spec].

## `amr`

- Built-in mapper `oidc-amr-mapper` ("Authentication Method Reference (AMR)"), access + ID token only; **not** an introspection mapper, **not** in any default scope [src `AmrProtocolMapper.java`; doc [authenticator reference value](https://www.keycloak.org/docs/26.8.1/server_admin/index.html)]. Needs a *reference value* (+ optional max age) configured on each authentication execution (config keys `default.reference.value`, `default.reference.maxAge`) [src `Constants`].
- Value present only while `completionTime + maxAge >= now`; default maxAge 0 -> disappears after the same second for later token issuance unless configured [src `AmrUtils.isAmrValid`]. Claim is a string array, e.g. `["pwd","otp"]` [unverified format].
- Not a timestamp, not an OTP proof beyond "execution with that reference completed in this session". Optional corroboration only; not required.

## Service-account / `client_credentials` identification

Source path `protocol/oidc/grants/ClientCredentialsGrantType.java` [src]:
- Creates a user session for the service-account user (`service-account-<clientId>`), login method `client_auth`, **TRANSIENT** by default (persistent only if deprecated "Use refresh tokens for client credentials grant" is on) [src, doc [transient sessions](https://www.keycloak.org/docs/26.8.1/server_admin/index.html#_transient-session)].
- `azp` = requesting client's `clientId` (`initToken`: `token.issuedFor(client.getClientId())`); `sub` = service-account user id; `sid` omitted for transient sessions (`OAuth2GrantTypeBase`: "transient sessions do not add the session ID to the token") [src]. `azp`, `sid`, `exp`, `iat`, `jti`, `iss`, `typ`, `scope`, `cnf` are core claims that cannot be dropped by mappers [doc: [lightweight tokens](https://www.keycloak.org/docs/26.8.1/server_admin/index.html#_using_lightweight_access_token)].
- Dedicated default scope **`service_account`** (added by `ClientManager.enableServiceAccount`) carries session-note mappers emitting `client_id`, `clientHost`, `clientAddress` (access, ID, introspection) [src, doc [session note mappers](https://www.keycloak.org/docs/26.8.1/server_admin/index.html#_protocol-mappers_oidc-user-session-note-mappers)]. Notes `client_id`/`clientId` set only in this grant [src], so a human session never has them.
- `preferred_username` = `service-account-<clientId>` (user name, [doc]), emitted by `profile` scope `username` mapper; absent if the scope is not assigned. Changes if client is renamed [src `ClientManager`]. Introspection adds `username` and `client_id` itself regardless [src].
- `acr`: `acr` scope is a realm default, so service-account token likely has `acr` = `1`/mapped (NO_LOA -> non-SSO -> 1) [src, unverified]. `auth_time`: absent. `amr`: absent unless mapper added.

Safest discriminator ranking:
1. `azp` (core, immutable by mappers, authoritative for "which client obtained this token") in the configured allowlist. Never trust request-supplied identity.
2. Corroborate it is a client-principal token: `client_id` claim present and `== azp` (session-note based, only emitted for service-account sessions), and/or `preferred_username == "service-account-" + azp`. These depend on scope assignment; verify in imported realm JSON (see checks).
3. Do **not** use: absence of `sid` (default-only; flips with deprecated refresh switch), `acr` absence/value, `sub` shape.
4. Hardening: allowlisted integration clients should have standard flow, implicit, and direct access grants disabled so no human token can carry their `azp`. Without that, `azp` alone cannot separate human vs client principal on that client.
- `client_credentials` does not grant bypass by itself (allowlist required) - consistent with spec.
- Service accounts get roles via Service account roles tab (not user role mapping UI) [doc]; client roles appear in `resource_access.poc-app.roles` through `roles` scope.

## `sid`

| Token | `sid`? | Source |
| --- | --- | --- |
| Access token (human) | yes, always (core claim in `initToken`, `setSessionId(userSession.getId())`) | [src `TokenManager.initToken`] |
| ID token | yes, copied from access token | [src `generateIDToken`] |
| Refresh token | yes (`RefreshToken` copies `sessionId`); `aud`=realm issuer, original aud in `aud_x`; treat as opaque, API never parses | [src `RefreshToken.java`] |
| Introspection response | yes (basic claim copied by `getAccessTokenFromStoredData`) | [src `AccessTokenIntrospectionProvider`] |
| Logout Token | only if client "Backchannel logout session required" is on | [src `DefaultTokenManager.initLogoutToken`; doc] |
| Service-account access token | no (transient default) | [src, doc] |

- Format since 26.5.0: opaque 24-char random base64 string, not UUID; spec says opaque. Compare as string only, scoped by issuer [doc [26.5.0 migration](https://www.keycloak.org/docs/26.8.1/upgrading/index.html#session_state-and-sid-are-no-longer-uuids), spec].
- `session_state` claim removed from tokens in 25.0.0 (kept only in token endpoint JSON response); the "Session State" mapper can restore it. Use `sid`, not `session_state` [doc [25.0.0](https://www.keycloak.org/docs/26.8.1/upgrading/index.html#removed-session_state-claim)].
- Stability across refresh: refresh mints a new access token via `createClientAccessToken` with the **same** `userSession` -> same `sid` [src `AbstractRefreshTokenProvider`]. Not changed by lightweight tokens: `sid` is in the non-removable list [doc]. Lightweight tokens are not planned for the PoC; the `basic`/`acr` mappers also have "add to lightweight access token" flags (default off) [doc].
- Logout Token (back-channel): `iss`, `aud`=receiving client's clientId, `iat`, `exp` (+2 min), `jti`, `events` (`http://schemas.openid.net/event/backchannel-logout`), `sub` = user id (pairwise-adjusted), `sid` (conditional) [src `initLogoutToken`]. Only clients with a client session in the logged-out user session are notified [src `ResourceAdminManager`, unverified for expiry-driven session end]. Config: client attrs "Backchannel logout URL" and "Backchannel logout session required"; set both explicitly in realm JSON. [doc [client settings](https://www.keycloak.org/docs/26.8.1/server_admin/index.html)].
- Match rule for the API: authorization `sid` (from validated ID token of own code exchange) equals introspection `sid` equals Logout Token `sid`, same `iss`. Subject-only logout tokens (no `sid`) -> invalidate all of that `sub`'s pending authorization (spec [back-channel](https://openid.net/specs/openid-connect-backchannel-1_0.html#BCLogoutActions)).

## Introspection and audience

- Endpoint `/realms/{realm}/protocol/openid-connect/token/introspect`, confidential client auth required [doc: [oidc-layers](https://www.keycloak.org/securing-apps/oidc-layers)].
- Since **26.6.2** it returns `{"active":false}` unless the *authenticated* client is in the token's `aud` (checked on the transformed token, so lightweight-token audiences resolved at introspection work). Opt-out flags (server-wide `allow-token-introspection-without-audience-check`, per-client "Allow token introspection without audience check") are deprecated [doc [26.6.2 migration](https://www.keycloak.org/docs/26.8.1/upgrading/index.html#token-introspection-now-validates-audience-claim), src `verifyAudience`]. Refresh/offline tokens: only the `azp` client may introspect [doc].
- Active only if: signature/issuer valid, client enabled, not revoked, not-before OK, **user session valid** (`UserSessionUtil.findValidSessionForAccessToken`), user enabled; transient sessions are handled via token type (no `sid`) [src].
- Response fields (expected): `active`, `token_type`, `client_id` (= `azp`, always added), `username` (always filled), `iss`, `sub`, `typ`, `exp`, `iat`, `jti`, `scope`, `aud`, `azp`, `sid`, plus mapper claims flagged "add to token introspection": `acr` (on), `auth_time` (on), `client_id`/`clientHost`/`clientAddress` for service accounts, roles claims. **No `amr`.** [src]
- Audience sources [doc [audience](https://www.keycloak.org/docs/26.8.1/server_admin/index.html#_audience_resolve)]: Audience Resolve (in `roles` scope) adds each *other* client whose roles the user holds in the token; **skips the token's own client** [src `AudienceResolveProtocolMapper.setAudience`]. Hardcoded Audience mapper (`oidc-audience-mapper`) adds an explicit client. Disabled/removed clients never appear in `aud` (26.8.0).
- Implication: if the API introspects as `poc-app` (confidential) and roles live on `poc-app`:
  - Human tokens issued to `poc-app`: `aud` lacks `poc-app` -> `active:false` unless a hardcoded Audience mapper (e.g. on a `poc-api` audience scope or poc-app dedicated scope) adds it. Needs explicit realm config.
  - Service-account tokens from other clients with `poc-app` roles: Audience Resolve adds `poc-app` -> introspectable by `poc-app`. The no-permission service account has no roles -> no `aud` -> introspection `active:false` (a different failure than 403 if it were ever introspected).
  - Alternative: dedicated resource-server client `poc-api` for introspection credentials plus audience mapper on `poc-app` scope; separates concerns but adds a client. Decision for realm design.
- Spring Security JWT validation of `aud` is separate and still advised.

## Client scopes that emit these claims (26.8.1 realm defaults)

| Scope | Default? | Mappers relevant here |
| --- | --- | --- |
| `basic` | default | `sub` (access, introspection; ID/userinfo always have it), `auth_time` |
| `acr` | default | `acr` (access, ID, introspection) |
| `roles` | default | realm roles, client roles (`resource_access.<client>.roles`), audience resolve |
| `profile` | default | `preferred_username`, names |
| `service_account` | default on service-account clients only | `client_id`, `clientHost`, `clientAddress` |
| `web-origins` | default | `allowed-origins` |

[src `OIDCLoginProtocolFactory.java`]. `sid` and `azp` have no mapper (core). Realm JSON import: scopes and client-scope assignment must be explicit or verified; do not assume defaults attach (see checks). Prior migration notes: `sub` is a mapper since 25.0.0 and does nothing for service accounts (always added) [doc].

## OTP-completion timestamp

- No claim carries it. `iat` = issuance time, `auth_time` = last non-SSO completion (above), AMR has no time [src].
- Keycloak stores execution completion times in user-session note `authenticators-completed` (JSON `{executionId: epochSeconds}`) [src `AuthenticatorUtils.updateCompletedExecutions`]. A generic *User Session Note* mapper (note name configurable) could in principle emit it [src `UserSessionNoteMapper`], but execution ids are realm-instance-specific, so this is brittle and **[unverified]**; only worth a feasibility spike.
- Recommendation: keep ADR 0001 fallback (expiry origin = API flow initiation); add `auth_time >= initiation - skew` and `acr`==LoA 2 and nonce/state match as acceptance checks. Optionally record `auth_time` for audit.

## Example claims (expected, to verify at feasibility baseline)

Human UI access token, after step-up (poc-app, confidential + PKCE, LoA 2). Redacted shape only.

| Claim | Expected |
| --- | --- |
| `iss` | `https://<kc>/realms/<realm>` |
| `azp` | `poc-app` |
| `aud` | `poc-app` only if hardcoded Audience mapper added; else absent or other resource clients |
| `sub` | user UUID |
| `sid` | 24-char opaque string |
| `typ` | `Bearer` |
| `acr` | `2` (or mapped name, e.g. `gold`) |
| `auth_time` | epoch secs of last interactive auth |
| `iat`/`exp`/`jti` | present |
| `scope` | `openid profile email` (+ `acr`/`basic`/`roles` not listed: `includeInTokenScope=false`) |
| `resource_access.poc-app.roles` | business roles |
| `preferred_username` | username |
| `amr` | absent unless AMR mapper + execution reference configured |

ID token (human): `iss`, `aud=poc-app`, `azp=poc-app`, `sub`, `sid`, `nonce` (from request), `acr`, `auth_time`, `iat`/`exp`/`jti`, `typ=ID`; `amr` only if configured. Verify `nonce` per OIDC Core [spec].

Refresh token (opaque to app): `typ=Refresh` (or `Offline`), `azp=poc-app`, `sid`, `sub`, `aud`=realm issuer, `aud_x`=original audience, `scope`, `prov`; no `acr`/`auth_time`.

Introspection response (human):

| Claim | Expected |
| --- | --- |
| `active` | `true` |
| `client_id` | `poc-app` |
| `username` | username |
| `token_type` | `Bearer` |
| `sub`, `sid`, `azp`, `iss`, `jti`, `iat`, `exp`, `scope`, `typ` | as access token |
| `aud` | must include introspecting client |
| `acr`, `auth_time` | yes |
| `amr` | no |

Service-account access token (`integration-trusted`):

| Claim | Expected |
| --- | --- |
| `azp` | `integration-trusted` |
| `client_id` | `integration-trusted` |
| `preferred_username` | `service-account-integration-trusted` |
| `sub` | service-account user UUID |
| `sid` | absent (transient default) |
| `acr` | `1` (or mapped) - likely, not absent |
| `auth_time` | absent |
| `clientHost`, `clientAddress` | caller host/IP |
| `aud` | `poc-app` (via Audience Resolve, if it holds poc-app roles) |
| `resource_access.poc-app.roles` | business roles (trusted: yes; no-permission: none) |

Back-channel Logout Token: `iss`, `aud=poc-app`, `iat`, `exp`(iat+120s), `jti`, `sub` (user id), `sid` (if session-required on), `events` with `http://schemas.openid.net/event/backchannel-logout`; token `typ` claim = `Logout` (`TokenUtil.TOKEN_TYPE_LOGOUT`) [src `LogoutToken.java`]; no `nonce` [spec].

## Realm design facts (poc-app, integration clients, roles)

- `poc-app`: set `acr.loa.map` at realm level (`{"1":1,"2":2}` or named), flow with LoA 1 (password) then LoA 2 (OTP form, Max Age 0); explicit `backchannel.logout.url` and session-required = true; ensure `acr`, `basic`, `roles` scopes assigned; add hardcoded Audience mapper if API introspects as `poc-app`.
- Service-account clients: confidential, service accounts on, standard/implicit/direct-grants off; `service_account` scope present; roles via Service account roles. Trusted/untrusted differ only by allowlist membership; no-permission has no `poc-app` roles.
- Allowlist key: `azp` (== clientId). Optional second check `client_id`==`azp`.
- Expect `acr`=`1` on service-account tokens; do not write the "acr absent" assumption into code (poc-description.md L387 lists absence only as an anti-pattern, which remains correct).

## Required runtime checks (add to feasibility list)

1. Decode real tokens (human LoA 1, LoA 2, SA trusted/untrusted/no-permission) and fill the tables above; record `acr` of service-account tokens, `aud` contents, `sid` absence.
2. Realm import: confirm `acr`, `basic`, `roles`, `service_account` scopes attach to imported clients; `client_id` claim present.
3. Tamper `acr_values=1`/omit it on step-up: token `acr` lower, API rejects. Request `claims` essential vs `acr_values`.
4. Step-up via popup: `sid` equals original session `sid` with SSO cookie; differs without cookie. Avoid `prompt=login`; observe old-session removal if used.
5. `auth_time` after step-up vs flow-initiation time; after refresh; after concurrent second tab step-up.
6. Introspection: poc-app token introspected by poc-app (expect inactive w/o audience mapper); after adding mapper; service-account token; post-logout.
7. Back-channel logout: Logout Token `sid`/`sub`/`aud`, with session-required on and off; admin logout vs browser logout vs idle expiry; deliver to API URL in container network.
8. Optional: `authenticators-completed` note mapper spike; AMR mapper output shape.

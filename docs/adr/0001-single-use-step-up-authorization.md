# Single-use step-up authorization for human sensitive operations

The API owns the operation-bound step-up challenge, Keycloak callback, and authorization-code exchange. Issuance requires a validated, correlated authentication result and atomic one-time challenge completion; accepting an arbitrary UI-submitted LoA 2 token would not establish a new verification for that operation. Protocol details require baseline validation.

Measure expiry from successful OTP verification when trustworthy factor-completion evidence is available. Otherwise, use API initiation of the correlated OTP flow as a conservative fallback: completing OTP consumes part of the configured lifetime, and callback arrival cannot extend it. This avoids depending on an unavailable factor timestamp without allowing delayed callbacks to extend authorization.

Human sensitive operations require a fresh Keycloak OTP verification and an API-managed, single-use authorization bound to the user, login session, and stable operation identifier. We rejected a reusable assurance window because each invocation must require new verification; a valid LoA 2 token alone is insufficient. Trusted service-account integrations retain their explicit exemption and the same business-permission requirements.

The authorization is consumed atomically when execution begins, even if execution fails. It expires after a configurable lifetime of two minutes by default; new verification atomically replaces any pending authorization for the same user and operation. Request payload binding is outside scope.

Check session status immediately before execution, reject inactive or unknown status, and invalidate unused authorization on validated logout notifications. We accept the race between a successful final check and execution because remote logout and local execution cannot be ordered atomically by the documented protocols. This replaces the earlier strict immediate-logout guarantee; local invalidation and consumption must still be serialized.

This commits the PoC to stateful authorization lifecycle enforcement rather than token-only assurance checks. The PoC uses one API instance with in-memory authorization state, invalidated on restart. Only built-in Keycloak capabilities are permitted; fresh verification, OTP replay prevention, verification evidence, and logout detection require feasibility validation before implementation. Unsupported requirements must be revisited explicitly.

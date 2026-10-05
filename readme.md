# Standalone PoC: Keycloak Step-Up Authentication

This repository establishes an independent, reproducible baseline for demonstrating step-up authentication with Keycloak and Spring Boot, separate from the CBS application.

Sensitive REST operations require stronger authentication for human/UI users. Explicitly trusted service-account integration clients may invoke those operations without OTP. Both types of callers must have the same required business permission. Keycloak handles OTP verification and secrets.

The baseline will include explicit versions, build/run instructions, reproducible Keycloak realm and application configuration, and authorization tests. Running it must not require access to CBS or an existing Keycloak deployment. Compatibility with CBS is a later validation step, outside this PoC’s scope.

The repository currently contains planning documentation; application implementation and baseline tooling have not yet been selected.

See [poc-description.md](poc-description.md) for scope, scenarios, and deliverables, and [CONTEXT.md](CONTEXT.md) for domain terminology.

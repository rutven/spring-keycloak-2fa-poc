# Repository Guidelines

## Project Structure & Module Organization

This repository is a standalone step-up authentication PoC. It currently contains planning documentation:

- `poc-description.md`: scope, authorization rules, test scenarios, and deliverables.
- `CONTEXT.md`: domain terminology; read it before proposing changes.

Keep future authorization logic reusable and separate from controllers. Document the chosen directory layout when implementation begins.

## Build, Test, and Development Commands

No build, test, or local-run commands are configured yet. Do not assume Maven, Gradle, Docker, or a particular Java/Kotlin version. Once tooling is selected, document exact commands and prerequisites alongside the implementation.

For documentation changes, use `git diff --check` to detect whitespace errors and `git diff -- AGENTS.md poc-description.md CONTEXT.md` to review edits.

## Coding Style & Naming Conventions

Use descriptive Markdown headings, short paragraphs, fenced examples, and repository-relative paths. Follow the vocabulary in `CONTEXT.md`, including “Standalone PoC.” No source formatter or linter is configured; establish language-specific indentation and naming conventions with the first implementation.

## Testing Guidelines

No test framework or numeric coverage threshold exists yet. Future automated tests must cover unauthenticated callers, LoA 1/2 users, missing or malformed authentication context, trusted/untrusted service accounts, and missing business permissions. Name tests after observable authorization outcomes. Separate policy unit tests from integration tests and keep Keycloak E2E checks lightweight.

## Commit & Pull Request Guidelines

The existing commit uses a concise imperative subject: `Add README for Keycloak step-up authentication PoC`. Follow that style. Track tickets in GitHub Issues. PRs should link the relevant issue, explain behavior and configuration changes, and report validation results. Include reproducible authentication-flow instructions when applicable.

## Security & Agent Instructions

Keep OTP verification and secrets in Keycloak. Require normal business permissions even for allowlisted integrations; ambiguous authentication context must fail closed. Never commit credentials or live tokens.

Before implementation, inspect actual versions and configuration and present a short plan. Use Context7 for current library documentation and codebase-memory-mcp first for code discovery. Read relevant `docs/adr/` decisions when present. Use the agreed triage labels: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, and `wontfix`.

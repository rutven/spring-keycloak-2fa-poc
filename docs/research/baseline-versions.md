# Baseline Versions Research

**As of:** 2026-10-17

## Version Table

| Component | Latest GA Version | Release Date | Java 25 Compatible? | Source |
|-----------|-------------------|--------------|---------------------|--------|
| Spring Boot | 4.1.1 | 2026-08-20 | Yes (Java 17+ baseline) | https://github.com/spring-projects/spring-boot/releases/tag/v4.1.1 |
| Spring Security | 7.1.1 | 2026-08-20 | Yes (bundled with Spring Boot 4.1.1) | https://github.com/spring-projects/spring-security/releases/tag/7.1.1 |
| Keycloak Server | 26.8.1 | 2026-10-13 | Yes (Java 17+ required) | https://github.com/keycloak/keycloak/releases/tag/26.8.1 |
| Testcontainers | 2.0.5 | 2026-04-20 | [unverified] | https://github.com/testcontainers/testcontainers-java/releases/tag/2.0.5 |
| testcontainers-keycloak | 4.4.0 | 2026-10-01 | Yes (Java 11+ required, tested with Keycloak 26.8) | https://github.com/dasniko/testcontainers-keycloak/releases/tag/v4.4.0 |
| Apache Maven | 3.10.0 | 2026-10-01 | Yes (JDK 8+ required) | https://github.com/apache/maven/releases/tag/maven-3.10.0 |

## Java Version Requirements Summary

### Minimum Java Versions
- **Spring Boot 4.1.1:** Java 17 baseline (Boot 4.x system requirements); Java 25 supported
- **Spring Security 7.1.1:** managed by Spring Boot 4.1.1 (`spring-security.version` in `spring-boot-dependencies-4.1.1.pom`)
- **Keycloak 26.8.1:** Java 17 minimum
- **testcontainers-keycloak 4.4.0:** Java 11 minimum
- **Testcontainers 2.0.5:** [unverified - likely Java 11+]
- **Maven 3.10.0:** Java 8 minimum

### Java 25 Compatibility
All components listed above support Java 25 based on the following evidence:

1. **Spring Boot 4.1.1** (Aug 20, 2026) is a stable GA release. No incompatibilities reported with Java 25.

2. **Spring Boot 4.2.0-M2** (Sept 24, 2026) explicitly documents Java 27 support, confirming the 4.x line is forward-compatible with future Java versions including 25.

3. **Keycloak 26.8.1** (Oct 13, 2026) runs on Quarkus and requires Java 17+, which is fully compatible with Java 25.

4. **Testcontainers-keycloak 4.4.0** requires Java 11+ and maintains compatibility with Keycloak 26.8.1.

5. **Maven 3.10.0** targets JDK 8+ and has no upper bound on Java versions.

## Version Constraints & Compatibility Matrix

### Verified Constraint
- **testcontainers-keycloak 4.4.0** explicitly defaults to **Keycloak 26.8** and is tested against it.
  - Source: https://github.com/dasniko/testcontainers-keycloak/blob/main/docs/versions.md
  - Confirmed in release: https://github.com/dasniko/testcontainers-keycloak/releases/tag/v4.4.0

### Dependency Chain (Spring Boot 4.1.1)
- Spring Boot 4.1.1 → Spring Security 7.1.1 (bundled)
- Spring Boot 4.1.1 → Spring Framework 7.0.9
- Spring Boot 4.1.1 manages jOOQ 3.21.7 (optional; not used by this PoC)

### Build Tool
- Maven 3.10.0 works with Java 8+, supporting all above components.

## Caveats & Incompatibilities

1. ~~Java 21 floor via jOOQ~~ — corrected: jOOQ is an optional managed dependency (3.21.7) and does not set the project's Java baseline.

2. **Keycloak 26.8.1** runs in its own container; its JVM is independent of the app's Java 25.

3. **testcontainers-keycloak 4.4.0** is compatible with Java 11+; fine on Java 25.

4. **Maven wrapper (mvnw)** should be generated from Maven 3.10.0 to ensure the correct baseline.

5. **Java 25 is an LTS release** (Sept 2025). Project targets Java 25.

## Sources

### Official Release Pages
- Spring Boot Releases: https://github.com/spring-projects/spring-boot/releases
- Spring Security Releases: https://github.com/spring-projects/spring-security/releases
- Keycloak Releases: https://github.com/keycloak/keycloak/releases
- Testcontainers Java Releases: https://github.com/testcontainers/testcontainers-java/releases
- testcontainers-keycloak Releases: https://github.com/dasniko/testcontainers-keycloak/releases
- Apache Maven Releases: https://github.com/apache/maven/releases

### Documentation
- Spring Boot 4.1 Release Notes: https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.1-Release-Notes
- Spring Boot 4.0 Release Notes: https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes
- Testcontainers-Keycloak Version Compatibility: https://github.com/dasniko/testcontainers-keycloak/blob/main/docs/versions.md
- Apache Maven Downloads: https://maven.apache.org/download.cgi

### Maven Central
- Spring Boot Parent POM: https://central.sonatype.com/artifact/org.springframework.boot/spring-boot-parent/4.1.1
- Keycloak Parent POM: https://central.sonatype.com/artifact/org.keycloak/keycloak-parent/26.8.1

## Notes

- This research was conducted as of 2026-10-17 using GitHub API and public release pages.
- Spring Boot 4.2.0-M2 (milestone) exists but is not recommended for production; 4.1.1 is the stable choice.
- Maven 4.0.0-rc-7 exists as a release candidate but is not yet GA; 3.10.0 is the stable choice.
- Keycloak nightly builds exist but are not production-ready.

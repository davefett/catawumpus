# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Quarkus 3.31.4 service targeting **Java 25**, built with Maven. All I/O is non-blocking: the REST layer uses Quarkus REST (formerly RESTEasy Reactive), persistence uses Hibernate Reactive with Panache, and the database driver is the Vert.x reactive PostgreSQL client.

- Group ID: `com.ebrdesign.catawumpus`
- Root package: `com/ebrdesign/catawumpus`

## Commands

```bash
# Dev mode (hot reload, DevServices starts PostgreSQL automatically via Docker)
./mvnw quarkus:dev

# Compile and run unit tests
./mvnw test

# Run a single test class
./mvnw test -Dtest=MyTestClass

# Run integration tests (requires a running app or DevServices)
./mvnw verify -DskipITs=false

# Package (fast-jar, output: target/quarkus-app/quarkus-run.jar)
./mvnw package

# Native build (requires GraalVM or Docker)
./mvnw package -Dnative
./mvnw package -Dnative -Dquarkus.native.container-build=true
```

## Design conventions

All architectural change plans must be written to the `./design/` directory as markdown files before implementation begins.

## Development conventions

All features must be developed TDD-style: write the unit tests first, confirm they fail, then implement until they pass.

Quarkus configuration changes (adding/removing extensions, updating platform version) must be made via the Quarkus CLI (`quarkus ext add`, `quarkus ext remove`, `quarkus update`) rather than editing `pom.xml` directly. The CLI is available at `~/.sdkman/candidates/quarkus/current/bin/quarkus`.

## Architecture

### Reactive-first throughout
Every layer must remain non-blocking. REST endpoints return `Uni<T>` or `Multi<T>` (Mutiny types). Panache entities and repositories use `PanacheEntity` / `PanacheRepository` from `quarkus-hibernate-reactive-panache`, and all database calls return Mutiny reactive types. Never block the event loop (no `.await().indefinitely()` in production paths).

### REST layer (`quarkus-rest-jackson`)
- Annotate resources with `@Path`, `@GET`, `@POST`, etc. from `jakarta.ws.rs`
- Jackson handles JSON serialization; customize with `@JsonProperty`, `ObjectMapper` beans, or `JacksonCustomizer`
- Annotate resources and methods with SmallRye OpenAPI annotations (`@Operation`, `@APIResponse`, `@Schema`) to populate the spec

### Persistence (`quarkus-hibernate-reactive-panache` + `quarkus-reactive-pg-client`)
- Entities extend `PanacheEntity` (auto `id`) or `PanacheEntityBase` (custom id)
- All Panache operations are reactive: `Entity.persist()`, `Entity.findById()`, etc. return `Uni<>`
- Database mutations must run inside a reactive transaction: wrap with `Panache.withTransaction(() -> ...)`

### OpenAPI
- Spec served at `GET /q/openapi`
- Swagger UI available at `GET /q/swagger-ui` in dev mode only
- Top-level API metadata is in `application.properties` under `quarkus.smallrye-openapi.*`

### Configuration profiles
| Profile | DB schema | Swagger UI | Keycloak | OTel |
|---------|-----------|------------|----------|------|
| `dev`   | `drop-and-create` | enabled | DevServices container | SDK disabled |
| `test`  | `drop-and-create` | disabled | DevServices container | SDK disabled |
| `prod`  | `validate` | disabled | env vars | OTLP export enabled |

Production env vars: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`, `KEYCLOAK_URL`, `KEYCLOAK_REALM`, `KEYCLOAK_CLIENT_ID`, `KEYCLOAK_CLIENT_SECRET`, `OTEL_EXPORTER_OTLP_ENDPOINT`.

### Security (`quarkus-oidc` + `quarkus-keycloak-authorization`)
All endpoints are secured via Bearer token (JWT) validated against Keycloak. Use `@RolesAllowed`, `@Authenticated`, or `@PermissionsAllowed` on resources. Fine-grained policies are enforced via Keycloak Authorization Services (`quarkus.keycloak.policy-enforcer.enable=true`).

In `dev` and `test` profiles, DevServices automatically starts a Keycloak container seeded from `src/main/resources/keycloak-realm-dev.json`. Production Keycloak is configured via env vars: `KEYCLOAK_URL`, `KEYCLOAK_REALM`, `KEYCLOAK_CLIENT_ID`, `KEYCLOAK_CLIENT_SECRET`.

**Important:** `quarkus.oidc.auth-server-url` must not be set in `dev`/`test` — its presence disables Keycloak DevServices entirely. It is scoped to `%prod` only.

The Keycloak policy enforcer intercepts every request, including Quarkus-internal `/q/*` paths, and will error if those paths are not registered as resources in Keycloak. Always add `enforcement-mode=DISABLED` path entries for `/q/health/*`, `/q/metrics/*`, and `/q/openapi` in all profiles, and for all of `/q/*` in `dev`.

### OpenTelemetry (`quarkus-opentelemetry`)
Traces, metrics, and logs are exported via OTLP. The collector endpoint defaults to `http://localhost:4317` and is overridden in production via the `OTEL_EXPORTER_OTLP_ENDPOINT` env var. The OTel SDK is disabled entirely in `dev` and `test` profiles (`quarkus.otel.sdk.disabled=true`) — no collector is required locally.

### DevServices
In `dev` and `test` profiles, Quarkus DevServices automatically provisions a PostgreSQL container — no local database setup is needed. Docker must be running.

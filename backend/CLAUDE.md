# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Java 17, Gradle wrapper. Spring Boot 2.6 + MyBatis + Netflix DGS (GraphQL) + Flyway + SQLite.

- Run app: `./gradlew bootRun` (http://localhost:8080/tags; creates `dev.db` SQLite file in the repo root)
- All tests (what CI runs): `./gradlew clean test`
- Single test class: `./gradlew test --tests io.spring.api.ArticleApiTest`
- Single method: `./gradlew test --tests 'io.spring.api.ArticleApiTest.someMethod'`
- Format (Google Java Format via Spotless): `./gradlew spotlessApply` / `./gradlew spotlessCheck`
- Docker image: `./gradlew bootBuildImage --imageName spring-boot-realworld-example-app`
- `./gradlew clean` also deletes `./dev.db`.


## Architecture

Implements the RealWorld (Conduit) spec over **both REST and GraphQL** as adapters on one domain layer. Package root `io.spring`, organized DDD/CQRS-style:

- `core` — write-side domain: entities (`article`, `comment`, `user`, `favorite`), repository interfaces, and service interfaces (`core/service`). No framework/persistence details.
- `infrastructure` — implementations: `repository` (implements core repository interfaces on top of MyBatis mappers), `mybatis/mapper` (write-side mappers), `mybatis/readservice` (read-side queries), `service` (e.g. JWT service), plus MyBatis type handlers.
- `application` — read side (CQRS query services) returning DTOs from `application/data`, and command-ish services like `user`/`article` that wrap use cases. Read services return DTOs directly from SQL rather than via core entities.
- `api` — Spring MVC REST controllers, `api/security` (Spring Security config + JWT filter), `api/exception` (error handling).
- `graphql` — DGS data fetchers/mutations and exception handling; schema in `src/main/resources/schema/schema.graphqls`. The `com.netflix.dgs.codegen` plugin generates types into package `io.spring.graphql` from that schema during `generateJava`, so edit the schema, not generated code.

MyBatis SQL lives in XML under `src/main/resources/mapper/*.xml` (mapper interfaces in `infrastructure/mybatis`); `map-underscore-to-camel-case` is on. Schema is managed by Flyway migrations in `src/main/resources/db/migration`.

Config (`application.properties`): JWT secret/session time (`jwt.*`), SQLite datasource, and `spring.jackson.deserialization.UNWRAP_ROOT_VALUE=true` (REST request bodies are wrapped in a root key like `{"user": {...}}`).

## Tests

Under `src/test/java/io/spring`: `api/` tests are REST-Assured MockMvc controller tests (`TestWithCurrentUser` provides an authenticated-user base); `infrastructure/` tests extend `DbTestBase` for MyBatis/DB tests; `application/` and `core/` cover services and domain logic.

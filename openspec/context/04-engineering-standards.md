# Engineering Standards — Kotlin Spring RealWorld API

> Scope: `kotlin-spring-realworld-example-app` at repository HEAD `848c29e`.
>
> This document separates the conventions already visible in the code from the rules that should govern new or modified code. The goal is to preserve the existing API while making security, persistence, and test behavior explicit. It does not authorize a broad refactor as part of an unrelated feature.

## 1. Normative language

- **MUST** — a change is not acceptable without following the rule or documenting an approved exception.
- **SHOULD** — the default choice; deviations need a concrete reason in the change description.
- **MAY** — an optional technique that fits the existing stack.
- **Current deviation** — behavior observed in the repository that should not be copied into new code without an explicit decision.

## 2. Build and compatibility standards

### 2.1 Authoritative toolchain

The project is a single Maven module. Use the Maven wrapper and the versions already declared in `pom.xml`:

- Spring Boot parent: `3.0.6`;
- Kotlin: `1.8.21`;
- Java runtime/property: `17`;
- Kotlin compiler `jvmTarget`: currently `11`.

New code MUST remain compatible with the Spring Boot 3 / Jakarta namespace generation. Use `jakarta.persistence.*` and `jakarta.validation.*`; do not reintroduce `javax.*` imports.

The Java/Kotlin target mismatch is a current deviation. Before changing language features, bytecode assumptions, or deployment tooling, align the Kotlin compiler target with the intended Java 17 runtime (or document why target 11 is deliberately retained) and verify the complete Maven build.

The project is Spring MVC plus blocking JPA. New request-path code MUST NOT assume WebFlux, Reactor, R2DBC, non-blocking I/O, or coroutine-based persistence. A future reactive migration must be a separately designed boundary change, not a dependency added to one handler.

### 2.2 Dependency management

- The Spring Boot parent is the primary authority for Boot-managed dependency versions.
- A manually pinned version MUST have a documented compatibility reason. This applies in particular to Jackson, Kotlin, Feign, JWT, and test libraries.
- Do not add a second web stack, persistence stack, or security framework without checking for classpath and runtime-mode conflicts.
- Keep test-only dependencies scoped as test dependencies.
- Do not rely on the POM description when choosing APIs; verify actual dependencies and imports.

### 2.3 Required verification

For every code change, run the narrowest relevant checks and then the full project check when possible:

```text
./mvnw test
```

A change that affects compilation, dependency versions, serialization, or startup MUST also be verified with a clean compile/package path appropriate to the environment. If Java or dependency resolution is unavailable, report that limitation instead of treating the change as verified.

## 3. Package boundaries and dependency direction

The existing dependency direction is:

```text
web  ->  service  ->  repository  ->  model
 |          |
 +------> model.inout

jwt  ->  service/repository
client -> external HTTP contract (test support today)
exception -> shared error types
```

### Rules

1. HTTP mapping, input binding, status selection, and response envelopes belong in `io.realworld.web`.
2. Workflows that coordinate multiple repositories, perform authorization decisions, or contain more than simple mapping SHOULD live in `io.realworld.service`.
3. Repository interfaces and JPA specifications belong in `io.realworld.repository`; persistence predicates MUST NOT be hidden inside output mappers.
4. `io.realworld.model` is persistence/domain state. Public API models belong in `io.realworld.model.inout`.
5. New production outbound integrations MUST have an explicit adapter boundary. The existing `io.realworld.client` Feign interfaces are test clients; do not assume they are Spring-managed application clients.
6. New code MUST use constructor injection, matching the constructors already used by handlers and `UserService`. Avoid field injection and mutable service dependencies.
7. Do not create a new architectural layer, shared utility, or abstraction for a one-off operation. If a workflow is repeated, extract it at the boundary where its responsibility is clear.
8. Package names and class names MUST communicate the current role. A class named `*Handler` is still an HTTP controller; do not place persistence or transport clients there merely because the class exists.

## 4. Kotlin, Spring, and JPA standards

### 4.1 Kotlin/Spring openness

The Maven Kotlin plugin enables the Spring all-open compiler plugin. Spring-managed classes and advice MUST remain compatible with proxying. When adding `@Transactional`, caching, async, retry, or other proxy-based annotations:

- place the annotation on a method that can be intercepted;
- avoid self-invocation when the behavior depends on a proxy;
- verify that the relevant class/method is open through the configured compiler plugin;
- add a test that proves the behavior, rather than only checking that the annotation compiles.

### 4.2 Entity design

Existing entities are Kotlin `data class`es with mutable relationships. This is a current deviation and MUST NOT be copied into new entities without a persistence-specific review.

For new or substantially changed entities:

- prefer identity-based equality semantics that do not include mutable associations;
- do not expose entity instances as new public API contracts;
- provide a safe persistence construction path and verify Hibernate proxy/no-argument requirements;
- make relationship ownership, `fetch`, `cascade`, and orphan behavior explicit when the default is not intentional;
- keep collections mutable only where the aggregate owns the mutation;
- avoid placeholder entities such as `User()` as a representation of an authenticated or missing user;
- define indexes and uniqueness constraints for fields used as identifiers or lookup keys, especially email, username, slug, and tag name.

DTO mappers MUST calculate presentation state (`following`, `favorited`, and counts) without mutating entities or triggering uncontrolled lazy-loading outside a transaction.

### 4.3 Time and identifiers

- Persist `OffsetDateTime` consistently with the existing entities.
- Serialize API timestamps as ISO date-time strings normalized to UTC, matching `model.inout.Article` and `model.inout.Comment`.
- Slugs MUST remain unique and stable unless a title change intentionally generates a new slug. Collision handling must be deterministic enough to test.
- Generated database IDs are persistence identifiers; they are not API-level authorization or business identifiers.

## 5. HTTP and serialization standards

### 5.1 Routes and envelopes

- API routes MUST use the `/api` prefix.
- Resource names and verbs SHOULD follow the existing RealWorld contract.
- Response envelopes MUST remain stable: `user`, `profile`, `article`, `articles` plus `articlesCount`, `comment`, `comments`, `tags`, and `errors` are public keys.
- New request models MUST use the appropriate `@JsonRootName` and be tested with `spring.jackson.deserialization.UNWRAP_ROOT_VALUE=true` in effect.
- New response models SHOULD be separate from JPA entities. Do not expose passwords, relationship internals, or newly added sensitive fields by relying on accidental Jackson defaults.
- Avoid returning untyped `Any` for new endpoint contracts when a typed response model can express the shape. Existing `Any` handlers are compatibility debt, not a reason to expand it.

### 5.2 Authentication header

Until the authentication mechanism is deliberately migrated, the public header contract is:

```text
Authorization: Token <jwt>
```

A change MUST NOT silently switch to `Bearer`, change wrapper names, or alter anonymous-versus-required behavior. If a compatibility migration is needed, support and test both contracts during an explicitly defined transition.

### 5.3 Validation

- Request bodies with constraints MUST use `@Valid`.
- Validation errors MUST be checked through the project’s `InvalidRequest.check(errors)` path or a consciously standardized replacement.
- New validation failures MUST preserve HTTP 422 and the `{ "errors": { "field": ["message"] } }` shape unless the API contract is intentionally versioned.
- Every mutable field in an update request MUST have an explicit absent-versus-empty policy. `null`, an omitted property, and `""` must not accidentally mean the same thing.
- `UpdateArticle` currently relies on manual checks and is not annotated with `@Valid`; new update models SHOULD use declarative constraints where possible and cover partial-update semantics with tests.
- Error responses MUST not expose stack traces, SQL, JWT contents, passwords, or internal implementation details.

### 5.4 Status codes and missing resources

Use the existing exception/status convention for compatible endpoints:

| Situation | Status |
| --- | --- |
| Invalid request fields | `422 Unprocessable Entity` |
| Missing resource | `404 Not Found` |
| Authenticated user lacks ownership/permission | `403 Forbidden` |
| Missing or invalid required token | `401 Unauthorized` |
| Successful deletion | `200 OK` in the current contract |

A new error path SHOULD go through the centralized advice where possible. Direct response writing is currently limited to the authorization aspect and should not spread to regular handlers.

## 6. Security standards

The current custom AOP authenticator is a sensitive boundary. Changes to it require focused tests and a security review.

### 6.1 Secrets and token handling

- Secrets, signing keys, and credentials MUST NOT be committed as literal values in `application.properties` or source code.
- Load `jwt.secret` from an environment, deployment secret, or profile-specific external configuration; provide only a safe local-development mechanism.
- Tokens MUST NOT be written to logs, exception messages, response headers, analytics, or test output.
- Passwords MUST be hashed with the established bcrypt path and MUST never be persisted or serialized in plaintext.
- JWT validation MUST verify signature, issuer, subject/user binding, and expiration. Any algorithm or claim change requires compatibility tests.
- Token persistence and rotation semantics MUST be documented when changing login or logout behavior.

The current repository violates the first and third rules: it contains a literal JWT secret and logs the supplied token on lookup failure. Those are remediation items, not patterns to reproduce.

### 6.2 Request-local identity

If `ThreadLocal<User>` remains in use:

- set the identity only after complete token validation;
- clear it in a `finally` block for every path after setting it, including handler exceptions;
- never use a mutable empty entity as an authorization decision;
- add tests for anonymous, missing-token, malformed-token, expired-token, and handler-exception paths.

A future migration to Spring Security should preserve endpoint semantics and the `Token` header during the transition, then remove the aspect/interceptor workaround only after contract tests pass.

### 6.3 CORS and transport

CORS MUST be restricted to known origins, methods, and headers for non-local environments. The current `allowedOrigins("*")`, `allowedMethods("*")`, and `allowedHeaders("*")` configuration is suitable only as an explicitly acknowledged local/demo default. Do not enable credentials with wildcard origins.

## 7. Persistence and transaction standards

### 7.1 Schema ownership

The application currently uses H2 in-memory storage and has no Flyway/Liquibase or SQL migration directory. Therefore:

- a restart is expected to lose data;
- H2 behavior MUST NOT be treated as proof of production-database compatibility;
- any move to a persistent database MUST add a migration and a database-specific integration test before deployment;
- schema changes MUST be backward-compatible with the application version overlap used during rollout.

### 7.2 Transactions and aggregate changes

A service method SHOULD define the transaction boundary for a workflow that performs multiple writes. This includes article creation/update with tags, follow/unfollow, favorite/unfavorite, comment deletion plus article checks, and token rotation.

Repository methods SHOULD remain focused on persistence operations. Do not use a sequence of unrelated repository calls in a controller when partial completion could leave inconsistent state. If manual deletion is retained, document why cascade/orphan removal is not used and test rollback behavior.

### 7.3 Query and performance behavior

- Every endpoint that accepts pagination MUST define whether `offset` means an item offset or a page number and test the contract. The current article implementation passes it as a page number.
- Do not expose an unbounded `findAll()` result on a growth-sensitive endpoint without a deliberate limit.
- Article filters and feed queries MUST have tests for empty follow lists, unknown tag/author/favorited users, ordering, and page boundaries.
- Watch for N+1 queries when mapping authors, tags, favorites, and comments. Choose fetch plans or projections deliberately rather than changing all relationships to eager loading.
- Uniqueness-sensitive writes (user registration, slug generation, tag creation) MUST be protected by database constraints and handle a race between existence check and insert.

## 8. Error handling and observability

- Exceptions MUST express a client-visible outcome without embedding secrets or infrastructure details.
- `InvalidRequestHandler` remains the compatibility point for field validation errors.
- Unexpected exceptions SHOULD be logged once at an appropriate boundary with a correlation/request identifier, then returned as a generic error response.
- Logs MUST not include `Authorization` headers, raw JWTs, passwords, or full request bodies containing credentials.
- Use parameterized logging rather than string interpolation for values that may contain user input.
- Authentication failures SHOULD be observable as counts and status outcomes, not as credential values.
- Configuration changes affecting authentication, CORS, database connectivity, or serialization MUST be visible in startup diagnostics without printing secret values.

The repository currently has no explicit observability module, actuator configuration, tracing setup, or correlation-ID convention. A new observability mechanism should be added intentionally and consistently rather than partially in individual handlers.

## 9. Testing standards

### 9.1 Test levels

The current test suite is a full-context HTTP test. Maintain that valuable contract test, but add the smallest useful test level for each change:

| Change | Minimum useful coverage |
| --- | --- |
| Pure mapper or slug/date logic | Unit test |
| Repository query/specification | JPA/repository integration test |
| Validation, envelope, status, serialization | MVC/controller contract test |
| Authentication aspect or ownership rule | Focused security/AOP test plus an HTTP contract case |
| Multi-repository workflow | Service test with transaction/rollback coverage plus an integration case |
| Schema or database configuration | Integration test against the intended database engine |
| End-to-end API behavior | Existing random-port Spring Boot test or equivalent |

### 9.2 Framework consistency

The current test class uses JUnit 4 annotations (`@RunWith`, `@Before`, `@Test`) and an explicit JUnit 4 dependency, while `spring-boot-starter-test` also brings the modern Spring test stack. The module SHOULD choose one JUnit generation deliberately. Until a migration is made, new tests MUST follow the established runner/configuration rather than introducing a third test style in the same module.

Feign clients under `io.realworld.client` are test support. They MUST remain out of production wiring unless the project intentionally introduces a real outbound integration and its timeout/error contract.

### 9.3 Mandatory regression cases

Changes to the existing API should cover, as applicable:

- wrapped request deserialization (`user`, `article`, `comment`);
- missing, empty, malformed, expired, and valid authorization tokens;
- optional authentication with and without a token;
- 401/403/404/422 response shapes;
- duplicate email and username registration;
- password hashing and token rotation;
- article ownership and comment ownership checks;
- follow/favorite idempotency;
- article slug collisions and tag creation;
- pagination boundaries and ordering;
- UTC date formatting;
- cleanup of request-local identity when a handler throws.

Tests MUST assert behavior, not only log output or non-null values. A test that merely prints tags, as `retrieveTags` currently does, is not sufficient as regression coverage.

## 10. Configuration and operational standards

- Environment-specific values MUST be separated into profiles or external configuration; do not duplicate production secrets in the repository.
- Use typed configuration properties for related settings when configuration grows beyond the current small property set.
- Keep local H2 defaults clearly separated from persistent-database configuration.
- Configuration changes MUST include startup or integration verification for the affected profile.
- Do not claim support for a database, reactive runtime, or deployment mode that is not exercised by tests and build configuration.
- Any new background task, cache, or thread-local state MUST define lifecycle, cleanup, and failure behavior.

## 11. Change and review checklist

Before merging a change, reviewers should be able to answer “yes” to the relevant items:

### Contract

- Does the route retain the `/api` prefix and expected RealWorld envelope?
- Are root wrapping, validation, status codes, auth mode, and date format covered by tests?
- Are absent, null, and empty update values distinguished intentionally?

### Security

- Are no secrets or credentials added to source, properties, logs, or fixtures?
- Does the change preserve token validation and ownership checks?
- Is request-local identity cleared on all paths?
- Is CORS no broader than the deployment requires?

### Persistence

- Are relationship ownership, indexes, uniqueness, transactions, and fetch behavior intentional?
- Is a migration required, and if so, is it included and tested?
- Are paging and ordering semantics explicit?

### Kotlin/Spring compatibility

- Does the code use Jakarta imports and remain compatible with Spring proxies?
- Are constructor injection and package boundaries preserved?
- Does the change avoid mixing blocking JPA calls with an implied reactive design?

### Verification

- Were the relevant unit, integration, and contract tests run?
- Was the Maven build/compile checked when dependencies or configuration changed?
- If verification was blocked by the local environment, is that reported explicitly?

## 12. Current deviations and remediation order

These items are observed in the repository and should be tracked separately from feature work:

| Priority | Deviation | Evidence | Required direction |
| --- | --- | --- | --- |
| P0 | Literal JWT secret in repository | `src/main/resources/application.properties` | Move to external secret configuration and rotate the exposed value |
| P0 | Raw token appears in authentication logging | `ApiKeySecuredAspect` logs the supplied authorization value | Remove credential logging and add a safe authentication-failure signal |
| P1 | Thread-local cleanup is not guaranteed on exceptional handler paths | `ApiKeySecuredAspect.aroundSecuredApiPointcut` clears after successful `proceed()` | Use `try/finally` and test executor-thread reuse |
| P1 | Unrestricted API CORS | `ApiApplication.addCorsMappings` allows `*` origins/methods/headers | Restrict by environment and documented client origins |
| P1 | Java property and Kotlin bytecode target disagree | `java.version=17`, Kotlin `jvmTarget=11` in `pom.xml` | Align intentionally and verify packaging/runtime |
| P1 | No persistent schema/migration strategy | H2-only properties; no migration files | Add migration ownership before production persistence |
| P2 | JPA entities are mutable Kotlin data classes | `model/User.kt`, `Article.kt`, `Comment.kt`, `Tag.kt` | Move toward proxy-safe identity semantics and explicit relationship configuration |
| P2 | Business workflows live in handlers and lack explicit transactions | `UserHandler`, `ProfileHandler`, `ArticleHandler` | Move multi-write workflows into transactional services |
| P2 | Pagination contract is ambiguous/likely incorrect | `PageRequest.of(offset, limit)` in `ArticleHandler` | Define item-offset semantics and add boundary tests before changing behavior |
| P2 | Integration coverage is narrow and partly observational | Only `ApiApplicationTests`; `retrieveTags` prints output | Add focused failure, article, comment, serialization, and persistence tests |
| P2 | Version/test-stack drift is possible | Explicit Jackson/Kotlin/JUnit/Feign versions in `pom.xml` | Review against the Boot parent and choose one JUnit generation |

This backlog is intentionally separate from the standards above: documenting a rule does not silently change production behavior. Each remediation should preserve or explicitly version the public API and should be delivered with focused tests.

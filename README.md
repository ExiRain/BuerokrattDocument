# Bürokrati proovitöö

This document outlines the solution for migrating Ruuter and Resql to Java 21, addressing error handling and reliability concerns, and updating documentation for ease of onboarding and general knowledge.

Both applications are event-driven REST microservices running on Spring Boot 3.2.5:

- **Resql** — Maven-based, REST → DB query service with dynamic datasource routing. Deployed as an internal-only Kubernetes pod with no external network access. Executes pre-configured SQL queries exposed via REST endpoints.
- **Ruuter** — Gradle-based, REST → REST routing service with a GraalVM JS-based DSL engine. Deployed in two separate instances — Public Ruuter (external/UI-facing) and Private Ruuter (back-office/internal orchestration) — each with its own set of DSL files. Both are the same codebase with different configurations and DSL directories.

Both were originally built on Java 11, then upgraded to Java 17 without thorough migration review. This document covers the next step to Java 21 alongside architectural improvements to error handling, reliability, and documentation.

The deployment architecture follows a trust boundary model: external traffic reaches Ruuter, which validates requests via guard DSL files (cookie authentication or nonce validation), evaluates DSL routing rules, and forwards calls to internal services. Resql and other internal services (DataMapper, etc.) are only reachable from within the Kubernetes cluster, operating with permissive security (`permitAll()`) by design.

### General Code Hygiene

During the code review, several instances of hardcoded values and unused imports were identified across both projects. These should be addressed as part of the migration effort:

- Hardcoded values (datasource names, paths, timezone defaults) should be externalized to `application.yml` configuration to support environment-specific overrides without code changes.
- Unused imports and dead code should be cleaned up to reduce confusion for developers onboarding to the codebase.
- These are minor but compound issues that affect maintainability. A single cleanup pass during the migration is the most efficient time to address them.

---

## 1. Migration from Java 17 to Java 21

### 1.1 Risks of Java 21 Adoption

#### Shared Risks

- **Stronger encapsulation** — Java 21 further tightens access to internal JDK APIs. Any `--add-opens` or `--add-exports` JVM flags introduced during the Java 11→17 upgrade need to be audited. These flags may exist in Dockerfiles, startup scripts, or CI pipelines. They still function on Java 21 but mask underlying issues that should be resolved properly.
- **Behavioral changes** — Minor `HashMap`/`HashSet` iteration order differences across JDK builds can cause inconsistent results in code or tests that depend on iteration order. This is relevant for Ruuter's DSL evaluation where `LinkedHashMap` is used in some paths but standard `HashMap` in others within `ScriptingHelper`.
- **Virtual threads caveat** — Java 21 introduces virtual threads as a production-ready feature. Both projects use thread-local patterns (`DataSourceContextHolder` in Resql, request context passing in Ruuter) that would behave unexpectedly with virtual threads. Adopting virtual threads is optional and not part of this migration, but should be noted for future architecture decisions. Spring Boot 3.2.5 does not enable virtual threads by default.
- **Framework-level risk is minimal** — Spring Boot 3.2.5 officially supports Java 21. No Spring-specific breaking changes are expected from the JDK version bump.

#### Codebase Health Observations

During the analysis, both projects were found to have test coverage concerns that directly affect migration confidence. Resql has a small but functional integration test suite (`QueryControllerIntegrationTest` with MockMvc assertions) that can serve as a validation baseline. Ruuter, however, has no functional test coverage — test execution is disabled in the Gradle build configuration (`exclude '**/*'`), and the test methods themselves are commented out with `//TODO: tests are currently broken`. The test classes exist (`HttpHelperTest`, `DslMappingHelperTest`, `ExternalForwardingHelperTest`, `ScriptingHelperIT`, `DslServiceIT`, and others), but their implementations need to be restored before they can provide any validation. As part of the migration effort, restoring and stabilizing Ruuter's test suite should be treated as a prerequisite — not an afterthought — since it is the only automated way to verify that DSL evaluation, script execution, and HTTP forwarding behave correctly after the JDK upgrade.

#### Resql-Specific Risks

| Risk | Severity | Detail |
|------|----------|--------|
| `id-log` library (system-scoped) | Critical | Compiled with **Java 11 bytecode**, contains `javax.servlet` imports incompatible with Spring Boot 3.x / Jakarta EE 9+. Code review of `RestConfiguration.java` confirms both `LogHandler` and `GenericHeaderLogHandler` are **actively registered as interceptors** via `addInterceptors()` and execute on every request — these are not dormant. The library uses Maven `system` scope with a hardcoded `systemPath` — no version resolution, no conflict detection, filesystem-path-dependent. |
| Spring Cloud BOM (2021.0.2) | Low | Targets Spring Boot 2.6.x/2.7.x. Full codebase search confirmed zero usage: no imports, no config properties, no annotations. Dead dependency — safe to remove. |
| PostgreSQL driver (42.3.9) | Medium | Functions on Java 21 but contains known CVEs. Significantly behind latest stable (42.7.x). |
| OpenTelemetry BOM scope | Low | Declared with `runtime` scope, preventing version management. Should be in `dependencyManagement` without scope. |
| Dynamic datasource routing | None | Standard `AbstractRoutingDataSource` with thread-local lookup. No behavioral changes between Java 17 and 21. |

#### Ruuter-Specific Risks

| Risk | Severity | Detail |
|------|----------|--------|
| GraalVM JS Engine (23.0.1) | Medium | Uses JSR-223 SPI in interpreter-only mode. `javax.script.*` imports are JDK-native — safe. Version targeting risk: `23.0.x` targets JDK 20; JDK 21 needs `23.1.x` or `24.x`. Critical because `ScriptingHelper` is the core DSL engine — every request flows through `engine.eval()`. Fix: bump version if smoke test fails. |
| Apache HttpClient 4.5.13 | Medium | EOL. Functions on Java 21 but receives no patches. Replacement: `httpclient5`. Central for `ExternalForwardingHelper` and `HttpHelper`. |
| WireMock `wiremock-jre8` (2.35.1) | Medium | Deprecated variant. Java 21 requires `org.wiremock:wiremock:3.x` (new group ID). Test infrastructure only. |
| Mockito version conflict | Low | `mockito-inline:4.9.0` alongside `mockito-core:5.6.0`. Mockito 5.x includes inline mocking — remove `mockito-inline`. |
| All tests non-functional | High | Tests are both excluded in Gradle (`exclude '**/*'`) and commented out at source level (`//TODO: tests are currently broken`). No safety net. Must be addressed before migration. |
| Missing Java version target | Medium | No `sourceCompatibility`/`targetCompatibility`. Bytecode version depends on build-time JDK — implicit and fragile. |

---

### 1.2 Impact on Spring/Jakarta Stack

The `javax` → `jakarta` namespace migration is **already complete** in both projects.

**Resql**: All servlet imports use `jakarta.servlet`. The only `javax` is `javax.sql.DataSource` — JDK-native (`java.sql` module), not subject to rename.

**Ruuter**: `javax.script.*` imports in the scripting layer are JDK-native (`java.scripting` module) — will never be renamed.

Java 21 introduces no new Jakarta EE requirements. Spring Boot 3.4.x offers virtual thread auto-config but upgrading is optional.

**Exception**: The `id-log` library in Resql contains `javax.servlet` classes and is actively registered in `RestConfiguration.addInterceptors()`. Solution: rewrite `GenericHeaderLogHandler` and `LogHandler` (~50–100 lines) in-project using `jakarta.servlet` imports. Remove the `id-log` dependency. This eliminates the namespace incompatibility, the Java 11 bytecode mismatch, and the fragile system-scoped dependency.

---

### 1.3 Backward Compatibility

#### API Contract Level

REST API contracts are unaffected — external consumers will not notice any difference. The JDK upgrade is a runtime-level change that does not alter either service's public interface. Ruuter DSL configurations calling Resql endpoints continue to work without modification.

#### Runtime Environment Alignment

Both services run as Docker images in Kubernetes pods. The JDK version must be consistent across three layers: build pipeline, Docker base image, and build configuration files. A mismatch — such as compiling with JDK 17 but targeting 21 in config — can cause subtle runtime failures despite clean compilation.

Resql produces a WAR artifact but runs via Spring Boot's embedded Tomcat inside the container — no external servlet container is involved. Ruuter uses a multi-stage Docker build with an exploded JAR layout running via classpath entry point. In both cases, the only runtime dependency is the JDK in the base image.

#### Build Configuration

Both projects must explicitly declare Java 21 as the source and target to ensure bytecode compatibility and build reproducibility.

**Resql** — update `pom.xml`:
```xml
<java.version>21</java.version>
<maven.compiler.source>21</maven.compiler.source>
<maven.compiler.target>21</maven.compiler.target>
```

**Ruuter** — add to `build.gradle` (currently missing — bytecode version depends on whichever JDK runs the build):
```groovy
java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}
```

#### Docker Images and CI Pipeline

Both Dockerfiles already use `eclipse-temurin:21-jdk-alpine` as the base image — no container-level changes are needed. Both `docker-compose.yml` files should be reviewed for any hardcoded JDK version references in build arguments or environment variables.

CI pipelines must target JDK 21 for compilation and testing to match runtime. Ruuter's Node.js tooling and Gradle/Maven wrapper scripts are unaffected.

#### Dependency Compatibility

| Dependency | Project | Current | Java 21 Status | Action |
|-----------|---------|---------|----------------|--------|
| Spring Boot | Both | 3.2.5 | Officially supported | None |
| PostgreSQL Driver | Resql | 42.3.9 | Works, has CVEs | Upgrade to 42.7.x |
| GraalVM JS | Ruuter | 23.0.1 | Likely works, targets JDK 20 | Smoke test; bump if needed |
| Apache HttpClient | Ruuter | 4.5.13 | Works, EOL | Migrate to httpclient5 |
| WireMock | Ruuter | 2.35.1 | Deprecated variant | Migrate to wiremock 3.x |
| Mockito | Ruuter | 5.6.0 + inline 4.9.0 | Version conflict | Remove mockito-inline |
| id-log | Resql | 1.0.0-SNAPSHOT (J11) | Incompatible | Rewrite and remove |

---

### 1.4 How to Test for Migration Problems

Migration from Java 17 to 21 can introduce problems at multiple levels — compilation, dependency resolution, and runtime behavior. Each level requires a different testing approach. Notably, both services already run on JDK 21 at the container level (`eclipse-temurin:21-jdk-alpine`), while build configurations still target Java 17. This means some migration problems may already be silently present but masked by the bytecode version mismatch.

**Static Analysis**

The `jdeps` tool (included with the JDK) scans compiled bytecode for usage of internal JDK APIs that may have been removed or further restricted between versions. Running `jdeps --jdk-internals` against all project artifacts and third-party JARs identifies classes relying on `sun.misc.*`, `com.sun.*`, or other internal APIs before any code changes are made. This is especially important for dependencies not managed through Maven/Gradle (such as Resql's system-scoped `id-log` library), where version management and compatibility are not tracked automatically.

**Dependency Tree Verification**

Maven's `dependency:tree` and Gradle's `dependencies` task expose the full resolved dependency graph, including transitive dependencies. After any version change — whether upgrading a library, removing a dead dependency, or replacing an incompatible one — the tree should be re-examined for conflicting versions, duplicate classes, and unexpected transitive pulls. Conflicts at this level manifest as `NoSuchMethodError` or `ClassNotFoundException` at runtime despite clean compilation, making them particularly difficult to diagnose without proactive verification.

**Automated Test Suites**

Automated tests are the most reliable way to detect behavioral regressions — cases where code compiles and dependencies resolve correctly, but the application produces different results at runtime. Resql has a functional integration test suite (`QueryControllerIntegrationTest`) exercising the full request path, which can serve as a validation baseline. Ruuter's test suite is currently non-functional — execution is disabled in Gradle (`exclude '**/*'`) and test methods are commented out at source level (`//TODO: tests are currently broken`). The test classes exist (`HttpHelperTest`, `ScriptingHelperIT`, `DslServiceIT`, and others) but their implementations need to be restored. Restoring coverage should begin with `ScriptingHelperIT` (core DSL engine) and `HttpHelperTest` (outbound REST calls), as these cover the highest-risk migration areas.

**Smoke Testing**

Some migration problems only surface in a fully running application — particularly those involving runtime service discovery, classloader behavior, and third-party engine initialization. Two paths are critical:

- Ruuter's GraalVM JS engine registers via JSR-223 SPI at startup. If the engine name `"graal.js"` fails to resolve on JDK 21, every DSL evaluation fails. A single request triggering script evaluation confirms or rules out this risk.
- Resql's dynamic datasource routing relies on thread-local context switching through `RoutingDataSource`. A request exercising GET, POST, and batch endpoints with datasource routing validates the full query execution path.

**End-to-End Validation in Staging**

Individual service testing does not cover inter-service communication. Both services should be deployed together in a staging environment mirroring production (same Kubernetes configuration, network policies, database connectivity). Testing the full Ruuter → Resql chain — external request through guard authentication, DSL evaluation, internal service call, query execution, and response — validates that the migration does not break cross-service behavior. This should include both cookie-based and nonce-based authentication paths, as well as error scenarios (invalid paths, missing parameters, unavailable datasources) to verify error propagation across service boundaries.

---

### 1.5 Error Handling and Reliability

#### 1.5.1 Distinguishing Error Types

Both projects handle errors, but neither does so in a standardized or fully correct way.

**Resql** has a `GlobalExceptionHandler` with a `@ControllerAdvice` that catches `ResqlRuntimeException` and a catch-all for `Exception`. However, both handlers return `400 BAD_REQUEST` regardless of the actual error type — meaning a missing query, a database connection failure, and an unexpected `NullPointerException` all produce the same HTTP status code. The response body is a custom `ErrorResponseBody` record with an exception class name and message, but no error code, no correlation ID, and no way for the caller to programmatically distinguish between error categories. A consumer receiving a 400 cannot know whether to retry (transient DB issue) or fix the request (bad parameters).

**Ruuter** has no `@RestControllerAdvice` — errors during DSL execution are caught internally and wrapped in `DSLExecutionException`, which carries a `dslName`, `stepName`, `causeCode`, and `message`. This is a structured approach, but the cause code assignment contains a bug: the `if` chain does not use `else if`, so the `causeCode` is set by each matching condition and then overwritten by the final `else` branch. In practice, a `NullPointerException` sets `E_null` but immediately falls through to `E_unknown`, making the error codes unreliable. The structured `ErrorObject` is only returned when the `x-ruuter-testing` header is present — in normal operation, errors are either swallowed by `stopInCaseOfException` or returned as generic failures.

Both projects should adopt Spring Boot 3.x's RFC 7807 `ProblemDetail` format, which provides a standardized, machine-parseable error response with proper HTTP status mapping:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail detail = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        detail.setProperty("errorCode", ex.getErrorCode());
        return detail;
    }

    @ExceptionHandler(UpstreamServiceException.class)
    public ProblemDetail handleUpstream(UpstreamServiceException ex) {
        ProblemDetail detail = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_GATEWAY, ex.getMessage());
        detail.setProperty("service", ex.getServiceName());
        return detail;
    }
}
```

For Resql, this means replacing the existing `GlobalExceptionHandler` with proper status code mapping: `InvalidQueryException` → 400, `NotFoundException` → 404, `UnknownDataSourceNameException` → 500, `ResqlRuntimeException` → 500 — instead of returning 400 for everything.

For Ruuter, this means adding a `@RestControllerAdvice` alongside the existing DSL-level error handling, fixing the `DSLExecutionException` cause code bug (`if` → `else if`), and preserving the existing error codes (`E_unknown`, `E_null`, `E_script`, `E_network`) as `errorCode` properties within the `ProblemDetail` format.

The standardized response format:

```json
{"type":"about:blank","title":"Not Found","status":404,"detail":"Query 12345 not found","errorCode":"RESOURCE_NOT_FOUND"}
```

This enables consumers — particularly Ruuter when calling Resql — to programmatically distinguish between client errors (don't retry), server errors (maybe retry), and upstream failures (retry with backoff).

#### 1.5.2 Handling Unexpected Errors

Resql's current catch-all handler logs the error with `log.error()` but returns only the exception class name and message to the client as a `400 BAD_REQUEST`. There is no correlation between what the client sees and what appears in server logs — if multiple errors occur simultaneously, there is no way to match a client's error report to the corresponding log entry.

Ruuter has no catch-all at the controller level. Unexpected exceptions during DSL execution are caught by the service layer and either swallowed (if `stopInCaseOfException` is false) or returned as a generic failure. Stack traces may appear in logs but have no link to the response the client received.

The solution is a catch-all handler in both projects that generates a unique correlation ID per error, logs the full diagnostic context server-side, and returns only the correlation ID to the client:

```java
@ExceptionHandler(Exception.class)
public ProblemDetail handleUnexpected(Exception ex, HttpServletRequest request) {
    String correlationId = UUID.randomUUID().toString();
    log.error("Unexpected error [correlationId={}, path={}, method={}]",
        correlationId, request.getRequestURI(), request.getMethod(), ex);

    ProblemDetail detail = ProblemDetail.forStatusAndDetail(
        HttpStatus.INTERNAL_SERVER_ERROR,
        "Internal error. Reference: " + correlationId);
    detail.setProperty("correlationId", correlationId);
    return detail;
}
```

For the Ruuter → Resql call chain, the `X-Correlation-Id` header should be propagated across service boundaries — when Ruuter receives an error from Resql, the correlation ID links the client-facing error in Ruuter to the server-side log entry in Resql. This integrates with Ruuter's existing OpenSearch logging, enabling cross-service error tracing with a single identifier.

#### 1.5.3 Retry Strategy

Ruuter's architecture follows a fail-fast principle — cutting execution at the first sign of error. Retry logic is a deliberate exception to this philosophy, applied only to transient network failures where the request itself is valid but the downstream service is temporarily unreachable.

Retry logic is primarily relevant for Ruuter, where `ExternalForwardingHelper` and `HttpHelper` make outbound REST calls that can fail due to transient network issues, service restarts, or temporary downstream unavailability. The `DSLExecutionException` already distinguishes `E_network` for `WebClientRequestException` — this is exactly the error type that benefits from retry.

Two approaches are suitable depending on the level of control needed:

**Spring Retry** provides annotation-driven retry with minimal configuration — suitable for straightforward cases where the retry policy is the same across all outbound calls:

```java
@Retryable(
    retryFor = {UpstreamServiceException.class, ConnectException.class},
    noRetryFor = {ValidationException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 500, multiplier = 2.0, maxDelay = 5000))
public ResponseEntity<String> callUpstream(String url, Object payload) { ... }
```

**Resilience4j** adds circuit breaker capability on top of retry — when a downstream service is consistently failing, the circuit breaker stops sending requests for a configurable period, preventing cascade failures and giving the downstream service time to recover:

```yaml
resilience4j:
  retry:
    instances:
      upstreamService:
        max-attempts: 3
        wait-duration: 500ms
  circuitbreaker:
    instances:
      upstreamService:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
```

Both approaches should integrate with Ruuter's existing `externalForwarding.proceedPredicate.httpStatusCode` configuration, which already defines which HTTP status codes are acceptable for proceeding with DSL processing. The retry policy should align: status codes outside the `proceedPredicate` allowlist that indicate transient failure (502, 503, 504) should trigger retry, while client errors (4xx) should not.

Key principles: use exponential backoff to avoid overwhelming recovering services, never retry non-idempotent operations (POST) without explicit confirmation of safety, and set a maximum retry duration to prevent requests from hanging indefinitely.

Resql's database connectivity retry is handled automatically by HikariCP, which maintains the connection pool and transparently recovers from connection loss. Additional retry logic should only be considered for transient PostgreSQL errors such as connection spikes or temporary `too many connections` scenarios.

---

## 2. Improving Ruuter Documentation

### 2.1 How to Enhance or Restructure Documentation for Newer Developers

Analysis of existing documentation: the `README.md` contains only Docker, build, and test commands with links to Guide and Configuration — no introduction, no architecture context. The `GUIDE.md` has solid DSL content (file structure, request types, responses, optional parameters, step types, scripting) but a flat structure with no progression from overview to detail. The `CONFIGURATION.md` is the most complete document, covering DSL paths, exception handling, CORS, external forwarding, internal requests, OpenSearch logging, response limits, and more. Security and input validation are scattered or absent across all three documents.

#### Solution Proposal

**README.md** — restructure as the entry point:
- Add an introduction explaining Ruuter: a service executing custom DSL files to orchestrate REST-based workflows, acting as the gateway in the Bürokratt architecture
- Add architecture overview: external traffic → Ruuter → internal services. Explain the dual deployment model — Public and Private Ruuter are separate deployments of the same codebase with different DSL directories, isolated at infrastructure level.
- Add a quickstart with a minimal DSL example including the mandatory declaration block
- Keep Docker/build/test below the introduction
- Link to GUIDE.md, CONFIGURATION.md, and new SECURITY.md

**GUIDE.md** — restructure for progressive disclosure:
- **Top**: Overview, directory structure (GET/POST method directories with guard files), declaration block as mandatory first element. Distinguish regular DSLs from template DSLs (predefined internal flows).
- **Middle**: Detailed DSL writing — step types, scripting, parameter passing, guard files as auth middleware. Include a practical multi-step workflow example (see below).
- **Bottom**: Advanced topics — reloading, default services, mock steps, batch processing, recursion overrides.
- Remove content duplicating CONFIGURATION.md; reference SECURITY.md for security topics.

**Practical multi-step workflow example** to include in GUIDE.md, demonstrating a real routing scenario:

```yaml
declaration:
  call: declare
  version: 0.1
  description: "Fetches user data and returns formatted response"
  method: post
  accepts: json
  returns: json
  namespace: backoffice
  allowlist:
    body:
      - field: userId
        type: number
        description: "User ID to look up"
    header:
      - field: cookie
        type: string
        description: "Session cookie"

get_user:
  call: http.post
  args:
    url: "[#RESQL_URL]/byk/get-user-by-id"
    body:
      userId: ${incoming.body.userId}
  result: user_response
  next: check_result

check_result:
  switch:
    - condition: ${user_response.response.body[0] == null}
      next: not_found
  next: format_response

format_response:
  assign:
    user_name: ${user_response.response.body[0].firstName}
  next: return_success

return_success:
  return: ${user_name}
  status: 200
  next: end

not_found:
  return: "User not found"
  status: 404
  next: end
```

This example demonstrates: declaration with body and header allowlist, HTTP call to an internal service (Resql), conditional branching with `switch`, variable assignment, and multiple return paths with status codes.

**DSL Limitations** — consolidate into a clear section in GUIDE.md:
- Undeclared parameters silently become `null` (declaration allowlist)
- DSL files are loaded at startup — no hot-reload unless `allowDslReloading` is enabled
- Step recursion is capped by `maxStepRecursions` (default 10)
- Only lambda syntax for anonymous functions in JavaScript (curly braces reserved for DSL parameter identifiers)
- DSL names must be unique within the same method directory, even across subdirectory levels
- Disallowed file types in the DSL directory prevent Ruuter from starting entirely
- `stopInCaseOfException` affects whether a failed step halts the entire DSL or allows subsequent steps to continue

**CONFIGURATION.md** — minor improvements:
- Add intro on where config lives and how to override per environment
- Ensure all examples include default values
- Cross-reference SECURITY.md for security properties

### 2.2 How to Document Use and Validation of Inputs

Add an input validation section to GUIDE.md.

**Declaration Block (Mandatory)**

Every DSL file must begin with a declaration block defining its contract:

```yaml
declaration:
  call: declare
  version: 0.1
  description: "Description of the DSL"
  method: post
  accepts: json
  returns: json
  namespace: backoffice
  allowlist:
    body:
      - field: userId
        type: number
        description: "User identifier"
    params:
      - field: page
        type: number
        description: "Pagination page number"
    header:
      - field: x-ruuter-nonce
        type: string
        description: "Single-use nonce for service-to-service auth"
```

The allowlist has three sections mapping to incoming request parts:

| Section | Maps to | Typical use |
|---------|---------|-------------|
| `body` | `incoming.body.*` | POST/PUT request body fields |
| `params` | `incoming.params.*` | GET query parameters |
| `header` | `incoming.headers.*` | HTTP request headers |

Critical behavior: **any incoming field not listed in the allowlist is set to `null`**, even if the caller provides it. This is silent — no error, no warning. The `namespace` field (e.g., `backoffice`, `service`, `analytics`) is a project identifier — not an access control mechanism.

**DSL Validation**: Invalid YAML syntax handling (startup vs runtime), non-existent step references, circular jumps with `maxStepRecursions`, disallowed file types (prevents startup), `LoadDslsException` behavior.

**Script Validation**: `ScriptEvaluationException` on failed JavaScript evaluation, `stopInCaseOfException` flow control, error codes `E_unknown`, `E_null`, `E_script` from `DSLExecutionException`.

**HTTP Method Validation**: Response for methods not in `allowedMethodTypes`, interaction between declaration `method` field, directory-level method, and global config.

**Testing Mode**: `x-ruuter-testing` header with `apiRequestTestingKey` enables diagnostic error responses with `dslName`, `stepName`, `causeCode`, `message`.

### 2.3 Security Documentation

Create a dedicated `SECURITY.md` linked from README.md and GUIDE.md.

**Architecture-Level Security Model**
- Ruuter is the public-facing entry point enforcing the trust boundary
- Internal services operate with `permitAll()` — reachable only within Kubernetes
- Public and Private Ruuter are **separate deployments** — boundary enforced at infrastructure level

**Authentication: Guard DSL Files**

Authentication is implemented at the DSL level via guard files at the root of each method directory. Every request passes through the guard. The guard implements:

1. **Nonce-based authentication** — `x-ruuter-nonce` header validated against Resql as a single-use token. Used by internal systems (cron, scheduled tasks) that cannot obtain browser cookies.
2. **Cookie-based authentication** — Session cookie validated via template call to `check-user-authority`. Standard path for UI/browser requests.
3. **Rejection** — Neither valid nonce nor valid cookie → 403.

Both deployments use guard files. The guard is itself a DSL — authentication logic is configurable without code changes. As a potential future improvement, mTLS or shared secrets at Spring Security level could replace nonce management, though this would move auth outside the DSL engine.

**Internal Service Access Control**: `allowedIPs` and `allowedURLs` for `internal` subdirectory DSLs, filtering behavior, unauthorized access response, allowlist maintenance.

**External Forwarding as Authentication**: `externalForwarding` as pre-processing validation hook, `proceedPredicate.httpStatusCode`, behavior when endpoint is unreachable.

**Script Evaluation Security**: GraalVM interpreter-only boundaries, JS access restrictions, script injection risk from untrusted input.

**CORS, Response Code Masking, Testing Mode**: `allowedOrigins` per environment, `finalResponse` status codes to prevent backend probing, `x-ruuter-testing` exposure risk in production.

---

## 3. Improving Resql Documentation

### 3.1 How to Improve Existing Swagger UI API Documentation

Analysis of the codebase identified three issues: the Swagger dependency is `springdoc-openapi-ui:1.8.0` (Spring Boot 2.x / `javax` variant, incompatible with Boot 3.x). The `QueryController` wildcard mapping `/{project}/**` intercepts Swagger UI paths, making documentation inaccessible. Endpoint annotations are minimal — one `@Operation(description=...)` per method. The service layer has no Javadoc.

#### Solution Proposal

##### Step 1 — Replace the Dependency

Replace `springdoc-openapi-ui:1.8.0` with the Spring Boot 3.x compatible version:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

##### Step 2 — Resolve the Path Conflict

Currently controllers intercept the Swagger page, making it inaccessible. Add an `/api` prefix to separate application routes from framework routes. This also requires adjustments from the Ruuter side:

```java
@RestController
@RequestMapping("/api")
public class QueryController { ... }
```

##### Step 3 — OpenAPI Configuration Class

Resql currently has no central API metadata definition. Adding an `OpenApiConfiguration` class provides the "title page" for the Swagger UI — giving developers and consumers immediate context about what the service does, who maintains it, and the critical note that it is internal-only. This information renders at the top of the Swagger UI page and in the generated OpenAPI spec:

```java
@Configuration
public class OpenApiConfiguration {
    @Bean
    public OpenAPI resqlOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Resql API")
                .description("Central database query microservice. Executes pre-configured "
                    + "SQL queries via REST endpoints with dynamic datasource routing. "
                    + "Intended for internal network access only.")
                .version("0.0.1-SNAPSHOT")
                .contact(new Contact().name("Bürokratt Team")))
            .servers(List.of(
                new Server().url("/").description("Current environment")));
    }
}
```

##### Step 4 — Annotate Controllers

The existing controller annotations provide only a single-line `@Operation(description=...)` with no information about parameters, possible response codes, or request/response body structure. Each endpoint should be fully annotated so that the Swagger UI becomes a self-contained API reference — a consumer should be able to understand how to call the endpoint, what to send, and what to expect back, without reading source code. This includes `@Operation` with both a summary and description, `@Parameter` annotations for path variables and query parameters, `@ApiResponse` for each possible HTTP status code, and `@Content`/`@Schema` with concrete examples for request and response bodies:

```java
@Operation(
    summary = "Execute a configured POST query",
    description = "Executes a pre-configured SQL query identified by the URL path. "
        + "The {project} segment determines the datasource, the remaining path "
        + "identifies the query file.")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Query executed successfully",
        content = @Content(mediaType = "application/json",
            schema = @Schema(type = "array",
                example = "[{\"id\": 1, \"name\": \"example\"}]"))),
    @ApiResponse(responseCode = "404", description = "Query not found"),
    @ApiResponse(responseCode = "500", description = "Query execution failed")
})
@PostMapping(value = "/{project}/**")
public List<Map<String, Object>> execute(
    @Parameter(description = "Project/datasource identifier", example = "byk", required = true)
    @PathVariable String project, ...) { ... }
```

The same level of annotation should be applied to the batch endpoint, `DataSourceController`, and `HeartBeatController` — every public endpoint should be fully documented in the Swagger UI.

##### Step 5 — Annotate DTOs and Models

Controller annotations alone are not enough — the "Schemas" section at the bottom of Swagger UI is only useful if the DTO and model classes carry `@Schema` annotations with field-level descriptions, types, and examples. Without these, the schema section shows raw field names with no context. Every object that appears in an API request or response should be annotated so that a developer can understand the data contract without reading Java source:

```java
@Data @Builder
@Schema(description = "Application health and version information")
public class HeartBeatInfo {
    @Schema(description = "Application name", example = "sql-ms")
    private String appName;
    @Schema(description = "Application version", example = "0.0.1-SNAPSHOT")
    private String version;
    @Schema(description = "Packaging timestamp (epoch millis)", example = "1700000000000")
    private long packagingTime;
    @Schema(description = "Startup timestamp (epoch millis)", example = "1700001000000")
    private long appStartTime;
    @Schema(description = "Current server timestamp (epoch millis)", example = "1700002000000")
    private long serverTime;
}
```

For `DataSourceConfigProperties`, any sensitive fields (passwords, connection strings containing credentials) must be explicitly marked with `@Schema(hidden = true)` or `@JsonIgnore` to guarantee they never appear in the Swagger UI or API responses — this should be enforced at the schema level, not relied upon by convention. The `BatchRequest` record is currently defined inline inside `QueryController` — it should be extracted to its own file for better visibility and annotated with `@Schema` to document the expected batch input format.

##### Step 6 — Javadoc on Service Layer

While Javadoc does not render in Swagger UI, it is essential for developers working in the codebase. The service layer is where query resolution, datasource routing, and execution logic live — currently with no documentation. Each service class and its public methods should have Javadoc explaining what the method does, how it resolves its inputs (e.g., how a project name and path map to a SQL file on disk), what exceptions it throws and under what conditions, and any side effects such as caching:

```java
/**
 * Core query execution service. Resolves REST paths to pre-configured
 * SQL files and executes them against the appropriate datasource.
 *
 * @param project    datasource identifier for routing
 * @param httpMethod determines query directory
 * @param name       query path relative to method directory
 * @param parameters named parameters to bind (nullable for GET)
 * @return result rows as column-name-to-value maps
 * @throws NotFoundException if query file not found
 * @throws ResqlRuntimeException if execution fails
 */
```

The same treatment should be applied to `SavedQueryService` (how queries are loaded from disk, whether results are cached, how filesystem paths are resolved), `ServerInfoService` (what server data it provides and where it sources it), and `HeartBeatService` (how `packagingTime` is determined — sourced from `heartbeat.properties` via `PackageInfoConfiguration` at build time).

---

### 3.2 How to Better Document Configuration Parameters

Resql has no configuration documentation. Create a `CONFIGURATION.md` covering:

**Datasource Configuration**
- Connections configured via `sqlms.datasources` prefix in `application.yml`, bound to `List<DataSourceConfigProperties>`
- Dynamic routing: `DataSourceConfiguration` builds `RoutingDataSource` → `DataSourceContextHolder` (thread-local) sets active datasource per request → `RoutingDataSource.determineCurrentLookupKey()` reads it
- Current limitation: `SavedQuery.getDatabaseName()` returns hardcoded `"byk"` — multi-datasource routing at query level not yet functional
- Timezone: `DataSourceConfiguration` sets `SET TIME ZONE` via HikariCP `connectionInitSql`, defaults to `Europe/Tallinn`

**Connection Pool (HikariCP) — Environment-Specific Configuration**

The connection pool directly impacts reliability and performance. Configuration should differ by environment:

| Property | Development | Staging | Production |
|----------|-------------|---------|------------|
| `maximum-pool-size` | 2–5 | 5–10 | 10–20 (tuned to load) |
| `connection-timeout` | 30000ms | 20000ms | 10000ms |
| `idle-timeout` | 600000ms | 300000ms | 180000ms |
| `max-lifetime` | 1800000ms | 1200000ms | 900000ms |
| `datasource.url` | H2 in-memory | PostgreSQL (staging DB) | PostgreSQL (production DB) |

Example environment-specific configuration:

```yaml
# application-dev.yml
sqlms:
  datasources:
    - name: byk
      driverClassName: org.h2.Driver
      jdbcUrl: jdbc:h2:mem:testdb
      username: sa
      password:
      timeZone: Europe/Tallinn

# application-prod.yml
sqlms:
  datasources:
    - name: byk
      driverClassName: org.postgresql.Driver
      jdbcUrl: jdbc:postgresql://db-host:5432/buerokratt
      username: ${DB_USERNAME}
      password: ${DB_PASSWORD}
      timeZone: Europe/Tallinn
```

**Reliability impact**: An undersized pool causes request queuing — every REST call holds a connection for the full duration of query execution. An oversized pool wastes database connections and can hit PostgreSQL's `max_connections` limit. `connection-timeout` determines how long a request waits for a free connection before failing — too high and users experience long hangs, too low and transient spikes cause unnecessary failures.

**Performance impact**: `idle-timeout` and `max-lifetime` control connection recycling. Stale connections can cause intermittent failures. `max-lifetime` should be set lower than the database server's connection timeout to prevent the pool from holding dead connections.

**Query Configuration**
- SQL files under configurable path, organized by project and HTTP method
- Filesystem path directly maps to REST endpoint (path-based routing)
- Named parameter syntax (`:paramName`) for prepared statements
- GET: query parameters; POST: request body

**Security Headers, Heartbeat, H2, Logging**
- `headers.content-security-policy` and per-environment values
- `heartbeat.properties`: `app.name`, `app.version`, `app.packaging.time`
- H2 for dev/test, PostgreSQL for staging/production via Spring profiles
- `logback-spring.xml` levels and format

---

### 3.3 How to Better Document Use Cases and Security Risks

Resql's `SecurityConfiguration` uses `permitAll()` with CSRF disabled and no documentation. Create three documents and restructure `README.md` to link to all.

#### SECURITY.md

**Deployment Security Model**

Resql runs exclusively within the internal Kubernetes network with no external ingress. `permitAll()` is a deliberate architectural decision — authentication is delegated to Ruuter. Network policies enforce the boundary.

**The Ruuter → Resql Trust Boundary**

Call chain: clients → Ruuter (guard DSL auth, routing, IP filtering) → Resql (trusts all). Implications: misconfigured network policies expose raw SQL execution; compromised internal services get full query access. Mitigations: network policy validation, monitoring, potential future mTLS.

**SQL Injection Prevention**

`ResqlJdbcTemplate` extends `NamedParameterJdbcTemplate` with `MapSqlParameterSource` — prepared statements, never concatenation. SQL files are pre-configured on disk — users cannot submit arbitrary SQL.

**Allowed and Forbidden SQL Operations**

No application-level restriction on operation types exists. Any valid SQL placed in a file will execute via REST.

*Allowed (intended):*
- Parameterized SELECT, INSERT, UPDATE, DELETE — `queryOrExecute()` auto-detects via `ResultSetMetaData`

*Forbidden (enforced by convention and code review):*
- DDL (CREATE, DROP, ALTER, TRUNCATE) — no runtime guard
- Unbounded SELECTs without LIMIT — full result loaded into memory

**Known Architectural Risks and Reliability Impact**

- **No query timeout** — `queryOrExecute()` sets no statement timeout. A runaway query holds a HikariCP connection indefinitely, eventually exhausting the pool and making the service unresponsive to all requests. Mitigation: configure `statement_timeout` at PostgreSQL level or via `connectionInitSql`.
- **No result set size limit** — results map directly into `List<Map>` with no cap. A 10-million-row SELECT attempts to load everything into JVM heap → `OutOfMemoryError` → service crash. Mitigation: enforce `LIMIT` in all queries or implement result truncation.
- **Batch without transactions** — `executeBatch` calls `executePost` in a stream with no `@Transactional`. If query 3 of 5 fails, queries 1–2 are committed, 4–5 never execute. Partial writes, inconsistent data. Mitigation: wrap in `@Transactional`.
- **ThreadLocal datasource leak** — `DataSourceContextHolder.setDataSourceName()` is called but `clearDataSourceName()` is never called in the request flow. In a thread pool, the previous request's datasource context leaks to the next request on the same thread. Mitigation: clear in a servlet filter or interceptor.
- **CORS wildcard default** — `RestConfiguration` defaults `cors.allowedOrigins` to `*`. Acceptable for internal-only but must be documented as intentional.

**CSRF Disabled** — Stateless API, no cookies/sessions, machine-to-machine only. Correct decision for this architecture.

#### GUIDE.md

**What Resql Is**: Centralized query microservice exposing pre-configured SQL as REST endpoints with dynamic datasource routing.

**SQL File Lifecycle**:

1. Developer writes `.sql` file with `:paramName` parameters
2. Placed in `{config-path}/{project}/{METHOD}/query-name.sql`
3. `SavedQueryService` loads files at startup into `SavedQuery` records
4. Client sends `POST /byk/get-user` with `{"userId": 123}`
5. `QueryController` parses: project=`byk`, method=`POST`, path=`/get-user`
6. `QueryService` retrieves `SavedQuery`, sets datasource via `DataSourceContextHolder`
7. `RoutingDataSource` routes to correct database
8. `ResqlJdbcTemplate.queryOrExecute()` executes with bound parameters
9. Results returned as JSON with `snake_case` → `camelCase` conversion

Key implications: no hot-reload, no startup syntax validation, path = API contract, auto-detection of SELECT vs write.

**Practical examples**: GET query, POST query, batch, adding a new query file, adding a new datasource.

**Error mapping**: `NotFoundException` → 404, `InvalidQueryException` → 400, `InvalidDirectoryException` → 500, `UnknownDataSourceNameException` → 500, `ResqlRuntimeException` → 500.

#### Updated README.md

- Brief introduction (2–3 sentences)
- Architecture note: internal-only, called by Ruuter
- Links to GUIDE.md, CONFIGURATION.md, SECURITY.md
- Docker/build/test commands
- License
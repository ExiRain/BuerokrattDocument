# Bürokrati proovitöö

This document outlines the solution for migrating Ruuter and Resql to Java 21, addressing error handling and reliability concerns, and updating documentation for ease of onboarding and general knowledge.

Both applications are event-driven REST microservices running on Spring Boot 3.2.5:

- **Resql** — Maven-based, REST → DB query service with dynamic datasource routing. Deployed as an internal-only Kubernetes pod with no external network access. Executes pre-configured SQL queries exposed via REST endpoints.
- **Ruuter** — Gradle-based, REST → REST routing service with a GraalVM JS-based DSL engine. Acts as the public-facing gateway that orchestrates calls to internal services including Resql.

Both were originally built on Java 11, then upgraded to Java 17 without thorough migration review. This document covers the next step to Java 21 alongside architectural improvements to error handling, reliability, and documentation.

The intended deployment architecture follows a trust boundary model: external traffic reaches Ruuter, which validates requests, evaluates DSL routing rules, and forwards calls to internal services. Resql is only reachable from within the internal network (Kubernetes cluster), which is why it operates with open security (`permitAll()`) — it delegates authentication and authorization responsibility to the gateway layer (Ruuter).

---

## 1. Migration from Java 17 to Java 21

### 1.1 Risks of Java 21 Adoption

#### Shared Risks

- **Stronger encapsulation** — Java 21 further tightens access to internal JDK APIs. Any `--add-opens` or `--add-exports` JVM flags that were introduced during the Java 11→17 upgrade need to be audited. These flags may have been added to Dockerfiles, startup scripts, or CI pipelines to suppress `InaccessibleObjectException` errors at the time. They still function on Java 21 but mask underlying issues that should be resolved properly rather than worked around indefinitely.
- **Behavioral changes** — There are minor `HashMap`/`HashSet` iteration order differences across JDK builds. If any code or tests depend on iteration order (including JSON serialization of unordered maps), they may produce inconsistent results. This is especially relevant for Ruuter's DSL evaluation where `LinkedHashMap` is used in some code paths but standard `HashMap` in others within `ScriptingHelper`.
- **Virtual threads caveat** — Java 21 introduces virtual threads as a production-ready feature. Both projects use thread-local patterns (`DataSourceContextHolder` in Resql, request context passing in Ruuter) that would behave unexpectedly with virtual threads, since virtual threads can share carrier threads. Adopting virtual threads is entirely optional and not part of this migration, but it should be noted as a consideration for future architecture decisions. Spring Boot 3.2.5 does not enable virtual threads by default.
- **Framework-level risk is minimal** — Spring Boot 3.2.5 officially supports Java 21. The Spring team tests against JDK 21 as part of their CI matrix. No Spring-specific breaking changes are expected from the JDK version bump alone.

#### Resql-Specific Risks

| Risk | Severity | Detail |
|------|----------|--------|
| `id-log` library (system-scoped) | High | This is the single highest risk item in Resql. The library is compiled with **Java 11 bytecode** while the application runs on Java 17, creating a bytecode version mismatch that has silently persisted since the last upgrade. The library contains `javax.servlet.http.HttpServletRequest` and `javax.servlet.http.HttpServletResponse` imports — these are **Java EE** namespace classes that do not exist in Spring Boot 3.x's Jakarta EE 9+ runtime. The library currently "works" because the affected interceptor classes (`GenericHeaderLogHandler`, `LogHandler`) are likely never fully invoked at runtime — only their utility methods are called. This is a latent defect regardless of Java version. Additionally, `id-log` is declared with Maven `system` scope and a hardcoded `systemPath`, which is inherently fragile: Maven does not include system-scoped JARs in WAR packaging by default, there is no version resolution or conflict detection, and the JAR is referenced by absolute filesystem path, tying the build to a specific project structure. |
| Spring Cloud BOM (2021.0.2) | Low | Declared in `dependencyManagement` but targets Spring Boot 2.6.x/2.7.x — completely wrong for Boot 3.2.5. A full codebase search confirmed zero Spring Cloud usage: no `org.springframework.cloud` imports in Java files, no `spring.cloud.*` configuration properties in application.yml, no Spring Cloud annotations (`@EnableDiscoveryClient`, `@FeignClient`, `@LoadBalanced`, `@EnableConfigServer`, `@CircuitBreaker`). The only "cloud" keyword matches were SonarCloud CI configuration (`sonarcloud.yml`) and already-commented-out POM entries. This is dead weight from a previous project template — safe to remove entirely. |
| PostgreSQL driver (42.3.9) | Medium | Functions correctly on Java 21 but contains known CVEs. The current version is significantly behind the latest stable line (42.7.x). As the sole database driver in a DB-focused microservice, this should be upgraded as part of the migration for both compatibility and security reasons. |
| OpenTelemetry BOM scope | Low | The OpenTelemetry BOM is declared with `<scope>runtime</scope>`, which prevents it from functioning as a BOM for version management of child dependencies. The individual OpenTelemetry dependencies (`opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-context`, `opentelemetry-exporter-logging`) resolve their versions transitively rather than through the BOM. Should be moved into `<dependencyManagement>` without a scope qualifier to work as intended. |
| Dynamic datasource routing | None | `RoutingDataSource` extends Spring's `AbstractRoutingDataSource` with a standard thread-local based lookup via `DataSourceContextHolder`. Code review confirmed this is a straightforward, well-established pattern with no behavioral changes between Java 17 and 21. No action needed. |

#### Ruuter-Specific Risks

| Risk | Severity | Detail |
|------|----------|--------|
| GraalVM JS Engine (23.0.1) | Medium | `ScriptEngineConfiguration` initializes the engine via `new ScriptEngineManager().getEngineByName("graal.js")` using the JSR-223 SPI bridge. The `javax.script.*` imports (`ScriptEngine`, `Bindings`, `ScriptEngineManager`) are **JDK-native** — they belong to the `java.scripting` module and are not subject to the Jakarta rename. The `polyglot.engine.WarnInterpreterOnly=false` system property confirms the application runs in interpreter-only mode without GraalVM JIT compilation. The risk is version targeting: GraalVM `23.0.x` targets JDK 20, while proper JDK 21 alignment requires `23.1.x` or `24.x`. This matters because `ScriptingHelper` is the **core of Ruuter's entire routing DSL** — every incoming request flows through `evaluateScripts()` → `engine.eval()`. If GraalVM JS engine resolution fails on JDK 21, the entire service is non-functional. However, the fix is straightforward: if the smoke test fails, bump the GraalVM dependency version. In practice, interpreter-only mode is the most portable configuration and will likely work without changes. |
| Apache HttpClient 4.5.13 | Medium | The old `org.apache.httpcomponents:httpclient` (HttpClient 4.x) is end-of-life. It functions on Java 21 but receives no further security patches or bug fixes. The replacement is `org.apache.httpcomponents.client5:httpclient5`. Since Ruuter's core purpose is REST→REST forwarding (implemented in `ExternalForwardingHelper` and `HttpHelper`), this is a central dependency worth upgrading — not just for Java 21 compatibility, but for long-term maintainability and security. |
| WireMock `wiremock-jre8` (2.35.1) | Medium | The `-jre8` artifact variant is deprecated by the WireMock project. For Java 21 compatibility, the migration path is to `org.wiremock:wiremock:3.x` (note: this is a new Maven group ID, not just a version bump). This affects test infrastructure only, not production code. |
| Mockito version conflict | Low | `mockito-inline:4.9.0` is declared alongside `mockito-core:5.6.0`. This is a version mismatch — Mockito 5.x includes inline mocking capability by default, which was the primary motivation for the 5.x major version bump. The separate `mockito-inline` artifact is redundant at best and can cause unpredictable test behavior due to the 4.x/5.x version conflict. Remove `mockito-inline` entirely. |
| All tests disabled | High | The Gradle build configuration contains `exclude '**/*'` in the test block, which means **zero tests execute during builds**. This eliminates any automated safety net for the migration. Before any Java 21 work begins, this must be addressed — either by re-enabling and fixing existing tests, or by formally acknowledging that the migration proceeds without automated test coverage and relies entirely on manual smoke testing. This is a risk multiplier for every other item in this table. |
| Missing Java version target | Medium | The `build.gradle` file does not declare `sourceCompatibility` or `targetCompatibility`. The compiled bytecode version is determined by whichever JDK happens to run the build — this is implicit, fragile, and means builds are not reproducible across different developer machines or CI agents. Must be set explicitly for Java 21. |

---

### 1.2 Impact on Spring/Jakarta Stack

#### Jakarta EE Namespace Migration Status

The `javax` → `jakarta` namespace migration (required by Spring Boot 3.x / Jakarta EE 9+) has **already been completed** in both projects. This was part of the Spring Boot 2.x → 3.x upgrade.

**Resql**: All servlet imports use `jakarta.servlet`. A full codebase search for `import javax.` found only `javax.sql.DataSource` in `ResqlJdbcTemplate`. This class is part of the JDK's `java.sql` module — it was never part of Java EE and is not subject to the Jakarta rename. No action needed.

**Ruuter**: Spring Boot 3.2.5 starter dependencies pull in Jakarta namespace automatically. A search for `javax.` found `javax.script.ScriptEngine`, `javax.script.Bindings`, and `javax.script.ScriptEngineManager` in the scripting layer (`ScriptEngineConfiguration`, `ScriptingHelper`). These belong to the JDK's `java.scripting` module — same situation as `javax.sql` in Resql. They are JDK-native and will never be renamed. No action needed.

#### Java 21 Impact on the Stack

Java 21 introduces no new Jakarta EE version requirements. The existing Spring Boot 3.2.5 + Jakarta EE 9+ combination is fully compatible with Java 21 without modification. Spring Boot 3.4.x (latest stable at time of writing) offers improved Java 21 support including virtual thread auto-configuration via `spring.threads.virtual.enabled=true`, but upgrading Spring Boot version is optional and not a prerequisite for the Java 21 migration.

#### Exception: `id-log` Library in Resql

The `id-log-1.0.0-SNAPSHOT.jar` is the only component across both projects that still contains `javax.servlet` (Java EE) namespace classes. The library's interceptor classes import `javax.servlet.http.HttpServletRequest` and `javax.servlet.http.HttpServletResponse` — these are incompatible with Spring Boot 3.x's Jakarta EE 9+ runtime.

The project uses only two classes from this library: `GenericHeaderLogHandler` and `LogHandler`. Both are `HandlerInterceptor` implementations that perform request/response logging — straightforward functionality that does not justify carrying an incompatible, unmaintained, system-scoped external dependency.

**Recommendation**: Write replacement classes directly in the Resql project using `jakarta.servlet` imports. This is approximately 50–100 lines of code. Then remove the `id-log` system-scoped dependency from the POM entirely. This eliminates three problems at once: the `javax.servlet` namespace incompatibility, the Java 11 bytecode version mismatch running on a Java 17/21 runtime, and the fragile system-scoped Maven dependency with a hardcoded filesystem path.

---

### 1.3 Backward Compatibility

#### API Contract Level

REST API contracts are unaffected by the JDK version change — external consumers will not notice any difference. Request/response formats, endpoint paths, HTTP status codes, and behavioral logic remain identical. The JDK upgrade is a runtime-level change that does not alter the application's public interface.

#### Runtime Environment Alignment

Build-time and runtime JDK must both be Java 21. This alignment must be verified across all environments:

**Resql** (WAR packaging) — the servlet container must support Java 21. Spring Boot 3.2.5's embedded Tomcat does. If Resql is deployed to an external Tomcat instance in any environment, that instance must also be updated to a Java 21-compatible version.

**Ruuter** (JAR packaging) — self-contained executable JAR with an embedded server. Only the JDK installation on the target host/container needs to be Java 21.

#### Build Configuration Changes

**Resql** — update `pom.xml` properties:
```xml
<java.version>21</java.version>
<maven.compiler.source>21</maven.compiler.source>
<maven.compiler.target>21</maven.compiler.target>
```

**Ruuter** — add explicit Java target to `build.gradle`:
```groovy
java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}
```

#### Docker and CI Pipeline Changes

Both projects use Docker for deployment and have CI pipelines. These require updates:

- **Docker base images** must be updated to JDK 21. For example, `eclipse-temurin:17-jre` becomes `eclipse-temurin:21-jre` (or `21-jdk` for build stages). Both `Dockerfile` and `docker-compose.yml` should be reviewed for hardcoded JDK version references.
- **CI pipeline JDK version** must match. If the pipeline uses a JDK matrix or a specific JDK image for building, it must be updated to 21.
- **Ruuter's Node.js tooling** (`package.json`, `bump-version.sh`, `generate-changelog.sh`, Husky git hooks) is unaffected by the Java version change — these are JavaScript/shell tools that run independently of the JVM.
- **Gradle and Maven wrapper scripts** (`gradlew`, `mvnw`) do not need changes — they are JDK-version-agnostic and will use whatever JDK is on the system `PATH`.

#### Dependency Compatibility

All third-party dependencies must be verified for Java 21 support. The following matrix summarizes the current state:

| Dependency | Project | Current | Java 21 Status | Action |
|-----------|---------|---------|----------------|--------|
| Spring Boot | Both | 3.2.5 | Officially supported | None required |
| PostgreSQL Driver | Resql | 42.3.9 | Works, but has CVEs | Upgrade to 42.7.x |
| GraalVM JS | Ruuter | 23.0.1 | Likely works (interpreter mode), targets JDK 20 | Smoke test; bump to 23.1.x/24.x if needed |
| Apache HttpClient | Ruuter | 4.5.13 | Works, but EOL | Migrate to httpclient5 |
| WireMock | Ruuter | 2.35.1 (jre8) | Deprecated variant | Migrate to org.wiremock:wiremock:3.x |
| Mockito | Ruuter | 5.6.0 + inline 4.9.0 | Version conflict | Remove mockito-inline |
| Lombok | Both | Managed | Supported | None |
| H2 | Resql | 2.2.224 | Supported | None |
| OpenTelemetry | Both | 1.37.0 / 1.15.0 | Supported | Align versions across projects |
| id-log | Resql | 1.0.0-SNAPSHOT (J11) | Incompatible (javax.servlet) | Rewrite and remove |

---

### 1.4 Testing the Migration

#### Step 1 — Static Analysis (before any code changes)

Run `jdeps --jdk-internals` on all JAR/WAR artifacts to detect usage of internal JDK APIs that may have been removed or further encapsulated in Java 21:

```bash
# Resql — especially important for the id-log library
jdeps --jdk-internals libs/id-log-1.0.0-SNAPSHOT.jar
jdeps --jdk-internals target/sql-ms.war

# Ruuter
jdeps --jdk-internals build/libs/*.jar
```

This analysis is non-destructive and can be performed before making any changes to the codebase. It will flag any dependencies on `sun.misc.*`, `com.sun.*`, or other internal APIs that are scheduled for removal.

#### Step 2 — Compile on JDK 21 Without Code Changes

Switch only the JDK used for compilation — do not modify source code, dependency versions, or build configuration yet. This isolates compilation-level incompatibilities from intentional changes. The majority of issues will surface at this stage as compile errors or deprecation warnings.

#### Step 3 — Dependency Conflict Analysis

Analyze the full dependency tree for version conflicts, duplicate classes, and unresolved version management:

```bash
# Resql
mvn dependency:tree -Dverbose

# Ruuter
gradle dependencies --configuration runtimeClasspath
```

Pay particular attention to transitive dependencies that may pull in older versions of libraries already declared directly.

#### Step 4 — Automated Test Execution

- **Resql**: Run the existing integration test suite (`QueryControllerIntegrationTest` and others) on JDK 21. These tests use H2 in-memory database and cover the core query execution paths.
- **Ruuter**: The test configuration currently excludes all tests (`exclude '**/*'`). This **must be resolved first** — either re-enable existing tests and fix any that fail, or formally acknowledge the absence of automated test coverage. Running the migration without any test suite means all validation falls to manual testing, significantly increasing the risk of undetected regressions.

#### Step 5 — Targeted Smoke Testing

Two critical paths require manual verification regardless of test suite coverage:

- **Ruuter**: Boot the application on JDK 21 and send a request that triggers DSL script evaluation (any route that passes through `ScriptingHelper.evaluateScripts()` → `engine.eval()`). This verifies GraalVM JS engine resolution on JDK 21. If this single test passes, the highest-risk item in Ruuter is cleared. If it fails, bump GraalVM to `23.1.x` or `24.x` and retest.
- **Resql**: Execute database queries through the REST API, including scenarios that exercise dynamic datasource routing (`RoutingDataSource` → `DataSourceContextHolder`), to verify the full request path from controller to database and back.

#### Step 6 — Staging Environment Deployment

Deploy both services to a staging environment that mirrors production (same Kubernetes configuration, same network policies, same database connectivity). Execute end-to-end scenarios that exercise the Ruuter → Resql call chain to verify inter-service communication under Java 21.

---

### 1.5 Error Handling and Reliability

#### 1.5.1 Distinguishing Error Types

Resql already has a `GlobalExceptionHandler` and a typed exception hierarchy (`InvalidDirectoryException`, `InvalidQueryException`, `ResqlRuntimeException`, `UnknownDataSourceNameException`). Ruuter has its own exception set (`InvalidDslException`, `InvalidDslStepException`, `InvalidHttpMethodTypeException`, `LoadDslsException`, `ScriptEvaluationException`). The foundation exists in both projects — the task is not building from scratch but **standardizing** both to use a consistent error response format.

Spring Boot 3.x includes built-in support for RFC 7807 `ProblemDetail`, which provides a standardized JSON error response format. Both projects should adopt this to ensure consumers (including Ruuter when calling Resql) receive uniform, machine-parseable error responses regardless of which service produced the error.

Implementation approach using `@RestControllerAdvice` with `ProblemDetail`:

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

This produces standardized JSON responses that follow RFC 7807:
```json
{
    "type": "about:blank",
    "title": "Not Found",
    "status": 404,
    "detail": "Query 12345 not found",
    "errorCode": "RESOURCE_NOT_FOUND"
}
```

Both projects should map their existing exception types to appropriate HTTP status codes and error codes using this pattern. Ruuter already has a partial error code system (`E_unknown`, `E_null`, `E_script` in `DSLExecutionException`) — these should be preserved and integrated into the `ProblemDetail` response format.

#### 1.5.2 Handling Unexpected Errors

Unexpected errors (unhandled `NullPointerException`, `IllegalStateException`, etc.) currently reach clients as generic 500 responses without useful context for debugging, or worse, with leaked stack traces that expose internal implementation details.

The solution is a catch-all handler that logs full diagnostic context server-side while returning only a correlation ID to the client:

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

For the Ruuter → Resql call chain, the `X-Correlation-Id` header should be propagated across service boundaries. When Ruuter makes a request to Resql, it should attach the correlation ID as a header. When Resql logs an error, it should include the incoming correlation ID. This enables tracing a single user request across both services using one identifier.

This is especially important given that Ruuter already has OpenSearch-based external logging configured — correlation IDs would make those logs significantly more useful for debugging cross-service failures.

#### 1.5.3 Retry Strategy

Retry logic is primarily relevant for **Ruuter**, where `ExternalForwardingHelper` and `HttpHelper` make outbound REST calls that can fail due to transient network issues, service restarts, or temporary unavailability of downstream services (including Resql).

**Option A — Spring Retry** (simpler, annotation-driven):

```java
@Retryable(
    retryFor = {UpstreamServiceException.class, ConnectException.class},
    noRetryFor = {ValidationException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 500, multiplier = 2.0, maxDelay = 5000))
public ResponseEntity<String> callUpstream(String url, Object payload) { ... }

@Recover
public ResponseEntity<String> fallback(UpstreamServiceException ex, String url, Object payload) {
    log.error("All retry attempts failed: url={}", url, ex);
    throw new UpstreamServiceException("Service unavailable after retries", url);
}
```

**Option B — Resilience4j** (more control, adds circuit breaker):

```yaml
resilience4j:
  retry:
    instances:
      upstreamService:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - ee.buerokratt.ruuter.helper.exception.UpstreamServiceException
          - java.net.ConnectException
  circuitbreaker:
    instances:
      upstreamService:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
```

The circuit breaker pattern is particularly useful for Ruuter's architecture: if Resql (or another downstream service) becomes unavailable, the circuit breaker stops sending requests after a failure threshold is reached, allowing the downstream service time to recover instead of overwhelming it with retry attempts.

Key principles for retry implementation:
- **Retry only recoverable errors** — connection timeouts, 502/503/504 responses. Never retry 4xx client errors (400, 401, 403, 404) as these indicate problems with the request itself, not transient failures.
- **Exponential backoff** — each subsequent retry waits longer (e.g., 500ms → 1s → 2s) to avoid thundering herd problems.
- **Idempotency awareness** — retry is safe for GET and PUT (idempotent operations). For POST requests, retry must be applied carefully as it can cause duplicate side effects.
- **Ruuter already has `externalForwarding.proceedPredicate.httpStatusCode`** configuration — retry logic should integrate with this existing mechanism rather than replacing it.

**Resql** — database connection retry is largely handled automatically by HikariCP connection pool, which is included with Spring Boot's JDBC starter. HikariCP maintains a pool of connections, validates them before use, and transparently recovers from connection loss. Additional retry logic should only be considered for transient database errors such as temporary connection spikes or PostgreSQL "too many connections" scenarios.

---

## 2. Improving Ruuter Documentation

### 2.1 How to Enhance or Restructure Documentation for Newer Developers

#### Current State

The main `README.md` is a bare skeleton containing only technical commands (Docker, testing, building) and links to the Guide and Configuration documents. It does not explain what Ruuter is, what problem it solves, or why it exists. For developers already working with the project, the README provides minimal value. For new developers joining the team, it offers no entry point to understanding the system's purpose or architecture.

The `GUIDE.md` file contains solid technical content covering DSL file structure, request types, query responses, optional parameters, internal services, step types, and JavaScript usage. However, the structure is flat — general concepts, DSL writing details, step type references, and scripting instructions are all at the same level without a clear progression from overview to detail.

The `CONFIGURATION.md` is the most complete documentation file, covering DSL folder location, exception handling, incoming request configuration, external forwarding, CORS, allowed filetypes, default services, internal request IP filtering, OpenSearch logging, HTTP response limits, testing flow, and more.

#### Recommended Changes

**README.md** should be restructured to serve as the entry point for new developers:
- Add an introduction section explaining what Ruuter is: a service that executes custom DSL (Domain-Specific Language) files to orchestrate REST-based workflows. It receives HTTP requests, matches them to YAML-defined DSL configurations, executes the defined steps (which can include calling external services, evaluating JavaScript expressions, conditional logic, and data transformation), and returns structured responses.
- Add a high-level architecture overview showing how Ruuter fits into the broader system: external traffic → Ruuter (DSL evaluation, routing, forwarding) → internal services (Resql, etc.).
- Add a quickstart section with a minimal but complete DSL example that shows a working request → DSL → response flow, so a new developer can understand the core concept in under five minutes.
- Keep the existing Docker/build/test commands, but move them below the introduction and overview sections.
- Link to GUIDE.md for detailed DSL writing instructions and CONFIGURATION.md for configuration parameters.

**GUIDE.md** should be restructured to follow a progressive disclosure pattern:
- **Top section**: General overview of how DSL files work (already exists, but should be the clear starting point).
- **Middle section**: Detailed DSL writing instructions — step types, scripting, parameters, jumps, conditionals (already exists, needs better organization).
- **Bottom section**: Advanced topics — reloading DSLs, default services, mock steps, template steps.
- Move all configuration-related content to CONFIGURATION.md (some duplication currently exists between the two files).
- Move all security-related content (internal services, IP filtering) to a dedicated SECURITY.md and link from the Guide.

**CONFIGURATION.md** is already thorough and well-structured. Minor improvements:
- Add introductory text explaining where configuration lives (`application.yml`) and how to override properties per environment.
- Group related properties with clear section headers (some of this already exists).
- Ensure all configuration examples are consistent in format and include default values.

### 2.2 How to Document Use and Validation of Inputs

#### Current State

Input validation behavior is not documented. The GUIDE.md mentions optional parameters (prefix `optional_`) and request body formats (JSON, formdata, multipart), but does not describe what happens when inputs are malformed, missing, or invalid. The CONFIGURATION.md documents `allowedMethodTypes` and `allowedFiletypes`, which are forms of input validation at the system level, but there is no documentation covering request-level validation.

#### Recommended Changes

Document input validation behavior either as a section in the existing GUIDE.md (for consistency, since it already contains usage-related documentation) or as a separate `INPUT_VALIDATION.md` linked from the Guide. Content should cover:

- **Request validation**: What happens when a required parameter is missing from a request body? What HTTP status code is returned? What does the error response look like?
- **DSL validation**: What happens when a DSL file contains invalid YAML syntax, references a non-existent step, or creates a circular jump? The existing `InvalidDslException`, `InvalidDslStepException`, and `LoadDslsException` suggest these scenarios are handled — but the behavior is not documented.
- **Script validation**: What happens when a JavaScript expression in a DSL fails to evaluate? The `ScriptEvaluationException` handles this at the code level, and the CONFIGURATION.md mentions `stopInCaseOfException` behavior, but the complete error flow is not described.
- **HTTP method validation**: The `allowedMethodTypes` configuration is documented, but the resulting error response when an unsupported method is used is not.
- **File type validation**: The `allowedFiletypes` configuration states that Ruuter will not start if invalid file types are present, but this behavior should be documented more prominently as it can cause confusing deployment failures.

As a **secondary option**, a separate document covering the security aspects of input handling can be created and linked from the Guide. This would cover topics such as: JavaScript injection risks through the DSL scripting engine, oversized request payloads and the existing `httpResponseSizeLimit` configuration, and request key duplication handling (`allowDuplicateRequestKeys`).

### 2.3 Security Documentation

#### Current State

Security-related configuration is scattered across CONFIGURATION.md (internal request IP filtering, allowed IPs, allowed referral URLs) and briefly mentioned in GUIDE.md (internal services section). There is no dedicated security documentation that provides a holistic view of Ruuter's security model.

#### Recommended Changes

Create a dedicated `SECURITY.md` document and link to it from both README.md and GUIDE.md. This document should cover:

- **Architecture-level security model**: Ruuter is the public-facing gateway in the deployment architecture. It is responsible for validating, filtering, and routing all incoming requests before they reach internal services (such as Resql). Internal services operate with open security policies (`permitAll()`) because they are only reachable from within the Kubernetes cluster — this is a deliberate design decision, not an oversight.
- **Internal service access control**: The `allowedIPs` and `allowedURLs` configuration for internal request endpoints. How to configure, what happens when an unauthorized IP attempts access, and best practices for maintaining the allowlist.
- **External forwarding security**: The `externalForwarding` configuration forwards every incoming request to an external validation endpoint (e.g., `https://turvis/ruuter-incoming`) before processing. The `proceedPredicate.httpStatusCode` determines whether to proceed. This is effectively an external authentication/authorization hook — its security implications should be clearly documented.
- **Script evaluation security**: Ruuter evaluates JavaScript expressions within DSL files via GraalVM. The security boundaries of this execution environment (what JavaScript code can and cannot access, whether it can make network calls, filesystem access, etc.) should be documented.
- **CORS configuration**: The existing `allowedOrigins` configuration and its implications for cross-origin request handling.
- **Response security**: The `finalResponse.dslWithResponseHttpStatusCode` and `dslWithoutResponseHttpStatusCode` configuration exists specifically to prevent external actors from probing backend systems via HTTP response code analysis. This security rationale is already documented in the CONFIGURATION.md but should be referenced in the security document as well.
- **Testing flow security**: The `x-ruuter-testing` header and `apiRequestTestingKey` mechanism for diagnostic testing. Document that this should be disabled or secured in production environments.

---

## 3. Improving Resql Documentation

### 3.1 How to Improve Existing Swagger UI API Documentation

#### Current State

The Swagger/OpenAPI dependency in the POM is `springdoc-openapi-ui:1.8.0` — this is the Spring Boot 2.x / `javax` namespace variant and is incompatible with the current Spring Boot 3.x / Jakarta EE 9+ stack. It must be replaced with the correct Spring Boot 3.x dependency.

The `QueryController` uses wildcard path mappings (`/{project}/**`) which intercept all root-level requests, including Swagger UI paths (`/swagger-ui/**`, `/v3/api-docs/**`), making the documentation page inaccessible.

Endpoint annotations are minimal — a single `@Operation(description=...)` per method with no parameter, response, or schema documentation. The service layer has no Javadoc.

#### Solution Proposal

##### Step 1 — Replace the Dependency

Replace the incompatible Spring Boot 2.x dependency with the correct Spring Boot 3.x version:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

##### Step 2 — Resolve the Swagger UI Path Conflict

Add an `/api` prefix to the query controller to separate application routes from framework routes:

```java
@RestController
@RequestMapping("/api")
public class QueryController {
    @PostMapping("/{project}/**")
    // ...
    @GetMapping("/{project}/**")
    // ...
}
```

This establishes a clear convention: `/api/**` is application logic, everything else (`/swagger-ui/**`, `/actuator/**`, `/datasources/**`) is infrastructure. All consumers (primarily Ruuter DSL configurations) will need to update their endpoint URLs to include the `/api` prefix.

##### Step 3 — Create an OpenAPI Configuration Class

Add a central configuration that defines the API's identity in Swagger UI:

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
                .contact(new Contact()
                    .name("Bürokratt Team")))
            .servers(List.of(
                new Server().url("/").description("Current environment")));
    }
}
```

##### Step 4 — Annotate Controllers

Each endpoint should have complete OpenAPI annotations — operation summary, parameter descriptions, response codes, and request/response examples.

Example for `QueryController` POST endpoint:

```java
@Operation(
    summary = "Execute a configured POST query",
    description = "Executes a pre-configured SQL query identified by the URL path. "
        + "The {project} segment determines the datasource, and the remaining path "
        + "identifies the query file. Parameters in the request body are passed "
        + "as named parameters to the SQL query.")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Query executed successfully",
        content = @Content(mediaType = "application/json",
            schema = @Schema(type = "array",
                example = "[{\"id\": 1, \"name\": \"example\"}]"))),
    @ApiResponse(responseCode = "404", description = "Query not found at the specified path"),
    @ApiResponse(responseCode = "500", description = "Internal error during query execution")
})
@PostMapping(value = "/{project}/**")
public List<Map<String, Object>> execute(
    @Parameter(description = "Project/datasource identifier", example = "byk", required = true)
    @PathVariable String project,
    @io.swagger.v3.oas.annotations.parameters.RequestBody(
        description = "Named parameters to pass to the SQL query. Keys must match "
            + "parameter placeholders in the query file.",
        content = @Content(mediaType = "application/json",
            schema = @Schema(type = "object",
                example = "{\"userId\": 123, \"status\": \"active\"}")))
    @RequestBody(required = false) Map<String, Object> parameters,
    HttpServletRequest request) {
    // ...
}
```

Example for the batch endpoint:

```java
@Operation(
    summary = "Execute queries in batch mode",
    description = "Executes the same query multiple times with different parameter sets. "
        + "Results are returned in the same order as the input queries.")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "All queries executed successfully"),
    @ApiResponse(responseCode = "400", description = "Request body is null or malformed"),
    @ApiResponse(responseCode = "500", description = "One or more queries failed")
})
@PostMapping("/{name}/batch")
public List<List<Map<String, Object>>> executeBatch(
    @Parameter(description = "Query name identifier", required = true)
    @PathVariable String name,
    @RequestBody BatchRequest batchRequest) {
    // ...
}
```

Example for `DataSourceController`:

```java
@Operation(
    summary = "List all configured datasources",
    description = "Returns configuration details of all registered datasources. "
        + "Passwords and sensitive credentials are excluded from the response.")
@ApiResponse(responseCode = "200", description = "Datasource list retrieved successfully",
    content = @Content(mediaType = "application/json",
        array = @ArraySchema(schema = @Schema(implementation = DataSourceConfigProperties.class))))
@GetMapping
public List<DataSourceConfigProperties> findAll() {
    // ...
}
```

##### Step 5 — Annotate DTOs and Models

All objects appearing in API requests or responses should have `@Schema` annotations with field descriptions and examples.

`HeartBeatInfo`:
```java
@Data
@Builder
@Schema(description = "Application health and version information")
public class HeartBeatInfo {
    @Schema(description = "Application name", example = "sql-ms")
    private String appName;
 
    @Schema(description = "Application version from build info", example = "0.0.1-SNAPSHOT")
    private String version;
 
    @Schema(description = "Timestamp when the application was packaged (epoch millis)",
            example = "1700000000000")
    private long packagingTime;
 
    @Schema(description = "Timestamp when the application started (epoch millis)",
            example = "1700001000000")
    private long appStartTime;
 
    @Schema(description = "Current server timestamp (epoch millis)",
            example = "1700002000000")
    private long serverTime;
}
```

`BatchRequest` (consider extracting from the inline record in `QueryController` to its own file for better visibility):
```java
@Schema(description = "Batch query execution request containing multiple parameter sets")
public record BatchRequest(
    @Schema(description = "List of parameter maps, each representing one query execution",
            requiredMode = Schema.RequiredMode.REQUIRED,
            example = "[{\"userId\": 1}, {\"userId\": 2}]")
    List<Map<String, Object>> queries
) {
    public BatchRequest {
        Objects.requireNonNull(queries);
    }
}
```

`DataSourceConfigProperties` — annotate all fields with `@Schema`. Ensure sensitive fields (passwords, connection strings with credentials) are marked with `@Schema(hidden = true)` or `@JsonIgnore` to enforce exclusion at the schema level, not just by convention.

`SavedQuery` — primarily internal (not in API responses), but should have Javadoc for developer documentation. Note the hardcoded `"byk"` datasource name as a known limitation:
```java
/**
 * Represents a loaded SQL query paired with its target datasource.
 * Queries are loaded from filesystem paths and associated with a datasource name
 * that maps to a configured DataSource via RoutingDataSource.
 *
 * Known limitation: datasource name is currently hardcoded to "byk".
 * See TODO in getDatabaseName() for multi-datasource support.
 */
public record SavedQuery(String query, String dataSourceName) { ... }
```

##### Step 6 — Add Javadoc to the Service Layer

While Javadoc does not appear in Swagger UI, it is essential for developers working on the codebase. Each service class and its public methods should document what it does, how it resolves inputs, what exceptions it throws, and any side effects.

Example for `QueryService`:
```java
/**
 * Core query execution service. Resolves REST request paths to pre-configured
 * SQL query files and executes them against the appropriate datasource.
 *
 * Query resolution: /{project}/{httpMethod}/{queryPath} maps to a SQL file
 * on the filesystem. The project determines the datasource via RoutingDataSource.
 */
@Service
public class QueryService {
 
    /**
     * Executes a named SQL query with the given parameters.
     *
     * @param project    the project/datasource identifier used for routing
     * @param httpMethod the HTTP method (GET/POST) determining the query directory
     * @param name       the query path relative to the method directory
     * @param parameters named parameters to bind to the SQL query (nullable for GET)
     * @return list of result rows, each row as a map of column names to values
     * @throws NotFoundException if no query file exists at the resolved path
     * @throws ResqlRuntimeException if query execution fails
     */
    public List<Map<String, Object>> execute(String project, String httpMethod,
                                              String name, Map<String, Object> parameters) {
        // ...
    }
}
```

Apply similar Javadoc to `SavedQueryService` (query loading, caching behavior, filesystem path resolution), `ServerInfoService` (what server information it provides), and `HeartBeatService` (health data collection, how `packagingTime` is determined).
 
---

### 3.2 How to Better Document Configuration Parameters

#### Current State

Resql has no configuration documentation. The `application.yml` file and property classes (`DataSourceConfigProperties`) are the only source of truth, requiring developers to read source code to understand what is configurable.

#### Solution Proposal

Create a dedicated `CONFIGURATION.md` covering:

**Datasource Configuration**
- How to configure database connections in `application.yml` (JDBC URL, username, password, driver class)
- How dynamic multi-datasource routing works: the relationship between `DataSourceConfigProperties`, `RoutingDataSource`, and `DataSourceContextHolder`
- How the `{project}` path variable maps to a specific datasource at runtime
- The current limitation where the datasource name is hardcoded to `"byk"` in `SavedQuery.getDatabaseName()` and what this means for multi-datasource setups
- How to add a new datasource: what configuration to add and how to register it in the routing layer
- Connection pool settings (HikariCP properties: `maximum-pool-size`, `connection-timeout`, `idle-timeout`)

**Query Configuration**
- Where SQL query files are stored (`resources/script` directory) and how they are structured
- The naming convention and directory structure that maps filesystem paths to REST endpoints
- How parameterized queries work: how request body keys bind to SQL placeholders
- How GET vs POST queries differ in parameter passing (query params vs request body)

**Security Headers**
- The `headers.content-security-policy` property: what it controls, current default value, guidance for customizing per environment

**Heartbeat Configuration**
- The `heartbeat.properties` file: what properties it contains and how `HeartBeatService` uses them

**H2 Database**
- The H2 in-memory database dependency: its purpose (development/testing environments)
- How to switch between H2 and PostgreSQL via Spring profiles

**Logging**
- The `logback-spring.xml` configuration: logging levels, output format, how to adjust per environment

Restructure the main `README.md` to link to `CONFIGURATION.md` rather than documenting configuration inline.
 
---

### 3.3 How to Better Document Use Cases and Security Risks

#### Current State

Resql has no documentation explaining its purpose, deployment model, or security posture. The `SecurityConfiguration` uses `permitAll()` with CSRF disabled and no inline comments explaining why. For a service that executes SQL queries via REST, this undocumented security model is a significant operational risk — new developers cannot distinguish intentional design from oversight.

#### Solution Proposal

Create three documents and restructure `README.md` to link to all of them.

##### SECURITY.md

**Deployment Security Model**

Resql runs exclusively within the internal Kubernetes network as a pod with no external ingress. The `permitAll()` security configuration is a deliberate architectural decision — Resql delegates authentication and authorization to the gateway layer (Ruuter). Network-level access control (Kubernetes network policies, pod isolation) enforces this boundary. This must be stated explicitly and prominently so that anyone reviewing the code understands the design intent.

**The Ruuter → Resql Trust Boundary**

The intended call chain: external clients → Ruuter (validates requests, evaluates DSL routing, enforces IP filtering, forwards to authentication endpoints) → Resql (executes database queries, trusts all requests that reach it).

This trust model has specific implications that should be documented:
- If Kubernetes network policies are misconfigured and Resql becomes externally reachable, there is no application-level security preventing unauthorized query execution
- If another internal service is compromised, it can issue arbitrary queries to Resql
- These risks should be accompanied by mitigation guidance (network policy validation, monitoring, potential future addition of internal API keys or mTLS)

**SQL Injection Prevention**

Document how query parameterization works: queries use `JdbcTemplate` with bound parameter placeholders (not string concatenation), and the SQL queries themselves are pre-configured files loaded from disk — users cannot submit arbitrary SQL. Document what validations exist on parameter values before they reach the query executor.

**CSRF Disabled**

Resql is a stateless REST API without cookie-based authentication, session management, or direct browser access. CSRF protection serves no purpose for machine-to-machine APIs — disabling it is correct for this architecture.

**Content-Security-Policy**

Document the `headers.content-security-policy` property: its purpose, current value, and guidance per environment.

##### GUIDE.md

**What Resql Is**

Resql is a centralized database query microservice that exposes pre-configured SQL queries as REST endpoints with dynamic datasource routing. It receives HTTP requests, resolves the query from the filesystem, binds parameters, executes against the appropriate database, and returns JSON results.

**Request Lifecycle**

Walk through a complete request:
1. Client sends `POST /byk/get-user` with body `{"userId": 123}`
2. `QueryController` parses the path: project = `byk`, query path = `/get-user`
3. `QueryService` resolves the query file from the filesystem
4. `RoutingDataSource` routes to the correct database based on the project name
5. `ResqlJdbcTemplate` executes the parameterized query
6. Results returned as a JSON array of row objects

Include concrete examples: simple GET query, simple POST query, batch execution, adding a new query file.

**How to Add New Queries**

Step-by-step: where to create the SQL file, how to name it, how to reference parameters, how to test it via REST.

**How to Add New Datasources**

Step-by-step: what to add in `application.yml`, how the routing key maps to `{project}`, and the current `"byk"` hardcoding limitation.

**Error Handling**

Map existing exceptions to HTTP responses:
- `NotFoundException` → 404 (query file not found)
- `InvalidQueryException` → 400 (invalid or missing parameters)
- `InvalidDirectoryException` → 500 (query directory misconfigured)
- `UnknownDataSourceNameException` → 500 (datasource not configured)
- `ResqlRuntimeException` → 500 (unexpected execution error)

##### Updated README.md

Restructure to serve as the entry point:
- Brief introduction: what Resql is (2–3 sentences)
- Architecture note: internal-only service, called primarily by Ruuter
- Links to `GUIDE.md`, `CONFIGURATION.md`, `SECURITY.md`
- Docker / build / test commands (keep existing)
- License information
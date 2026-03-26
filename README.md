# Bürokrati proovitöö

This document outlines the solution for migrating Ruuter and Resql to Java 21, addressing error handling and reliability concerns, and updating documentation for ease of onboarding and general knowledge.

Both applications are event-driven REST microservices running on Spring Boot 3.2.5:

- **Resql** — Maven-based, REST → DB query service with dynamic datasource routing. Deployed as an internal-only Kubernetes pod with no external network access. Executes pre-configured SQL queries exposed via REST endpoints.
- **Ruuter** — Gradle-based, REST → REST routing service with a GraalVM JS-based DSL engine. Deployed in two separate instances — Public Ruuter (external/UI-facing) and Private Ruuter (back-office/internal orchestration) — each with its own set of DSL files. Both are the same codebase with different configurations and DSL directories.

Both were originally built on Java 11, then upgraded to Java 17 without thorough migration review. This document covers the next step to Java 21 alongside architectural improvements to error handling, reliability, and documentation.

The deployment architecture follows a trust boundary model: external traffic reaches Ruuter, which validates requests via guard DSL files (cookie authentication or nonce validation), evaluates DSL routing rules, and forwards calls to internal services. Resql and other internal services (DataMapper, etc.) are only reachable from within the Kubernetes cluster, operating with permissive security (`permitAll()`) by design.

---

## 1. Migration from Java 17 to Java 21

### 1.1 Risks of Java 21 Adoption

#### Shared Risks

- **Stronger encapsulation** — Java 21 further tightens access to internal JDK APIs. Any `--add-opens` or `--add-exports` JVM flags that were introduced during the Java 11→17 upgrade need to be audited. These flags may exist in Dockerfiles, startup scripts, or CI pipelines. They still function on Java 21 but mask underlying issues that should be resolved properly.
- **Behavioral changes** — Minor `HashMap`/`HashSet` iteration order differences across JDK builds can cause inconsistent results in code or tests that depend on iteration order. This is relevant for Ruuter's DSL evaluation where `LinkedHashMap` is used in some paths but standard `HashMap` in others within `ScriptingHelper`.
- **Virtual threads caveat** — Java 21 introduces virtual threads as a production-ready feature. Both projects use thread-local patterns (`DataSourceContextHolder` in Resql, request context passing in Ruuter) that would behave unexpectedly with virtual threads. Adopting virtual threads is optional and not part of this migration, but should be noted for future architecture decisions. Spring Boot 3.2.5 does not enable virtual threads by default.
- **Framework-level risk is minimal** — Spring Boot 3.2.5 officially supports Java 21. No Spring-specific breaking changes are expected from the JDK version bump.

#### Resql-Specific Risks

| Risk | Severity | Detail |
|------|----------|--------|
| `id-log` library (system-scoped) | Critical | Compiled with **Java 11 bytecode**, contains `javax.servlet.http.HttpServletRequest` and `javax.servlet.http.HttpServletResponse` imports — Java EE namespace classes incompatible with Spring Boot 3.x / Jakarta EE 9+. Code review of `RestConfiguration.java` confirms both `LogHandler` and `GenericHeaderLogHandler` are **actively registered as interceptors** via `addInterceptors()` and execute on every request — these are not dormant classes. The library is also declared with Maven `system` scope and a hardcoded `systemPath`, which is inherently fragile: no version resolution, no conflict detection, and the JAR is referenced by absolute filesystem path. |
| Spring Cloud BOM (2021.0.2) | Low | Declared in POM but targets Spring Boot 2.6.x/2.7.x. Full codebase search confirmed zero usage: no `org.springframework.cloud` imports, no `spring.cloud.*` properties, no Spring Cloud annotations. Only "cloud" matches are SonarCloud CI config and already-commented-out POM entries. Dead dependency — safe to remove. |
| PostgreSQL driver (42.3.9) | Medium | Functions on Java 21 but contains known CVEs. Significantly behind latest stable (42.7.x). |
| OpenTelemetry BOM scope | Low | Declared with `<scope>runtime</scope>`, preventing it from functioning as version management. Should be moved to `<dependencyManagement>` without scope. |
| Dynamic datasource routing | None | `RoutingDataSource` extends `AbstractRoutingDataSource` with standard thread-local lookup. No behavioral changes between Java 17 and 21. |

#### Ruuter-Specific Risks

| Risk | Severity | Detail |
|------|----------|--------|
| GraalVM JS Engine (23.0.1) | Medium | `ScriptEngineConfiguration` uses JSR-223 SPI (`ScriptEngineManager().getEngineByName("graal.js")`) in interpreter-only mode. The `javax.script.*` imports are **JDK-native** (not Jakarta) — safe. The risk is version targeting: GraalVM `23.0.x` targets JDK 20; JDK 21 alignment requires `23.1.x` or `24.x`. Critical because `ScriptingHelper` is the **core DSL engine** — every request flows through `engine.eval()`. Fix is straightforward: bump version if smoke test fails. |
| Apache HttpClient 4.5.13 | Medium | EOL. Functions on Java 21 but receives no patches. Replacement is `httpclient5`. Central dependency for `ExternalForwardingHelper` and `HttpHelper`. |
| WireMock `wiremock-jre8` (2.35.1) | Medium | Deprecated variant. Java 21 requires `org.wiremock:wiremock:3.x` (new Maven group ID). Test infrastructure only. |
| Mockito version conflict | Low | `mockito-inline:4.9.0` alongside `mockito-core:5.6.0`. Mockito 5.x includes inline mocking by default — `mockito-inline` is redundant and the version mismatch causes unpredictable test behavior. |
| All tests disabled | High | `exclude '**/*'` in Gradle means zero tests run during builds. No safety net for migration. Must be addressed before any Java 21 work. |
| Missing Java version target | Medium | No `sourceCompatibility`/`targetCompatibility` in `build.gradle`. Bytecode version depends on whichever JDK runs the build — implicit and fragile. |

---

### 1.2 Impact on Spring/Jakarta Stack

The `javax` → `jakarta` namespace migration (required by Spring Boot 3.x / Jakarta EE 9+) is **already complete** in both projects.

**Resql**: All servlet imports use `jakarta.servlet`. The only `javax` import is `javax.sql.DataSource` in `ResqlJdbcTemplate` — part of the JDK's `java.sql` module, not subject to Jakarta rename.

**Ruuter**: `javax.script.ScriptEngine`, `javax.script.Bindings`, and `javax.script.ScriptEngineManager` in the scripting layer belong to the JDK's `java.scripting` module — JDK-native, will never be renamed.

Java 21 introduces no new Jakarta EE requirements. Spring Boot 3.4.x offers virtual thread auto-config but upgrading is optional.

**Exception**: The `id-log` library in Resql contains `javax.servlet` (Java EE) classes. `RestConfiguration` actively registers both interceptors on every request. The project uses only `GenericHeaderLogHandler` and `LogHandler` — simple logging interceptors. Solution: rewrite these two classes (~50–100 lines) in-project using `jakarta.servlet` imports and remove the `id-log` dependency entirely. This eliminates the `javax.servlet` incompatibility, the Java 11 bytecode mismatch, and the fragile system-scoped Maven dependency.

---

### 1.3 Backward Compatibility

#### API Contract Level

REST API contracts are unaffected — external consumers will not notice any difference. Request/response formats, endpoints, and behavior remain identical.

#### Runtime Environment Alignment

Build-time and runtime JDK must both be Java 21.

**Resql** (WAR packaging) — the servlet container must support Java 21. Spring Boot 3.2.5 embedded Tomcat does. If deployed to an external Tomcat, that instance must also be updated.

**Ruuter** (JAR packaging) — self-contained with embedded server. Only the JDK on the target host/container needs to be Java 21.

#### Build Configuration

**Resql** — update `pom.xml`:
```xml
<java.version>21</java.version>
<maven.compiler.source>21</maven.compiler.source>
<maven.compiler.target>21</maven.compiler.target>
```

**Ruuter** — add to `build.gradle`:
```groovy
java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}
```

#### Docker and CI Pipeline

- Docker base images must be updated to JDK 21 (e.g., `eclipse-temurin:21-jre`). Both `Dockerfile` and `docker-compose.yml` must be reviewed.
- CI pipeline JDK version must match.
- Ruuter's Node.js tooling (`package.json`, Husky, bump-version.sh) is unaffected.
- Gradle/Maven wrapper scripts are JDK-version-agnostic.

#### Dependency Compatibility

| Dependency | Project | Current | Java 21 Status | Action |
|-----------|---------|---------|----------------|--------|
| Spring Boot | Both | 3.2.5 | Officially supported | None |
| PostgreSQL Driver | Resql | 42.3.9 | Works, has CVEs | Upgrade to 42.7.x |
| GraalVM JS | Ruuter | 23.0.1 | Likely works, targets JDK 20 | Smoke test; bump to 23.1.x if needed |
| Apache HttpClient | Ruuter | 4.5.13 | Works, EOL | Migrate to httpclient5 |
| WireMock | Ruuter | 2.35.1 (jre8) | Deprecated variant | Migrate to org.wiremock:wiremock:3.x |
| Mockito | Ruuter | 5.6.0 + inline 4.9.0 | Version conflict | Remove mockito-inline |
| id-log | Resql | 1.0.0-SNAPSHOT (J11) | Incompatible (javax.servlet) | Rewrite and remove |

---

### 1.4 Testing the Migration

**Step 1 — Static Analysis**

Run `jdeps --jdk-internals` on all artifacts to detect internal API usage:
```bash
jdeps --jdk-internals libs/id-log-1.0.0-SNAPSHOT.jar
jdeps --jdk-internals target/sql-ms.war
jdeps --jdk-internals build/libs/*.jar
```

**Step 2 — Compile on JDK 21** without code changes to isolate compilation-level issues.

**Step 3 — Dependency Conflict Analysis**
```bash
mvn dependency:tree -Dverbose          # Resql
gradle dependencies --configuration runtimeClasspath  # Ruuter
```

**Step 4 — Test Suites**
- Resql: Run existing integration tests (`QueryControllerIntegrationTest`, etc.) on JDK 21.
- Ruuter: **Remove `exclude '**/*'`** first, then run tests. Without this, all validation is manual.

**Step 5 — Targeted Smoke Testing**
- Ruuter: Boot on JDK 21, send a request triggering DSL script evaluation (`ScriptingHelper.evaluateScripts()` → `engine.eval()`). If this passes, the highest-risk item is cleared.
- Resql: Execute queries through the REST API, including dynamic datasource routing scenarios.

**Step 6 — Staging Deployment** — Full end-to-end testing of the Ruuter → Resql call chain in a staging environment mirroring production.

---

### 1.5 Error Handling and Reliability

#### 1.5.1 Distinguishing Error Types

Resql has a `GlobalExceptionHandler` and typed exceptions (`InvalidDirectoryException`, `InvalidQueryException`, `ResqlRuntimeException`, `UnknownDataSourceNameException`). However, all are bare `RuntimeException` subclasses with only a message string — no error codes, no HTTP status mapping, no structured fields. Ruuter has its own set (`InvalidDslException`, `InvalidDslStepException`, `InvalidHttpMethodTypeException`, `LoadDslsException`, `ScriptEvaluationException`) plus hardcoded error codes (`E_unknown`, `E_null`, `E_script`).

Both projects should standardize on Spring Boot 3.x's built-in RFC 7807 `ProblemDetail` for uniform, machine-parseable error responses:

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

This produces standardized JSON:
```json
{"type":"about:blank","title":"Not Found","status":404,"detail":"Query 12345 not found","errorCode":"RESOURCE_NOT_FOUND"}
```

Ruuter's existing error codes (`E_unknown`, `E_null`, `E_script`) should be preserved and integrated into the `ProblemDetail` format.

#### 1.5.2 Handling Unexpected Errors

Catch-all handler that logs full context server-side, returns only a correlation ID to the client:

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

For the Ruuter → Resql call chain, propagate `X-Correlation-Id` header across service boundaries. This integrates with Ruuter's existing OpenSearch logging, making cross-service error tracing possible with a single identifier.

#### 1.5.3 Retry Strategy

Primarily relevant for **Ruuter** where `ExternalForwardingHelper` and `HttpHelper` make outbound REST calls.

**Spring Retry** (simpler, annotation-driven):
```java
@Retryable(
    retryFor = {UpstreamServiceException.class, ConnectException.class},
    noRetryFor = {ValidationException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 500, multiplier = 2.0, maxDelay = 5000))
public ResponseEntity<String> callUpstream(String url, Object payload) { ... }
```

**Resilience4j** (more control, includes circuit breaker):
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

Ruuter already has `externalForwarding.proceedPredicate.httpStatusCode` — retry logic should integrate with this existing mechanism.

Key principles: retry only recoverable errors (connection failures, 502/503/504 — never 4xx), exponential backoff, idempotency awareness (safe for GET/PUT, cautious with POST).

**Resql** — HikariCP handles connection pool retry automatically. Additional retry only needed for transient DB errors (connection spikes).

---

## 2. Improving Ruuter Documentation

### 2.1 How to Enhance or Restructure Documentation for Newer Developers

Analysis of existing documentation identified the following: the `README.md` contains only Docker, build, and test commands with links to the Guide and Configuration — no introduction, no architecture context, no explanation of what Ruuter is. The `GUIDE.md` has solid DSL content (file structure, request types, responses, optional parameters, step types, scripting) but a flat structure with no progression from overview to detail. The `CONFIGURATION.md` is the most complete document, thoroughly covering DSL paths, exception handling, CORS, external forwarding, internal requests, OpenSearch logging, response limits, testing mode, and more.

#### Solution Proposal

**README.md** — restructure as the entry point:
- Add an introduction explaining what Ruuter is: a service that executes custom DSL files to orchestrate REST-based workflows, acting as the gateway in the Bürokratt architecture
- Add a high-level architecture overview: external traffic → Ruuter → internal services (Resql, DataMapper, etc.). Explain the dual deployment model — Public Ruuter and Private Ruuter are separate deployments of the same codebase, each with its own DSL files, isolated at the infrastructure level.
- Add a quickstart section with a minimal DSL example including the mandatory declaration block, demonstrating request → guard → DSL → response
- Keep Docker/build/test commands below the introduction
- Link to GUIDE.md, CONFIGURATION.md, and a new SECURITY.md

**GUIDE.md** — restructure for progressive disclosure:
- **Top**: General overview, directory structure (GET/POST as method directories with guard files), declaration block as mandatory first element. Distinguish regular DSL files from template DSL files (predefined flows for internal calls).
- **Middle**: Detailed DSL writing — step types (`return`, `assign`, `mock`, `http-get`, `http-post`, `conditional-jump`, `template`), JavaScript scripting, parameter passing, jumps, skip, sleep. How guard files work as authentication middleware.
- **Bottom**: Advanced topics — DSL reloading, default services, mock steps, batch processing, step-level recursion overrides.
- Remove content duplicating CONFIGURATION.md; reference SECURITY.md for security topics.

**CONFIGURATION.md** — minor improvements:
- Add introductory text on where config lives (`application.yml`) and how to override per environment
- Ensure all examples include default values
- Cross-reference SECURITY.md for security-related properties

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

Critical behavior: **any incoming field not listed in the allowlist is set to `null`**, even if the caller provides it. This is silent — no error, no warning. This is the most common source of bugs when writing new DSLs. The `namespace` field (e.g., `backoffice`, `service`, `analytics`) is a project identifier grouping DSLs by subsystem — it is not an access control mechanism.

**DSL Validation**
- Invalid YAML syntax — caught at startup or request time?
- Non-existent step references or circular jumps — interaction with `maxStepRecursions`
- Disallowed file types in DSL directory — application refuses to start (document prominently)
- `LoadDslsException` behavior and operator actions

**Script Validation**
- JavaScript evaluation failures — `ScriptEvaluationException`
- `stopInCaseOfException` behavior (halt vs continue)
- Error codes: `E_unknown`, `E_null`, `E_script` from `DSLExecutionException`

**HTTP Method Validation**
- Response for methods not in `allowedMethodTypes`
- Interaction between declaration `method` field, directory-level method, and global config

**Testing Mode**
- `x-ruuter-testing` header with `apiRequestTestingKey` for diagnostic error responses
- Error object structure: `dslName`, `stepName`, `causeCode`, `message`

### 2.3 Security Documentation

Create a dedicated `SECURITY.md` linked from README.md and GUIDE.md.

**Architecture-Level Security Model**
- Ruuter is the public-facing entry point enforcing the trust boundary
- Internal services operate with `permitAll()` — reachable only within Kubernetes
- Public Ruuter and Private Ruuter are **separate deployments** of the same codebase with different DSL files. The public/private boundary is enforced at infrastructure level (separate pods/images), not within the application.

**Authentication: Guard DSL Files**

Authentication is implemented at the DSL level via guard files at the root of each method directory (GET/POST). Every request passes through the guard before reaching the target DSL. The guard implements a multi-path flow:

1. **Nonce-based authentication** — If the request contains `x-ruuter-nonce` header (or `ruuter-nonce` query param), validated against Resql as a single-use token. Used by internal systems (cron jobs, scheduled tasks) that operate within the internal network and cannot obtain browser cookies.
2. **Cookie-based authentication** — If no nonce, checks for session cookie and validates via template call to the authentication layer (`check-user-authority`). Standard path for UI/browser requests.
3. **Rejection** — Neither valid nonce nor valid cookie → 403.

Both Public and Private Ruuter use guard files. The guard implementation is itself a DSL — authentication logic is configurable and auditable without code changes.

As a potential future improvement, service-to-service auth could use mTLS or shared secrets at the Spring Security level, removing nonce management overhead. However, this would move auth logic outside the DSL engine, contradicting the architecture where all request processing flows through DSLs.

**Internal Service Access Control**
- `allowedIPs` and `allowedURLs` for `internal` subdirectory DSLs
- IP/referrer filtering behavior and unauthorized access response
- Allowlist maintenance practices across environments

**External Forwarding as Authentication**
- `externalForwarding` forwards requests to validation endpoint before DSL processing
- `proceedPredicate.httpStatusCode` determines proceed/reject
- Behavior when forwarding endpoint is unreachable

**Script Evaluation Security**
- GraalVM interpreter-only mode security boundaries
- What JS code can/cannot access (network, filesystem, isolation between evaluations)
- Script injection risk from untrusted input flowing into evaluation

**CORS, Response Code Masking, Testing Mode**
- `allowedOrigins` configuration per environment
- `finalResponse` status code masking to prevent backend probing
- `x-ruuter-testing` header exposes internal details — disable or secure in production

---

## 3. Improving Resql Documentation

### 3.1 How to Improve Existing Swagger UI API Documentation

Analysis of the codebase identified three issues: the Swagger dependency in the POM is `springdoc-openapi-ui:1.8.0` — the Spring Boot 2.x / `javax` namespace variant, incompatible with the current Boot 3.x stack. The `QueryController` uses wildcard mappings (`/{project}/**`) intercepting all root-level requests including Swagger UI paths, making the documentation page inaccessible. Endpoint annotations are minimal — one `@Operation(description=...)` per method with no parameter, response, or schema documentation. The service layer has no Javadoc.

#### Solution Proposal

##### Step 1 — Replace the Dependency

Replace the incompatible dependency with the Spring Boot 3.x version:
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

##### Step 2 — Resolve the Path Conflict

Add an `/api` prefix to the query controller to separate application routes from framework routes:
```java
@RestController
@RequestMapping("/api")
public class QueryController { ... }
```

This establishes a clear convention: `/api/**` is application logic, everything else (`/swagger-ui/**`, `/actuator/**`, `/datasources/**`) is infrastructure. Consumers (primarily Ruuter DSL configurations) will need to update endpoint URLs.

##### Step 3 — Create an OpenAPI Configuration Class

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

Each endpoint should have complete annotations — summary, parameters, response codes, examples:

```java
@Operation(
    summary = "Execute a configured POST query",
    description = "Executes a pre-configured SQL query identified by the URL path. "
        + "The {project} segment determines the datasource, and the remaining path "
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
    @PathVariable String project,
    @io.swagger.v3.oas.annotations.parameters.RequestBody(
        description = "Named parameters to bind to the SQL query.",
        content = @Content(mediaType = "application/json",
            schema = @Schema(type = "object",
                example = "{\"userId\": 123, \"status\": \"active\"}")))
    @RequestBody(required = false) Map<String, Object> parameters,
    HttpServletRequest request) { ... }
```

Apply the same pattern to the batch endpoint, `DataSourceController`, and `HeartBeatController`.

##### Step 5 — Annotate DTOs and Models

```java
@Data
@Builder
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

For `DataSourceConfigProperties` — mark sensitive fields (passwords, connection strings) with `@Schema(hidden = true)` or `@JsonIgnore` to enforce exclusion at the schema level. For `BatchRequest` — consider extracting from the inline record in `QueryController` for visibility and annotate with `@Schema`.

##### Step 6 — Add Javadoc to the Service Layer

```java
/**
 * Core query execution service. Resolves REST paths to pre-configured
 * SQL files and executes them against the appropriate datasource.
 *
 * Resolution: /{project}/{httpMethod}/{queryPath} → SQL file on disk.
 * Project determines datasource via RoutingDataSource.
 */
@Service
public class QueryService {
    /**
     * @param project    datasource identifier for routing
     * @param httpMethod HTTP method determining query directory
     * @param name       query path relative to the method directory
     * @param parameters named parameters to bind (nullable for GET)
     * @return result rows as maps of column names to values
     * @throws NotFoundException if query file not found
     * @throws ResqlRuntimeException if execution fails
     */
    public List<Map<String, Object>> execute(...) { ... }
}
```

Apply to `SavedQueryService` (loading, caching, path resolution), `ServerInfoService`, and `HeartBeatService` (`packagingTime` sourced from `heartbeat.properties` via `PackageInfoConfiguration`).

---

### 3.2 How to Better Document Configuration Parameters

Resql has no configuration documentation. Create a `CONFIGURATION.md` covering:

**Datasource Configuration**
- How to configure connections in `application.yml` via `sqlms.datasources` prefix (bound to `List<DataSourceConfigProperties>`)
- How dynamic multi-datasource routing works: `DataSourceConfiguration` builds a `RoutingDataSource` from all configured datasources → `DataSourceContextHolder` (thread-local) sets the active datasource per request → `RoutingDataSource.determineCurrentLookupKey()` reads it
- Current limitation: `SavedQuery.getDatabaseName()` is hardcoded to `"byk"` regardless of the project parameter — multi-datasource routing at the query level is not yet functional
- Timezone handling: `DataSourceConfiguration` sets `SET TIME ZONE` via HikariCP's `connectionInitSql`, defaulting to `Europe/Tallinn`

**Connection Pool (HikariCP)**
- Relevant properties: `maximum-pool-size`, `connection-timeout`, `idle-timeout`, `max-lifetime`
- Environment-specific guidance: development (small pool, short timeouts) vs production (larger pool, tuned for expected load)
- Impact on reliability: an exhausted pool causes all requests to queue — critical for a service where every REST call holds a connection for the duration of query execution

**Query Configuration**
- SQL files stored under a configurable path, organized by project and HTTP method
- Naming convention: filesystem path directly maps to REST endpoint
- Named parameter syntax (`:paramName`) for parameterized queries
- GET vs POST: query parameters vs request body

**Security Headers, Heartbeat, H2, Logging**
- `headers.content-security-policy` property and per-environment guidance
- `heartbeat.properties`: `app.name`, `app.version`, `app.packaging.time` — sourced at build time via `PackageInfoConfiguration`
- H2 in-memory database for development/testing, profile-based switching to PostgreSQL
- `logback-spring.xml` configuration and logging levels

Restructure `README.md` to link to `CONFIGURATION.md`.

---

### 3.3 How to Better Document Use Cases and Security Risks

Resql's `SecurityConfiguration` uses `permitAll()` with CSRF disabled and no inline comments. For a service executing SQL via REST, this undocumented security posture is an operational risk. Create three documents and restructure `README.md` to link to all of them.

#### SECURITY.md

**Deployment Security Model**

Resql runs exclusively within the internal Kubernetes network as a pod with no external ingress. The `permitAll()` configuration is a deliberate architectural decision — Resql delegates authentication to the gateway layer (Ruuter). Network-level access control (Kubernetes network policies, pod isolation) enforces this boundary. This must be stated prominently.

**The Ruuter → Resql Trust Boundary**

Intended call chain: external clients → Ruuter (guard DSL authentication, DSL routing, IP filtering) → Resql (trusts all requests).

Implications:
- If Kubernetes network policies are misconfigured and Resql becomes externally reachable, there is no application-level security
- A compromised internal service can issue arbitrary queries
- Mitigations: network policy validation, monitoring, potential future mTLS or internal API keys

**SQL Injection Prevention**

`ResqlJdbcTemplate` extends `NamedParameterJdbcTemplate` and uses `MapSqlParameterSource` — parameters are bound via prepared statements, never concatenated. SQL queries are pre-configured files loaded from disk — users cannot submit arbitrary SQL. However, the absence of input validation on parameter *values* before they reach the query executor should be noted.

**Allowed and Forbidden SQL Operations**

There is no application-level restriction on SQL operation types. Any valid SQL can be placed in a query file:

*Allowed (intended use):*
- Parameterized SELECT queries (primary use case)
- Parameterized INSERT/UPDATE/DELETE — `queryOrExecute()` auto-detects writes via absent `ResultSetMetaData`

*Forbidden (must be enforced by code review and convention):*
- DDL operations (CREATE, DROP, ALTER, TRUNCATE) — no runtime guard prevents these from being exposed via REST
- Unbounded SELECT queries without LIMIT — results load entirely into `List<Map>` in memory, risking `OutOfMemoryError`

**Known Architectural Risks**

- **No query timeout** — `queryOrExecute()` sets no statement timeout. A runaway query holds a HikariCP connection indefinitely, eventually exhausting the pool. Mitigation: set `statement_timeout` at PostgreSQL level or via `connectionInitSql`.
- **No result set size limit** — query results map directly into memory with no pagination or cap. Mitigation: enforce `LIMIT` in all SELECT queries.
- **Batch operations without transactions** — `executeBatch` in `QueryController` calls `executePost` in a stream with no `@Transactional`. Partial failures leave data inconsistent — first two queries commit, third fails, fourth and fifth never execute. Mitigation: wrap in `@Transactional`.
- **ThreadLocal datasource context not cleaned** — `DataSourceContextHolder.setDataSourceName()` is called per request but `clearDataSourceName()` is never called in the request flow. In a thread pool, the previous request's datasource can leak to the next request on the same thread. Mitigation: clear context in a servlet filter or `@After` advice.

**CSRF Disabled**

Resql is a stateless REST API without cookie auth, sessions, or direct browser access. CSRF protection serves no purpose for machine-to-machine APIs — disabling it is correct.

**CORS**

`RestConfiguration` defaults `cors.allowedOrigins` to `*` (wildcard). For an internal-only service this is acceptable but should be documented as intentional.

#### GUIDE.md

**What Resql Is**

Centralized database query microservice exposing pre-configured SQL queries as REST endpoints with dynamic datasource routing.

**SQL File Lifecycle**

Walk through the complete path from file to response:

1. Developer writes a `.sql` file with named parameters (`:paramName`)
2. File placed in directory structure: `{config-path}/{project}/{METHOD}/query-name.sql`
3. At startup, `SavedQueryService` loads all SQL files into `SavedQuery` records (query text + datasource name)
4. Client sends `POST /byk/get-user` with body `{"userId": 123}`
5. `QueryController` parses URL: project=`byk`, method=`POST`, path=`/get-user`
6. `QueryService` retrieves the `SavedQuery`, sets datasource context via `DataSourceContextHolder`
7. `RoutingDataSource` routes to the correct database
8. `ResqlJdbcTemplate.queryOrExecute()` executes with bound parameters, auto-detects SELECT vs write
9. Results returned as JSON with `snake_case` → `camelCase` column name conversion

Key implications: no hot-reload (restart required for new queries), no syntax validation at startup, path-based routing where directory structure is the API contract.

**Practical examples**: simple GET query, POST query, batch execution, adding a new query file, adding a new datasource.

**Error Handling**

Exception-to-HTTP mapping:
- `NotFoundException` → 404 (query file not found)
- `InvalidQueryException` → 400 (invalid parameters)
- `InvalidDirectoryException` → 500 (directory misconfigured)
- `UnknownDataSourceNameException` → 500 (datasource not configured, includes the unrecognized name in message)
- `ResqlRuntimeException` → 500 (unexpected execution error)

#### Updated README.md

Restructure as entry point:
- Brief introduction (2–3 sentences)
- Architecture note: internal-only, called by Ruuter
- Links to GUIDE.md, CONFIGURATION.md, SECURITY.md
- Docker/build/test commands
- License
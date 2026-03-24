# Bürokrati provtöö.

This document outline solution on migrating Ruuter and Resql to Java 21 and update documentation for ease of unboarding and general knowledge

## 1. Migration from Java 17 to Java 21

### 1. Risks of Java 21 Adoption

#### Shared Risks

- **Stronger encapsulation** — any `--add-opens` flags from the Java 11→17 upgrade need auditing
- **Behavioral changes** — minor `HashMap`/`HashSet` iteration order differences can cause flaky tests
- Spring Boot 3.2.5 officially supports Java 21, so framework-level risk is minimal

### Resql

| Risk | Severity | Detail |
|------|----------|--------|
| `id-log` library (system-scoped) | High | Contains `javax.servlet` imports, compiled with Java 11 bytecode. Works now because the affected classes (`GenericHeaderLogHandler`, `LogHandler`) are likely never fully invoked at runtime. |
| Spring Cloud BOM (2021.0.2) | Medium | Wrong version for Boot 3.2.5. Analysis confirmed zero Spring Cloud usage — dead dependency. |
| PostgreSQL driver (42.3.9) | Medium | Works on Java 21 but has known CVEs. |
| OpenTelemetry BOM runtime scope | Low | Declared with `runtime` scope, doesn't function as version management. |

### Ruuter

| Risk | Severity | Detail |
|------|----------|--------|
| GraalVM JS Engine (23.0.1) | High | `js-scriptengine` bridges to `javax.script` API. If used for dynamic JS evaluation in routing logic, this is critical functionality requiring thorough testing. |
| Apache HttpClient 4.5.13 | Medium | EOL. Works on 21 but should migrate to `httpclient5`. Central dependency for a REST→REST service. |
| WireMock `wiremock-jre8` | Medium | Deprecated variant. Needs `org.wiremock:wiremock:3.x`. |
| Mockito version conflict | Low | `mockito-inline:4.9.0` conflicts with `mockito-core:5.6.0`. Inline mocking is built into Mockito 5.x. |
| All tests disabled | High | `exclude '**/*'` means zero tests run. No safety net for migration. |
| Missing Java version target | Medium | No `sourceCompatibility`/`targetCompatibility` declared. |

---

## 2. Impact on Spring/Jakarta Stack

Both projects already completed the `javax` → `jakarta` namespace migration. `Resql` uses `jakarta.servlet` imports; the only `javax` found is `javax.sql.DataSource` which is JDK-native and stays as-is.

Java 21 introduces no new Jakarta EE requirements. Spring Boot 3.4.x offers better Java 21 support (virtual threads auto-config) but upgrading is optional.

**Exception**: `id-log` in Resql contains `javax.servlet` classes. Only two classes are used — recommend rewriting them in-project with `jakarta.servlet` imports (~50–100 lines) and removing the dependency.

---

## 3. Backward Compatibility

REST API contracts are unaffected — external consumers won't notice the JDK change.

Build and runtime JDK must both be 21. For Resql (WAR), the servlet container must also support Java 21 (Spring Boot 3.2.5 embedded Tomcat does).

**Resql** — update POM:
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

---

## 4. Testing the Migration

1. **Static analysis** — run `jdeps --jdk-internals` on all JARs (especially `id-log`) to detect illegal internal API usage
2. **Compile on JDK 21** without code changes — catches most issues immediately
3. **Dependency tree analysis** — `mvn dependency:tree` / `gradle dependencies` to find conflicts
4. **Run existing tests on JDK 21** — Ruuter must first re-enable its disabled tests
5. **Staging deployment** — smoke tests, core business scenarios, GraalVM JS evaluation (Ruuter), DB queries (Resql)

---

## 5. Distinguishing Error Types

Implement a typed exception hierarchy with Spring Boot 3.x's built-in RFC 7807 `ProblemDetail` support:

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

Produces standardized JSON:
```json
{"type":"about:blank","title":"Not Found","status":404,"detail":"Query 12345 not found","errorCode":"RESOURCE_NOT_FOUND"}
```

Apply `@RestControllerAdvice` + `ProblemDetail` consistently across both projects.

---

## 6. Handling Unexpected Errors

Catch-all handler that logs full context internally but returns only a correlation ID to the client:

```java
@ExceptionHandler(Exception.class)
public ProblemDetail handleUnexpected(Exception ex, HttpServletRequest request) {
    String correlationId = UUID.randomUUID().toString();
    log.error("Unexpected error [correlationId={}, path={}]",
        correlationId, request.getRequestURI(), ex);

    ProblemDetail detail = ProblemDetail.forStatusAndDetail(
        HttpStatus.INTERNAL_SERVER_ERROR,
        "Internal error. Reference: " + correlationId);
    detail.setProperty("correlationId", correlationId);
    return detail;
}
```

For Ruuter (REST→REST), propagate `X-Correlation-Id` header to downstream services so errors can be traced across the chain.

---

## 7. Retry Strategy

Primarily relevant for **Ruuter** (REST→REST router). Two options:

**Spring Retry** (simpler):
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

Key rules: retry only recoverable errors (connection failures, 502/503/504 — never 4xx), use exponential backoff, and only retry idempotent operations (GET, PUT). Be cautious with POST.

**Resql** — DB retry is mostly handled by HikariCP connection pool. Consider retry only for transient DB errors.

---

## 2. Update Resql Documentation

### Define which sections of documentation should be updated.

Main Readme File
- Initial README file does not hold any information on what Ruuter is or does, it's made from generic information. For users who already work with it, does not hold much value, but for new developers to join and start working with it, its quite unclear on its purpose.
- The initial structure is there, but its missing an entire Intro block what would be more expressive about ruuter.
- Guide it self have solid information but structure might be updated, adding all possible elements to bottom of it and more general over view to the guide file.

### How to enhance or restructure documentation for newer developers.

- Main Readme File holds no information of Ruuter overview to give more understanding
- Configuration docs should be more explicit on configuration parameters.
- Guide file should hold more General examples to outline Ruuters functionality and purpose
- Would be Good to include big example file within main Readme or Guide file to emphasis Ruuter

### How to document use and validation of inputs

- Security aspect is fully missed from documents.
- As currently GUIDE holds use related files, for consistency we can keep all related information in this file.
- As SECONDARY option we can make an input validation document and Link location from inside guide
- As SECONDARY option for security aspect we can put extra document covering security aspect of using ruuter and link from inside guide

## 3. Improving Resql Documentation

### How to improve existing Swagger UI API Documentation

- Update dependency to include a proper library to display swagger page
- Fix Controllers to either have a separate swagger controller, or ensure that current controller is not blocking access to the page by adding something like  /api to root of controller.
- Current version of code base does not have much configurations
- configure all files via annotations or somehow differently

### How to better document configuration parameters

- Current documentation hold no relative information, so we need to have a separate file called Configuration.md
- Refactor main md file to navigate to configuration

### How to better document use case and security risks

- Current document hold no relative information on use case
- Current document hold no relative infromation on security aspect
- Make a separate file Security
- Make a separate file Configuration
- Make a separate file Guide

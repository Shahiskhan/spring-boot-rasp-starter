# Embedded RASP Spring Boot Starter

A Java 25, Spring Boot 3.5.16 starter that screens HTTP input inside a Spring MVC application. Add one dependency to enable eight built-in detectors, request blocking, policy configuration, Micrometer counters, and a health indicator. The sample application contains safe echo endpoints: it never executes submitted commands, expressions, SQL, XML entities, or database queries.

This is a defense-in-depth request screening library. Its heuristics do not replace parameterized queries, contextual output encoding, secure XML parsers, authorization, or safe process and filesystem APIs. It does not instrument JVM execution sinks or track tainted values through application code. Read [SECURITY.md](SECURITY.md) before deployment.

The project uses Java **25** as requested and Spring Boot **3.5.16**. The original Spring Boot 3.2 baseline was updated because [Spring Boot 3.5 supports Java 25](https://docs.spring.io/spring-boot/3.5/system-requirements.html). Spring announced 3.5.16 as the [last open-source release of the 3.5 line](https://spring.io/blog/2026/06/25/spring-boot-3-5-16-available-now/). Before an Internet-facing deployment, review the dependency tree, security advisories, and a migration to a supported Spring Boot line. The build and tests establish behavior for the pinned stack; they do not establish that every transitive dependency is free of known vulnerabilities.

## Build and run

Requirements: a **JDK 25**, Maven 3.8.6 or newer, and access to Maven Central on the first build. No database, external WAF, Docker daemon, or cloud credentials are needed for the normal test suite.

```sh
mvn clean verify
mvn install
mvn -pl rasp-samples spring-boot:run
```

On Windows, the helper selects an installed JDK 25 from `JAVA_HOME`, standard installation directories, or the workspace-local toolchain when present:

```powershell
.\scripts\maven.ps1 clean verify
.\scripts\maven.ps1 install
.\scripts\maven.ps1 -pl rasp-samples spring-boot:run
```

The helper also sets `jdk.net.unixdomain.tmpdir` to `.tools/socket-tmp` for Maven and its child JVMs, avoiding virtualized Windows temporary socket paths; it restores the previous `JDK_JAVA_OPTIONS` afterward and changes no machine-wide setting. Quote `-D` arguments in PowerShell, for example `'-Drasp.perf=true'`.

The install step makes the local starter and its dependencies available when running only the sample module. Alternatively, after `mvn clean verify`, run the packaged application:

```sh
java -jar rasp-samples/target/rasp-samples-1.0.0-SNAPSHOT.jar
```

The sample listens on port 8080; set `PORT` to override it. The sample intentionally scans requests from localhost. It has no authentication and exposes health details and metrics for demonstration; secure these endpoints in a consumer application.

## Install in a consumer

These coordinates are built locally; this repository does not claim a published Maven Central release. Run `mvn install` or publish the library modules to your organization's artifact repository, then add:

```xml
<dependency>
  <groupId>dev.rasp</groupId>
  <artifactId>rasp-spring-boot-starter</artifactId>
  <version>1.0.0-SNAPSHOT</version>
</dependency>
```

The starter targets the servlet Spring MVC stack and uses Spring Boot 3 `AutoConfiguration.imports`; consumers do not need component scanning or an `@EnableRasp` annotation. A pure WebFlux/Netty application is **not supported** and does not receive servlet protection. The core engine can be embedded independently, but reactive body interception is not included.

Add `spring-boot-starter-actuator` in the consumer for Boot's registry, metrics endpoints, and health endpoint. The RASP starter does not expose management endpoints by itself.

## Architecture

```text
HTTP request
    |
    v
RaspFilter -- path/IP exemptions -- bounded replayable body
    |
    v
URI, headers, query/form fields, JSON values, XML text
    |
    v
RaspEngine -> ordered ThreatDetector strategies (8 built-ins)
    |                 |
    |                 +-- cached compiled patterns + bounded normalization
    v
ThreatScorer -> PolicyEngine -> ALLOW / LOG / BLOCK
    |                             |
    +-- metadata logs/events      +-- structured error response
    +-- Micrometer counters       +-- STRICT IP violation window
    |
    v
Spring MVC controller -> annotated service boundary (@RaspProtect / @RaspScan)
```

Modules:

| Module | Purpose |
| --- | --- |
| `rasp-core` | Spring-free detectors, extraction helpers, normalization, policy, and engine API |
| `rasp-spring-boot-autoconfigure` | Validated properties, servlet interception, method argument scanning, events, health, and metrics |
| `rasp-spring-boot-starter` | Consumer dependency aggregator |
| `rasp-samples` | Runnable demonstration application and acceptance tests |

Built-in detectors do CPU work on the request thread. They perform no database, filesystem, DNS, or HTTP calls. Reading a servlet request body is synchronous and follows the servlet container's request timeout and size controls; "no blocking I/O" cannot literally apply to receiving a servlet body. Any custom detector used by the filter must remain bounded and avoid network or other slow I/O.

For non-servlet integrations, `RaspEngine.scanAsync(context, executor)` returns a `CompletableFuture` using a caller-owned executor. Supply a bounded executor and manage its lifecycle; this API does not add a timeout, cancel individual detectors, or turn servlet body receiving into nonblocking I/O.

## Try the sample

Use `curl` (`curl.exe` in Windows PowerShell where `curl` is an alias). These inputs are inert test strings sent to the local demo:

```sh
# Ordinary input and an opaque Authorization header: 200
curl --get --data-urlencode 'name=John' http://localhost:8080/api/users
curl --get --data-urlencode 'name=John' -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJKb2huIn0.signature' http://localhost:8080/api/users

# SQL injection: 403
curl --get --data-urlencode "q=' OR 1=1--" http://localhost:8080/api/search

# XSS: 403
curl -H 'Content-Type: application/json' --data '{"comment":"<script>alert(1)</script>"}' http://localhost:8080/api/comment

# SpEL: 403; this endpoint never evaluates expressions
curl --get --data-urlencode "expr=T(java.lang.Runtime).getRuntime().exec('id')" http://localhost:8080/api/eval

# Check the RASP health component and bounded-cardinality counters
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/metrics/rasp.threats.detected
curl http://localhost:8080/actuator/metrics/rasp.threats.blocked
```

The default rejection body includes `error` and a server-generated `traceId`. Enable `rasp.response.include-details` only when revealing the threat category is acceptable. Evidence and submitted payloads are never returned by the error handler.

Other sample surfaces are `/api/file?name=...`, `/api/command?value=...`, `/api/directory?filter=...`, form `POST /api/form` with a `name` field, JSON `POST /api/query`, XML `POST /api/xml`, and text `POST /api/service`. Every endpoint only echoes input as JSON.

## Configuration

Properties are bound and validated under `rasp`. This example shows all settings and their defaults. Place overrides in the consumer application's `application.yml`; `application-rasp.yml` in the autoconfiguration module is a reference, not an automatically activated profile.

```yaml
rasp:
  enabled: true
  mode: BLOCK
  fail-open: true
  block-threshold: 50
  scan:
    headers: true
    query-params: true
    body: true
    uri: true
    method-args: true
    max-body-size: 1MB
    skip-content-types: [multipart/form-data, application/octet-stream]
    max-object-depth: 8
    max-object-values: 2048
  detectors:
    sqli: { enabled: true, priority: 100 }
    xss: { enabled: true, priority: 90 }
    spel: { enabled: true, priority: 95 }
    path-traversal: { enabled: true, priority: 80 }
    command: { enabled: true, priority: 85 }
    ldap: { enabled: true, priority: 70 }
    nosql: { enabled: true, priority: 70 }
    xxe: { enabled: true, priority: 90 }
  whitelist:
    paths: [/actuator/**, /swagger-ui/**, /v3/api-docs/**]
    ips: []
    param-names: []
    headers: [Authorization, Cookie, X-CSRF, X-CSRF-TOKEN, Content-Length]
  response:
    include-details: false
    http-status: 403
  metrics:
    enabled: true
  alerting:
    enabled: false
    threshold: 10
    webhook-url: ""
  cache:
    pattern-cache-size: 500
    pattern-cache-ttl-seconds: 3600
  rate-limit:
    max-entries: 10000
    violation-threshold: 3
    window-seconds: 60
    block-seconds: 60
```

| Property | Default | Meaning |
| --- | --- | --- |
| `enabled` | `true` | Install the servlet protection components |
| `mode` | `BLOCK` | `MONITOR` records findings, `BLOCK` rejects qualifying findings, `STRICT` also tracks repeat offending remote addresses |
| `fail-open` | `true` | Allow on engine/extraction errors; set `false` to reject on errors in BLOCK/STRICT mode; MONITOR always allows |
| `block-threshold` | `50` | Risk score required for blocking, from 1 to 100 |
| `scan.headers` | `true` | Scan non-exempt request header values |
| `scan.query-params` | `true` | Scan query/form names and values |
| `scan.body` | `true` | Inspect eligible request bodies |
| `scan.uri` | `true` | Scan the request URI/path |
| `scan.method-args` | `true` | Enable scanning at supported annotated argument boundaries |
| `scan.max-body-size` | `1MB` | Body scan budget, validated from `1B` through `16MB` |
| `scan.skip-content-types` | multipart and octet-stream | Body media types omitted from byte scanning; this does not disable metadata scanning |
| `scan.max-object-depth` | `8` | Maximum depth for annotated object scanning |
| `scan.max-object-values` | `2048` | Maximum values for annotated object scanning |
| `detectors.<name>.enabled` | `true` | Enable a built-in detector by its configuration name |
| `detectors.<name>.priority` | shown above | Descending ordering priority; priority does not change detector risk scores |
| `whitelist.paths` | shown above | Ant-style path exemptions |
| `whitelist.ips` | empty | Exact servlet remote address exemptions; CIDR is not supported |
| `whitelist.param-names` | empty | Parameter/JSON field name exemptions; use sparingly |
| `whitelist.headers` | shown above | Case-insensitive header name exemptions |
| `response.include-details` | `false` | Include threat category in rejection JSON |
| `response.http-status` | `403` | Rejection status, validated from 400 to 599 |
| `metrics.enabled` | `true` | Register counters when a Micrometer registry exists |
| `alerting.enabled` | `false` | Enable asynchronous metadata-only webhook notifications |
| `alerting.threshold` | `10` | Threat notification threshold per minute |
| `alerting.webhook-url` | empty | Explicit destination, required when webhook alerts are enabled |
| `cache.pattern-cache-size` | `500` | Maximum cached compiled patterns |
| `cache.pattern-cache-ttl-seconds` | `3600` | Pattern cache expiration in seconds |
| `rate-limit.max-entries` | `10000` | Bound on tracked remote addresses in STRICT mode |
| `rate-limit.violation-threshold` | `3` | Violations in the window that trigger a temporary block |
| `rate-limit.window-seconds` | `60` | Violation counting window |
| `rate-limit.block-seconds` | `60` | Temporary block duration |

No password fields are exempt by default. Exemptions are intentional bypasses and apply before detection; a consumer must protect every exempt surface at its eventual sink. Never whitelist a load balancer's address when it is the servlet remote address for every client. The library does not trust `X-Forwarded-For` or `Forwarded` for security decisions. Configure any container-level proxy handling only for explicitly trusted proxies.

Bodies over the configured size are passed intact to the application without body scanning. This is a documented coverage gap even with fail-closed engine handling; use your servlet container or ingress request-size limits if oversized bodies must be rejected. Multipart body bytes and binary data are not inspected; filenames and other supported metadata are separate surfaces. File contents, archive entries, WebSocket messages, and traffic outside Spring MVC are outside the protection boundary.

For unknown-length/chunked form requests, interception defers body access until MVC resolves the handler. The servlet container parses the original stream, preserving Spring's form parameter and form-body behavior; RASP then inspects decoded fields when they fit the scan budget. The scan budget does not constrain the servlet container's own form parser, so configure container/ingress body and parameter limits separately.

## Annotations and customization

HTTP protection works without annotations. Use annotations to inspect values passed through a Spring-managed service proxy:

```java
import dev.rasp.core.RaspProtect;
import dev.rasp.core.RaspScan;
import org.springframework.stereotype.Service;

@Service
public class SearchService {
    @RaspProtect
    public String search(@RaspScan String query) {
        return query; // use a prepared query if accessing a database
    }
}
```

`@RaspScan(types = {ThreatType.SQLI}, deepScan = true)` selects detector categories and enables bounded traversal of nested argument values. Deep reflection is opt-in at annotated boundaries, not applied to every request object. Spring AOP cannot intercept self-invocations or calls to objects created outside Spring; call the proxied bean from another bean. Final/private methods and proxy restrictions also apply.

For explicit output encoding, `dev.rasp.core.SafeOutput.forHtml(value)` encodes text destined for HTML content. The optional `dev.rasp.autoconfigure.RaspEncodeHtml` method annotation encodes an opted-in `String` response only when its negotiated media type is `text/html`. It does not sanitize JSON or replace context-specific JavaScript, CSS, URL, or attribute encoding. Request data is never silently rewritten.

Implement the Spring-free `ThreatDetector` interface and register it as a Spring bean for an additional detector. Use unique, deterministic rule behavior, bounded input processing, and the supplied `ScanContext` locations. The enum intentionally fixes the supported categories. To replace a built-in, use its documented bean name (`raspSqlInjectionDetector`, `raspXssDetector`, `raspSpelInjectionDetector`, `raspPathTraversalDetector`, `raspCommandInjectionDetector`, `raspLdapInjectionDetector`, `raspNoSqlInjectionDetector`, or `raspXxeDetector`) or supply a custom `RaspEngine` bean. A detector must not log submitted input, retain request objects, mutate shared state, or perform slow I/O on a servlet thread.

Policy uses the strongest finding rather than summing duplicate evidence into an exaggerated score. `MONITOR` still publishes findings while allowing the request. `STRICT` tracks the servlet remote address in a bounded in-memory cache; the state is local to one JVM and expires or can be evicted. It is supplementary abuse control, not a distributed request-rate limiter.

## Threat coverage

| Category | Representative signals | Essential application control |
| --- | --- | --- |
| SQLI | UNION SELECT, boolean clauses, time functions, stacked SQL, catalog and error-based probes | Prepared statements and allowlisted identifiers |
| XSS | Script/dangerous tags, event attributes, script URIs, encoded variants, DOM/eval markers | Contextual output encoding and content security policy |
| SPEL | Dangerous type/class access, execution calls, expression markers | Avoid evaluation of untrusted expressions; restrict evaluation context |
| PATH_TRAVERSAL | Relative parent segments, encoded separators, sensitive system paths | Resolve under an allowed root and verify the resolved path |
| COMMAND | Chaining/subshells coupled with command syntax, redirection probes | Avoid shells and use allowlisted executable/argument values |
| LDAP | Filter escape/bypass structures and wildcards in suspicious filter context | RFC-correct LDAP filter/DN escaping |
| NOSQL | Mongo-style dollar operators including nested JSON keys | Explicit query schemas; never merge untrusted objects into queries |
| XXE | XML DTD/entity declarations and external reference syntax in XML media types | Disable DTDs and external entities in every XML parser |

Matching is case-insensitive after bounded normalization. Two decoding passes address common URL/HTML/Unicode encoding variants; detection normalization removes control characters and recognizes known overlong separator encodings without changing the request delivered to the application. Arbitrary transformations and differences between a detector and the eventual parser can still bypass signatures. Core map/array/iterable extraction has depth 32, node-visit count 8192, and value-count 8192 safety budgets; exceeding a core budget is an engine error subject to `fail-open`. A corpus of normal login data, JWT-like strings, base64, braces, names, and common regular expressions is tested; no finite suite proves zero false positives. Start with representative traffic in `MONITOR` mode and tune narrowly.

## Observability and privacy

`rasp.threats.detected` and `rasp.threats.blocked` use fixed threat categories as tags. Request values, URLs, remote addresses, and trace IDs are not metric tags. Boot exposes `/actuator/metrics/...` only if the consumer exposes it; monitoring infrastructure should use authenticated management access. The `rasp` health component reports engine readiness, not a claim that every request is safe.

Threat events and structured SLF4J log fields identify a category, decision, location, and trace. Logs never need raw evidence; application event listeners must apply the same privacy discipline. Do not log complete `ScanContext`, exception argument arrays, request bodies, headers, or records containing evidence. Default Boot/Logback logging works without a logging encoder; configure your organization's JSON logging layout separately if needed.

Optional webhook alerts are disabled by default and must use a destination controlled by the operator. Delivery is asynchronous and bounded, so an unavailable endpoint does not stall request handling. Alert delivery is best effort rather than a durable queue. Email and Kafka integrations are not included; consumers can implement an event listener outside the request path.

## Tests and performance

`mvn clean verify` executes detector positive/negative corpora, engine policy tests, false-positive cases, auto-configuration tests, MockMvc interception tests, and sample acceptance tests. The default sample suite also starts embedded Tomcat on a random port and uses the JDK HTTP client to verify actual servlet query parsing, JSON/form body replay, blocking, JWT header handling, and management endpoints. JaCoCo enforces at least 80% line coverage in the library modules; HTML reports are written to each tested module's `target/site/jacoco/` directory. The default suite is self-contained and requires no Docker daemon.

The optional Testcontainers profile copies the packaged executable jar into a JRE 25 container and exercises the real servlet server over HTTP, including body replay, required blocking cases, a normal request with an Authorization header, and management signals:

```sh
mvn verify -Pdocker-it
```

This explicit profile requires a working Docker daemon and permission to pull `eclipse-temurin:25-jre`. It fails when Docker is unavailable rather than marking the container check as passed. The test does not run during ordinary `mvn verify`. On PowerShell, use `.\scripts\maven.ps1 verify -Pdocker-it`.

The specification's 10,000 requests/second and p99 under 2 ms are **performance targets**, not guarantees. Run the opt-in core characterization on your hardware:

```sh
mvn -pl rasp-core -Drasp.perf=true -Djacoco.skip=true -Dtest=RaspEnginePerfTest test
```

This command disables JaCoCo instrumentation and runs only the characterization test. On PowerShell use `.\scripts\maven.ps1 -pl rasp-core '-Drasp.perf=true' '-Djacoco.skip=true' '-Dtest=RaspEnginePerfTest' test`.

The characterization writes `rasp-core/target/rasp-performance.txt` with measured p99 and scans/second for three scenarios: 10,000 small clean queries on one calling thread, 100 clean 64 KiB queries on one thread, and 40,000 small clean queries split across four workers. Each worker warms up before a common start barrier; the four-worker result reports aggregate throughput and combined per-scan latency. Four workers do not establish performance on four physical cores: the test sets no CPU affinity or container CPU quota.

Review these values together with JVM version, CPU, warmup, request corpus, and concurrency. A core-engine timing excludes servlet body buffering, container scheduling, application work, network time, TLS, event listeners, and request queueing. Use end-to-end load tests against your consumer, including large/adversarial inputs, before setting service latency budgets. The GitHub Actions manual performance job is diagnostic; shared runner numbers do not certify production throughput.

Useful cost controls are body size, depth/value budgets, skipped media types, selective method annotations, and detector enable flags. Disabling a detector or exempting an input reduces coverage. Cached compilation avoids recompiling expressions on each request; it does not make arbitrary regular expressions or unbounded custom detectors safe.

## Container and CI

```sh
docker compose up --build
```

The multi-stage image builds and verifies the project, then runs the demo as UID 10001 on JRE 25. The [official Maven image](https://hub.docker.com/_/maven/) supplies the JDK 25 builder, and the [official Eclipse Temurin image](https://hub.docker.com/_/eclipse-temurin) supplies the runtime. Compose publishes only on `127.0.0.1`, drops Linux capabilities, uses a read-only root filesystem and bounded `/tmp`, and limits memory. Change ingress/authentication and management exposure deliberately for a real deployment. Pin base image digests and scan the resulting image in your release process.

`.github/workflows/ci.yml` runs the Java 25 reactor verification and retains test/coverage evidence. Manually selected jobs run the opt-in performance characterization or Docker acceptance test. The workflow does not publish artifacts, send notifications, or deploy an application.

After normal verification, build source and API documentation artifacts with:

```sh
mvn -Prelease-artifacts -DskipTests package
```

The command packages artifacts locally; it does not publish or sign a release. On PowerShell, pass `'-DskipTests'` to the helper.

Before a release, choose your organization’s artifact coordinates and version, review the support/dependency position, define a private vulnerability-reporting contact, run representative monitoring and load tests, and publish signed artifacts using your release process. This generated project is not accompanied by a third-party penetration test or security certification.

# Security policy

## Reporting a vulnerability

Report security issues privately to the maintainers through your organization's established security channel. When this project is hosted, enable the repository's private vulnerability reporting feature or add a monitored security contact here before inviting external users. This generated repository intentionally contains no invented email address or disclosure URL.

Include the affected version, deployment configuration, a minimal inert reproduction, the expected and observed decision, and whether the problem depends on a proxy or servlet container. Remove customer data, credentials, tokens, and private endpoints. Do not publish working exploit details or sensitive request/response captures in a public issue before coordinating disclosure. No response SLA is asserted until a maintaining organization adopts one.

## Protection boundary

The starter screens servlet HTTP inputs and supported annotated Spring proxy arguments with heuristic patterns. It does not instrument JDBC, process creation, XML parsers, expression evaluators, filesystem operations, browser execution, or other sinks. Inputs modified after scanning, non-HTTP inputs, self-invoked/private/final method calls that bypass Spring AOP, WebSocket traffic, and reactive WebFlux/Netty traffic are not comprehensively covered. Keep secure APIs and application authorization at the actual operation.

The eight detector categories cover recognizable signatures, not all representations or variants of each attack family. Normalization is bounded, and parsing differences, encodings beyond the supported passes, content-type confusion, novel payloads, fragmented data, and application-specific transformations may bypass patterns. False positives remain possible. Regular expression rules must be reviewed for bounded behavior when customized.

## Deliberate exclusions and limits

- `rasp.fail-open=true` is the default: an engine/extraction failure allows the request. Operators who require rejection on processing errors in BLOCK/STRICT mode must explicitly configure `false` and validate availability behavior. MONITOR always allows requests.
- Body scanning stops at the configured size budget. Oversized bodies are replayed intact but are not scanned. This exclusion is independent of engine fail-open/closed policy. Enforce application/ingress size limits separately when required.
- Unknown-length form requests retain the container's form parser before RASP scans decoded fields that fit its budget. The RASP budget does not cap the container's parsing allocation or parameter count; enforce those limits in the container/ingress as well.
- Multipart/binary body bytes are excluded by default. Metadata scanning does not inspect uploaded file contents, archive members, or embedded documents.
- Header, parameter, field, path, and IP exemptions remove detection coverage. The default path exemptions include management/documentation routes; these still require authentication and network controls.
- `Authorization`, `Cookie`, CSRF headers, and `Content-Length` are excluded by default. The library is not an authentication, session, or CSRF implementation.
- IP decisions use the servlet remote address. Untrusted forwarding headers never grant exemptions. Container/proxy remote-address rewriting must trust only known proxies. IP exemptions do not accept CIDR ranges.
- STRICT violation tracking is bounded and local to a JVM; restarts, expiry, and cache eviction discard state. It is not a distributed denial-of-service defense.
- Health `UP` means the component is ready; it does not certify detection completeness or traffic safety. Metrics can disclose security activity if management endpoints are public.
- Custom detectors and Spring event listeners execute with the host application's privileges. Keep request-path work deterministic, bounded, and free of external I/O. Asynchronous webhook alerts are best effort and should not be treated as an audit archive.

## Evidence and privacy

Production logs and rejection responses must not contain submitted payloads, credentials, authorization/cookie values, or complete exception arguments. Raw evidence can exist inside in-process detection results for policy/testing; treat these records and any custom event listeners as sensitive. Do not serialize the entire `ScanContext` or engine results to logs. The built-in observable signals use bounded category labels and trace identifiers rather than payloads or unbounded metric tags.

Keep `rasp.response.include-details=false` for deployed services unless operators intentionally expose detector categories. The sample's health-details and metrics exposure are demonstration settings, not a hardened management configuration.

## Dependency and release status

The project targets Java 25 and Spring Boot 3.5.16. The original Boot 3.2 baseline was updated because [Boot 3.5 supports Java 25](https://docs.spring.io/spring-boot/3.5/system-requirements.html). Spring announced 3.5.16 as the [last open-source release of the 3.5 line](https://spring.io/blog/2026/06/25/spring-boot-3-5-16-available-now/). A passing test suite is not evidence of current dependency support or absence of CVEs. Review the full dependency tree and security advisories and upgrade to a supported baseline before an Internet-facing production release.

The repository has no independent security audit, production certification, maintenance SLA, or published release provenance. A maintaining organization must define its supported versions and patch/disclosure process. Performance measurements are workload/hardware dependent; enforce service limits with representative end-to-end load tests.

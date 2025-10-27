# PodiumD Hosting Standards 2025Q4

| Version | Date | Author | Comments |
|----|----|----|----|----|
| 2024Q3 | 27/8/2024 | Jim Leitch | SSC | Converted to Markdown |
| 2025Q4 | 27/10/2025 | SSC/Dimpact | Updated with SRE/DevOps best practices |

## Executive Summary

The PodiumD hosting environment runs a collection of Common Ground-Compliant resources currently hosted on Microsoft Azure Kubernetes Service (AKS).

Development, Test, Acceptance and Production (OTAP) environments are provided for multiple Dutch city councils (municipalities) to host the PodiumD application, a replacement for the current E-suite workflow system as used by many municipalities.

PodiumD is a collection of Open-Source components built by external software developers funded by the municipalities.

This document formalizes what SSC (the hosting provider) expects from Dimpact (the contracting party) and their contracted software developers in terms of packaging, deployment, operations, and integration into the SSC hosting environment.

Rather than **pre**scribing how developers should deliver their software, it describes mutual expectations between SSC Hosting and developers. It will develop over time with input from all parties through Git Pull Requests.

## Document Conventions

We use RFC-style terms to describe requirement levels:
- "**MUST**" / "**MUST NOT**" - Absolute requirements
- "**SHOULD**" / "**SHOULD NOT**" - Strong recommendations
- "**MAY**" - Optional capabilities

Reference: [RFC 2119: Key words for use in RFCs to Indicate Requirement Levels](https://www.rfc-editor.org/rfc/rfc2119)

## Multiple Environments Require Easy Deployments

Applications will be deployed for multiple municipalities across multiple OTAP environments. Therefore, **100% automated deployment is a hard requirement** for any applications being deployed.

## Infrastructure Description

### Subscriptions / Environments

Each customer/gemeente utilizes a set of Azure Subscriptions - one subscription for each OTAP environment. This simplifies billing and provides security demarcation between environments at customer, supplier and OTAP levels.

Not every customer/supplier requires the full set of OTAP environments.

### Cloud Resource Provider

We currently use **Microsoft Azure** as our cloud platform. However, we **MAY** migrate to alternative cloud platforms in the future. Any deployment efforts should use **open standards** and be as cloud-agnostic as possible to facilitate potential migration to alternative or multi-cloud platforms.

### Azure Resources

Current Azure resources in use:

- **Kubernetes (k8s)** – Azure Kubernetes Service (AKS)
  - Standard AKS with two autoscaling node groups: one for application workloads, another for management workloads
  - Regular monthly update cycle across environments with 1-week intervals

- **Container Registry** – Azure Container Registry (ACR)
  - ACR with integrated vulnerability scanning
  - All containers are scanned before deployment

- **SQL DBMS** - Azure Database for PostgreSQL flexible servers
  - PostgreSQL 14+ is our standard database server
  - One instance per environment with multiple databases (one per application)
  - Database timezone is **UTC**

- **Edge Load Balancer** - Azure Application Gateway
  - Internet-facing endpoint providing SSL termination, HTTP → HTTPS redirection, URL rewriting
  - Traffic distribution to appropriate K8S clusters
  - Separate load balancers for DEV/TEST and ACC/PROD
  - HTTP/2 support is currently DISABLED

- **Storage** - Azure Storage Accounts
  - Blob storage and Azure File Shares (SMB and NFS)
  - Object storage preferred for files

- **Secret and Certificate Management** – Azure Key Vault
  - Single source of truth for all certificates and secrets
  - Integrated with Kubernetes secret management

- **Identity Management** - Active Directory / Entra ID
  - Azure Entra ID **SHOULD** be the only method of Identity Management
  - Alternative identity tools require waiver with migration plan to Entra

- **Deployment Orchestration** - Azure DevOps
  - CICD orchestration tool for infrastructure and application deployment
  - Uses Terraform and bash scripting with az cli commands

### Infrastructure Lifecycle

- Most Azure infrastructure resources (storage, databases) have very long lifecycles
- Kubernetes versions are updated monthly via Azure Fleet Manager: **ontw → test → accp → prod** with one week between each environment
- Non-critical k8s patches run in ONTW for 3 weeks before reaching production
- Upgrades to RDBMS and storage will be announced well in advance for testing and upgrade planning

### SSL Certificates

- **LetsEncrypt** certificates are used for all web-facing services
- Certificates are automatically requested and downloaded by cert-manager
- Certificates/keys stored in Azure Key Vault, then deployed to Application Gateway listeners

### Environments (OTAP)

#### DEVELOPMENT (ONTWIKKEL)

Development environments verify proper working in production-similar environments. Developers can deploy container versions and helm charts. Azure resource and k8s cluster changes are performed by SSC hosting.

**Cost savings**: Dev, test and acceptance environments are switched off evenings and weekends. Individual environments can be kept on for short periods if needed.

#### Test

Test environments are for municipalities to test new features and provide staff training.

#### Acceptance

Acceptance environments are **100% like-for-like with production**: same hardware resources and copies of production data.

#### Production

Production environments have maximum uptime, monitoring, and resources.

## What We Expect from Suppliers

We expect Dimpact to require suppliers to deliver applications enabling secure, scalable, stable, manageable, and easily deployable systems.

### Service Naming Conventions [COMP001-naming]

**Consistency in naming is critical for integration and operations.**

#### Service Names

All services **MUST** follow a consistent naming pattern:

- Use lowercase with hyphens for separation (kebab-case): `open-zaak`, `open-forms`, `object-api`
- Service names **MUST** be descriptive and self-documenting
- Avoid abbreviations unless widely recognized in the domain
- Service names **SHOULD** align with their functional purpose
- Maximum length: 63 characters (Kubernetes DNS label limit)

**Examples of proper naming:**
```
✓ open-zaak
✓ notification-service
✓ document-api
✓ form-builder
✗ oz (too abbreviated)
✗ OpenZaak (incorrect case)
✗ open_zaak (underscores not allowed)
```

#### Resource Naming

All Kubernetes resources (deployments, services, configmaps, secrets) **MUST** follow the pattern:
```
{service-name}-{resource-type}
```

**Examples:**
- `open-zaak-deployment`
- `open-zaak-service`
- `open-zaak-config`
- `notification-service-secret`

#### Environment-Specific Naming

When environment-specific resources are needed, use:
```
{service-name}-{environment}
```

**Examples:**
- `open-zaak-prod`
- `form-builder-test`

#### Database Naming

Database names **MUST** match the service name:
- Service: `open-zaak` → Database: `openzaak` (underscores allowed in PostgreSQL)

#### Helm Chart Naming

Helm charts **MUST** be named identically to the service name:
- Service: `open-zaak` → Chart name: `open-zaak`

### Versioning [COMP002-versioning]

PodiumD consists of a suite of sub-applications. The PodiumD version is defined by the versions of all sub-applications.

- [COMP002.1-semantic-versioning] All components **MUST** use **semantic versioning** (MAJOR.MINOR.PATCH)
- Pre-release versions **MAY** use keywords (e.g., `1.2.3-rc1`, `1.2.3-beta`)
- [COMP002.2-version-api] Each component **MUST** expose a version API endpoint at `/api/v1/version` or `/health/version` that returns:
  ```json
  {
    "version": "1.2.3",
    "commit": "abc123def",
    "buildDate": "2025-10-27T12:00:00Z",
    "component": "open-zaak"
  }
  ```
- Version information **MUST** be accessible without authentication
- Version endpoint **SHOULD** respond within 100ms

### Containers

- [COMP003.1-app-as-containers] Applications **MUST** be delivered as one or a set of application containers

- [COMP003.2-container-registry] Containers **MUST** be available via a public Container Registry (e.g., **Docker Hub**, **GitHub Container Registry**)
  - Non-open-source containers **MAY** be secured by key/token
  - SSC hosting will copy containers to ACR in the SSC Azure environment

- [COMP003.3-semantic-versioning] Container labels **MUST** use semantic versioning

- [COMP003.4-immutable-tags] Container images **MUST** use immutable tags (version numbers, not `latest`)
  - Development builds **MAY** use `latest` or branch names
  - Production deployments **MUST** reference specific versions

- [COMP003.5-minimal-images] Container images **SHOULD** be minimal (no unnecessary shells/tools)
  - Use distroless or alpine base images where possible
  - Multi-stage builds to reduce attack surface

- [COMP003.6-non-root] Containers **MUST NOT** run as root unless explicitly justified and approved

### Timezone [COMP004-timezone]

For logging and monitoring purposes, timezone **MUST** be **UTC**. Default database timezone is UTC. All logging and metrics **MUST** use UTC output. End-user interfaces **SHOULD** present data in appropriate local timezone.

### Software Supply Chain Security [COMP005-software-supply-chain]

SSC Hosting scans all containers for vulnerabilities regularly.

- [COMP005.1-scanning] Containers **MUST** be scanned for vulnerabilities by the supplier before delivery
- [COMP005.2-sbom] Each release **MUST** include a Software Bill of Materials (SBOM) in CycloneDX or SPDX format
- [COMP005.3-vulnerability-remediation] Vulnerabilities flagged **MUST** be fixed, mitigated, or receive waiver from system architects (supplier + Dimpact + SSC Hosting) within:
  - Critical: 7 days
  - High: 30 days
  - Medium: 90 days
- [COMP005.4-dependency-updates] Dependencies **SHOULD** be kept current; deprecated APIs removed proactively
- [COMP005.5-signed-images] Container images **SHOULD** be signed using cosign or similar

**Recommended scanning tools:**
- Code-level: SonarQube, Snyk
- Container scanning: Trivy, GitHub container scanning, Grype

### Scalability and Startup/Shutdown

- [COMP006.1-horizontal-scaling] Applications **SHOULD** support horizontal scaling (stateless or externalized state)

- [COMP006.2-stop-start-safe] Applications **MUST** withstand sudden stops and starts (node failures, container migrations)

- [COMP006.3-graceful-shutdown] Applications **MUST** implement graceful shutdown:
  - Handle SIGTERM signal
  - Complete in-flight requests (max 30 seconds)
  - Close connections cleanly
  - Deregister from service discovery

- [COMP006.4-fast-startup] Applications **SHOULD** start within 60 seconds
  - Use startup probes for slow-starting applications
  - Defer non-critical initialization

- Applications **MAY** use init containers, startup and shutdown scripts

### Upgrade Time Limits [COMP006.5-upgrade-duration]

**Component upgrades are critical to operational efficiency and must be fast.**

- [COMP006.5-upgrade-time-limit] Any component upgrade **MUST** complete within **15 minutes** for standard deployments
  - This includes database migrations, rolling updates, and validation
  - Downtime during upgrade **SHOULD** be zero (rolling updates)
  - If zero-downtime is not possible, downtime **MUST NOT** exceed 5 minutes

- [COMP006.6-rollback-capability] All upgrades **MUST** support rollback within **5 minutes**
  - Database migrations **MUST** be backwards-compatible or include down-migrations
  - State changes **MUST** be reversible

- [COMP006.7-upgrade-testing] Upgrade procedures **MUST** be tested in lower environments before production
  - Document actual upgrade times in release notes
  - If upgrade exceeds 15 minutes, maintenance window must be pre-approved

### Helm Charts & Deployment

- [COMP007.1-helm] All applications **MUST** be accompanied by Helm Charts
  - Applications **MUST** be deployable as sub-charts in an umbrella chart
  - Charts **MUST** follow consistent variable naming conventions (provided by Dimpact)

- [COMP007.2-operators] Kubernetes Operators are **NOT** supported **unless** they can be encapsulated inside a single Umbrella Helm chart

- [COMP007.3-dependency-apps] Dependency applications **MUST** be deployed as Helm sub-charts

- [COMP007.4-rolling-updates] Deployments **SHOULD** use rolling updates
  - Zero-downtime deployments for stateless applications
  - Database schema changes requiring downtime need waiver

- [COMP007.5-storage] Helm chart **MUST** support SSC's storage choices (if file storage required)
  - Parameterized storage class
  - Support for Azure Files, Azure Blob, or NFS

- [COMP007.6-ingress] Helm chart **MUST** support SSC's ingress choices (if ingress required)
  - Configurable ingress class
  - Support for path-based and host-based routing

- [COMP007.7-database] Helm chart **MUST** support external cloud database usage (if database required)
  - Parameterized connection strings
  - No embedded databases in production

- [COMP007.8-helm-validation] Helm charts **MUST** pass validation:
  - `helm lint` with no errors
  - `helm template` successful for all environments
  - Schema validation for values

### K8s Namespaces

PodiumD as a whole is deployed in one single **"podiumd"** kubernetes namespace.

All PodiumD components share this namespace for simplified service discovery and communication.

### 100% Automation [COMP008-automation]

PodiumD software deploys to many customers and environments.

- [COMP008.1-100percent-automated] All components **MUST** be deployable completely from Helm chart
  - No manual configuration steps
  - No manual database setup
  - No manual secret creation (automated via Azure Key Vault integration)

**100% deployment means**: Containers running correctly in K8s cluster with correct parameters, secrets, connecting to databases and other PodiumD applications.

Final functional configuration "inside" the application is out of scope.

**Deployment phases** (as defined by Maykin Media):
- **OTAP-omgevingen**: Ontwikkel-, Test-, Acceptatie, Productie-omgevingen
- **Platform**: Set of components providing integrated solution
- **Installatie**: Deploying component (Helm chart execution for technical operation)
- **Configuratie**: Parameters for component operation (e.g., inter-component connections)
- **Inrichting**: Functional content and specific configuration (e.g., case types, processes)

### Application Addressing

- Dimpact provides application URLs for all public-facing applications
- Dimpact liaises with gemeentes for application addressing, hostnames, and CNAMEs

### Communication Between Components

- [COMP009.1-internal-communication] Communication between components inside the same k8s cluster **SHOULD** use cluster-internal DNS
  - Use Kubernetes service names: `http://service-name.podiumd.svc.cluster.local`
  - Avoid external URLs for internal communication

- [COMP009.2-api-communication] Communication and data transfer **SHOULD** use APIs (RESTful or async messaging)
  - No direct database access between components
  - Use standard protocols (HTTP/REST, gRPC, or message queues)

- [COMP009.3-async-preferred] Asynchronous communication **SHOULD** be preferred where latency tolerance exists
  - Events, message queues for decoupling
  - Synchronous calls **MUST** include timeouts, retries, circuit breaking

### Redundancy and Fault Tolerance [COMP009A-resilience]

**Applications MUST implement redundancy and fault tolerance mechanisms to meet reliability requirements.**

Based on the Azure Well-Architected Framework, applications **MUST** be designed to handle failures gracefully and maintain availability.

- [COMP009A.1-multi-zone] **Multi-Zone Deployments**: Applications **SHOULD** support deployment across Azure Availability Zones
  - Pod anti-affinity rules to spread replicas across zones
  - Topology spread constraints for even distribution
  - Zone-aware persistent storage when needed
  - Graceful handling of zone failures

- [COMP009A.2-retry-mechanisms] **Retry Logic**: Applications **MUST** implement retry mechanisms for transient failures
  - Exponential backoff for retries (e.g., 1s, 2s, 4s, 8s)
  - Maximum retry attempts (typically 3-5)
  - Jitter to prevent thundering herd
  - Retry only idempotent operations or use idempotency keys
  - Log retry attempts for debugging
  
- [COMP009A.3-circuit-breaker] **Circuit Breaker Pattern**: Applications **SHOULD** implement circuit breakers for external dependencies
  - Prevent cascading failures from slow/failing dependencies
  - Three states: Closed (normal), Open (failing), Half-Open (testing recovery)
  - Configurable failure threshold (e.g., 50% errors over 10 requests)
  - Configurable timeout before attempting recovery (e.g., 30-60 seconds)
  - Fallback mechanisms when circuit is open (cached data, degraded functionality)
  - Examples: Use libraries like Polly (.NET), Resilience4j (Java), PyBreaker (Python)

- [COMP009A.4-load-balancing] **Load Balancing**: Applications **MUST** support load balancing across multiple instances
  - Minimum 2 replicas for production (3+ recommended)
  - Kubernetes Service load balancing for internal traffic
  - Azure Application Gateway for external traffic
  - Session affinity **SHOULD** be avoided (use stateless design)
  - Health-based routing (only healthy pods receive traffic)

- [COMP009A.5-timeouts] **Timeouts**: All external calls **MUST** have appropriate timeouts
  - Connection timeout (typically 5-10 seconds)
  - Request timeout (typically 30-60 seconds, depending on operation)
  - Database query timeout (typically 10-30 seconds)
  - Never use infinite timeouts
  - Document expected timeout values

- [COMP009A.6-bulkhead] **Bulkhead Pattern**: Applications **SHOULD** isolate critical resources
  - Separate thread pools for different operations
  - Resource limits per operation type
  - Prevent resource exhaustion from affecting entire application
  - Example: Separate pools for user requests vs. background jobs

- [COMP009A.7-rate-limiting] **Rate Limiting**: Applications **SHOULD** implement rate limiting
  - Protect against overload and abuse
  - Per-client rate limits for public APIs
  - Backpressure mechanisms for internal queues
  - Return HTTP 429 (Too Many Requests) with Retry-After header

Reference: [Azure Well-Architected Framework - Reliability](https://learn.microsoft.com/en-us/azure/well-architected/reliability/)

### Database Migrations

- [COMP010.1-sql-migration-scripts] Applications **MUST** have database migrations built-in or use migration tools (Flyway, Liquibase, Alembic)

- [COMP010.2-migration-automation] Migrations **MUST** be:
  - Versioned and idempotent
  - Automated in deployment pipeline before application rollout
  - Backwards-compatible or include down-migrations

- [COMP010.3-migration-testing] Migrations **MUST** be tested with production-like data volumes

- [COMP010.4-backup-verification] Database backups **MUST** be verified before migration; rollback strategy documented

### Application Concurrency / Statelessness

- [COMP011.1-stateless] Applications **SHOULD** be stateless and commit all data to database

- [COMP011.2-object-storage] Files **SHOULD** be written to object store (Azure Blob Storage)

- [COMP011.3-file-shares] Files **MAY** be written to cloud file shares

- [COMP011.4-no-local-storage] Persistent files **MUST NOT** be written to local container storage but to Kubernetes PV

- [COMP011.5-permanent-data] Permanent data requiring backup **MUST** be written to file share created outside K8s cluster

- [COMP011.6-session-externalization] Session state **MUST** be externalized (Redis, database, or distributed cache)

**File storage preference order:**
1. Object Storage (Azure Blob)
2. SMB file share
3. Persistent volume storage

### Configuration / Secrets

- [COMP012.1-env-vars] All application configuration **SHOULD** use environment variables

- [COMP012.2-configmaps] Environment variables **SHOULD** be set via ConfigMaps in Helm chart

- [COMP012.3-secrets] Secret/sensitive information **MUST** use K8s secrets store
  - Secrets sourced from Azure Key Vault at deployment time
  - No secrets in code, container images, or version control

- [COMP012.4-secret-rotation] Applications **SHOULD** support secret rotation without restart

- [COMP012.5-log-verbosity] Logging verbosity **SHOULD** be controlled by environment variable (e.g., `LOG_LEVEL=DEBUG`)

### SSL Certificates [COMP013-ssl]

- [COMP013.1-http-endpoints] Application endpoints **MUST** be presented as HTTP (unencrypted)
  - Azure Application Gateway performs SSL termination
  - Secure path to k8s cluster via Azure VWAN and Virtual Hub
  - LetsEncrypt SSL certificates populated at deploy-time

- [COMP013.2-internal-tls] Internal service-to-service communication **MAY** use mutual TLS if required for compliance

### System State Definitions [COMP014-system-states]

**Applications MUST define and document clear operational states to enable proper monitoring, automated responses, and incident management.**

Based on the Azure Well-Architected Framework Reliability pillar, all applications **MUST** support the following operational states:

- [COMP014.1-state-healthy] **Healthy/Normal**: System operates within expected parameters
  - All dependencies available
  - Performance metrics within acceptable ranges
  - No degraded functionality
  - Ready to serve production traffic

- [COMP014.2-state-degraded] **Degraded**: Reduced functionality but still operational
  - Some non-critical features unavailable
  - Performance reduced but acceptable
  - Service continues with limitations
  - May operate with cached data or fallback mechanisms

- [COMP014.3-state-failed] **Failed/Unhealthy**: System not functioning correctly
  - Critical dependencies unavailable
  - Cannot serve requests properly
  - Requires intervention or restart
  - Should not receive production traffic

- [COMP014.4-state-maintenance] **Maintenance**: Planned downtime
  - Scheduled updates or repairs
  - Graceful shutdown of services
  - Clear communication to users
  - Documented maintenance windows

- [COMP014.5-state-recovery] **Recovery**: System returning to normal operation
  - Post-incident recovery phase
  - Dependencies reconnecting
  - Caches rebuilding
  - Gradual return to full capacity

**State Transition Requirements:**

- [COMP014.6-state-transitions] Applications **MUST**:
  - Document valid state transitions (e.g., Healthy → Degraded → Failed)
  - Emit events/logs when state changes occur
  - Expose current state via health endpoints
  - Support automated decision-making based on state

- [COMP014.7-state-visibility] State information **MUST** be:
  - Observable via health check endpoints
  - Logged with structured data
  - Exposed as metrics (e.g., `app_state{state="healthy"}`)
  - Included in monitoring dashboards

Reference: [Azure Well-Architected Framework - Reliability](https://learn.microsoft.com/en-us/azure/well-architected/reliability/)

### Health Probes [COMP014A-healthchecks]

Applications **MUST** provide probe endpoints for Kubernetes health management that accurately reflect the system states defined above.

- [COMP014A.1-liveness] **Liveness probe**: Indicates if application is running (maps to Healthy vs Failed states)
  - Should check critical internal components only
  - **MUST NOT** fail due to external dependency issues
  - Failure triggers container restart
  - Recommended endpoint: `/health/live` or `/healthz`
  - Example checks: internal process health, deadlock detection, memory leaks

- [COMP014A.2-readiness] **Readiness probe**: Indicates if application can serve traffic (maps to Healthy vs Degraded states)
  - **MUST** check all critical dependencies (database, cache, downstream services)
  - Failure removes pod from load balancer rotation
  - Recommended endpoint: `/health/ready` or `/readyz`
  - Example checks: database connectivity, external API availability, queue connectivity

- [COMP014A.3-startup] **Startup probe**: For slow-starting applications
  - Disables liveness/readiness checks during startup
  - Higher failure threshold and longer interval
  - Recommended endpoint: `/health/startup` or same as liveness
  - **MUST** be configured for applications taking >30 seconds to start

- [COMP014A.4-probe-performance] Probes **MUST**:
  - Respond within 1 second (timeout)
  - Not have business logic side-effects
  - Be lightweight (no heavy database queries or complex computations)
  - Return appropriate HTTP status codes (200 = healthy, 503 = unhealthy)

- [COMP014A.5-probe-configuration] Probe configuration **MUST** include:
  - `initialDelaySeconds`: Time before first probe (typically 10-30s)
  - `periodSeconds`: How often to probe (typically 10s for liveness, 5s for readiness)
  - `timeoutSeconds`: Probe timeout (typically 1-3s)
  - `failureThreshold`: Consecutive failures before action (typically 3)
  - `successThreshold`: Consecutive successes to be considered healthy (typically 1)

- [COMP014A.6-probe-response] Health probe endpoints **MUST** return:
  - HTTP 200-299: Healthy/Ready
  - HTTP 503: Service Unavailable (degraded or failed)
  - HTTP 500-599: Server error (failed)
  - Response body **SHOULD** include state details in JSON format:
    ```json
    {
      "status": "healthy",
      "timestamp": "2025-10-27T12:00:00Z",
      "checks": {
        "database": "up",
        "cache": "up",
        "downstream_api": "degraded"
      }
    }
    ```

Reference: [Configure Liveness, Readiness and Startup Probes | Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

## Logging / Monitoring / Metrics / Alerting

### Logging

- [COMP015.1-logging-to-stdout] Applications **MUST** stream all logs to STDOUT
  - Logs captured and sent to centralized logging
  - No file-based logging in containers

- [COMP015.2-structured-logging] Logs **SHOULD** be in JSON format for parsing
  - Include correlation IDs for request tracing
  - Structure enables automated analysis

- [COMP015.3-log-levels] Logging **MUST** be configurable via `LOG_LEVEL` environment variable
  - Standard levels: DEBUG, INFO, WARN, ERROR, FATAL
  - Default production level: INFO

- [COMP015.4-utc-logs] Log timestamps **MUST** be in UTC format (ISO 8601)

- [COMP015.5-no-sensitive-data] Logs **MUST NOT** contain sensitive data (passwords, tokens, PII)
  - Mask or redact sensitive fields
  - Follow data privacy regulations

- [COMP015.6-correlation-ids] Logs **SHOULD** include correlation IDs for distributed tracing

Example JSON log format:
```json
{
  "timestamp": "2025-10-27T12:00:00Z",
  "level": "INFO",
  "service": "open-zaak",
  "correlationId": "abc-123-def",
  "message": "Request processed successfully",
  "duration": 150,
  "statusCode": 200
}
```

Reference: [Better Stack JSON Logging Guide](https://betterstack.com/community/guides/logging/json-logging/)

### Metrics [COMP016-metrics]

Applications **SHOULD** expose metrics for monitoring and alerting.

- [COMP016.1-prometheus-format] Metrics **MUST** be in Prometheus format at `/metrics` endpoint

- [COMP016.2-golden-signals] Applications **SHOULD** expose the "Four Golden Signals":
  1. **Latency**: Request duration/response time
  2. **Traffic**: Request rate (requests per second)
  3. **Errors**: Error rate (failed requests)
  4. **Saturation**: Resource utilization (CPU, memory, connections)

- [COMP016.3-business-metrics] Applications **MAY** expose business-specific metrics:
  - Number of transactions
  - Transaction latency
  - Incomplete transactions
  - Failed transactions
  - Queue depths
  - Custom domain metrics

- [COMP016.4-version-metric] Applications **SHOULD** expose version as metric:
  ```
  app_info{version="1.2.3", component="open-zaak"} 1
  ```

**Standard Prometheus metrics to include:**
- `http_requests_total` - Total HTTP requests (counter)
- `http_request_duration_seconds` - HTTP request latency (histogram)
- `http_requests_in_flight` - Current in-flight requests (gauge)
- `process_cpu_seconds_total` - Process CPU usage
- `process_resident_memory_bytes` - Process memory usage

Metrics enable:
- Graphing and analysis
- Automated alerting
- Capacity planning
- Performance optimization

Reference: [Golden Signals for Kubernetes - Sysdig](https://sysdig.com/blog/golden-signals-kubernetes/)

### Alerting [COMP016A-alerting]

**Applications and infrastructure MUST have comprehensive alerting configured to enable proactive problem detection and rapid incident response.**

SSC Hosting configures centralized alerting based on metrics and health states. Applications **MUST** expose appropriate metrics to enable effective alerting.

#### Critical Threshold Alerts

- [COMP016A.1-resource-alerts] **Resource Usage Alerts**: Monitor and alert on resource exhaustion
  - **CPU Usage**: Alert when >80% sustained for 5+ minutes
  - **Memory Usage**: Alert when >85% sustained for 5+ minutes
  - **Disk Space**: Alert when >80% used (persistent volumes)
  - **Network Saturation**: Alert on packet loss or bandwidth limits
  
- [COMP016A.2-response-time-alerts] **Response Time / Latency Alerts**: Monitor request performance
  - **P95 Latency**: Alert when >1000ms for critical endpoints
  - **P99 Latency**: Alert when >2000ms for critical endpoints
  - **Request Duration**: Alert on sudden increases (>2x baseline)
  - **Slow Queries**: Alert on database queries >5 seconds
  
- [COMP016A.3-error-rate-alerts] **Error Rate Alerts**: Monitor application errors
  - **HTTP 5xx Errors**: Alert when >5% of requests fail (over 5 minutes)
  - **HTTP 4xx Errors**: Alert when >20% of requests (potential attack or misconfiguration)
  - **Failed Requests**: Alert on sustained error rates
  - **Exception Rate**: Alert on uncaught exceptions or critical errors
  
- [COMP016A.4-availability-alerts] **Availability / Uptime Alerts**: Monitor service availability
  - **Pod Availability**: Alert when <minimum required replicas running
  - **Service Degradation**: Alert on readiness probe failures
  - **Dependency Failures**: Alert when external dependencies unavailable
  - **SLA Violations**: Alert when availability drops below SLA threshold (typically 99.9%)

#### Alert Configuration Requirements

- [COMP016A.5-alert-severity] Alerts **MUST** be categorized by severity:
  - **Critical (P1)**: Immediate action required - service down or major impact
    - Page on-call engineer immediately
    - Example: All pods failing, database unreachable, >50% error rate
  - **High (P2)**: Urgent attention needed - degraded service
    - Notify team immediately
    - Example: Single pod failing, elevated error rates, high latency
  - **Medium (P3)**: Important but not urgent - potential issues
    - Notify during business hours
    - Example: Resource usage trending high, intermittent errors
  - **Low (P4)**: Informational - no immediate action
    - Create ticket for investigation
    - Example: Approaching resource limits, minor configuration drift

- [COMP016A.6-alert-routing] Alert routing **MUST** include:
  - Clear escalation paths
  - Integration with incident management system
  - Runbook links for common issues
  - Automatic ticket creation for non-critical alerts

- [COMP016A.7-alert-quality] To prevent alert fatigue:
  - Alerts **MUST** be actionable (clear remediation steps)
  - Alerts **SHOULD NOT** fire for expected events (e.g., planned maintenance)
  - Alert thresholds **MUST** be tuned to reduce false positives
  - Alerts **SHOULD** auto-resolve when condition clears
  - Alert frequency **SHOULD** be limited (group related alerts)

- [COMP016A.8-monitoring-sla] Monitoring and alerting systems **MUST**:
  - Have their own high availability configuration
  - Alert delivery time <1 minute for critical alerts
  - Maintain alert history for trend analysis
  - Support alert testing and validation

**Example Alert Configuration:**
```yaml
alerts:
  - name: HighErrorRate
    severity: critical
    condition: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
    for: 5m
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value }}% over last 5 minutes"
      runbook: "https://wiki.example.com/runbooks/high-error-rate"
  
  - name: HighMemoryUsage
    severity: high
    condition: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.85
    for: 10m
    annotations:
      summary: "High memory usage detected"
      description: "Memory usage at {{ $value }}%"
      runbook: "https://wiki.example.com/runbooks/memory-issues"
```

Reference: [Google SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

### Resource Requirements [COMP017-resource-recommendations]

- [COMP017.1-resource-specs] Helm charts **MUST** include CPU/memory requests and limits

- [COMP017.2-baseline-specs] Applications **MUST** provide baseline resource specifications:
  
  **Example specification:**
  ```yaml
  # For application X to process Y transactions per minute:
  resources:
    requests:
      cpu: 500m        # 0.5 CPU cores
      memory: 512Mi    # 512 MB RAM
    limits:
      cpu: 2000m       # 2 CPU cores
      memory: 1Gi      # 1 GB RAM
  
  replicas: 3
  storageClass: azure-file-premium
  storageSize: 10Gi
  ```

- [COMP017.3-autoscaling] Applications **SHOULD** support Horizontal Pod Autoscaling (HPA)
  - Define min/max replicas
  - CPU/memory thresholds for scaling
  - Custom metrics for scaling (if applicable)

- [COMP017.4-capacity-planning] Applications **MUST** provide capacity planning guidance:
  - Baseline performance characteristics
  - Scaling thresholds
  - Resource consumption patterns
  - Database size estimates

### DNS Naming & Multi-Domain Support [COMP018-dns-flexibility]

Applications **MUST** support flexible DNS addressing without code or database changes.

- [COMP018.1-any-url] Applications **MUST** support access from **ANY** URL simultaneously
  - No hardcoded URLs in database
  - No restriction on changing application URLs
  - Gemeentes can change citizen-facing URLs anytime

- [COMP018.2-multiple-domains] Applications **MUST** be accessible via:
  - **Technical domains**: `app.test.gemeente.dimpact.nl`
  - **Vanity domains**: `burgerformulieren.gemeente.nl`
  - **Kubernetes internal DNS**: `openzaak.podiumd.svc.cluster.local`
  - **Relative path domains**: `gemeente.nl/formulieren`

- [COMP018.3-url-configuration] URL configuration **MUST** be:
  - Environment-variable driven
  - Not stored in database
  - Changeable without application restart (if possible)

Reference: [Toegang API's via verschillende domeinnamen - Dimpact Confluence](https://dimpact.atlassian.net/wiki/spaces/PCP/pages/175865864/)

### Release Notes [COMP019-release-notes]

Release notes **MUST** include:

- [COMP019.1-parameters] Full description of Helm deploy parameters
  - New parameters
  - Changed parameters
  - Deprecated parameters

- [COMP019.2-outputs] Full description of output parameters (e.g., API keys generated at deploy)

- [COMP019.3-dependencies] Dependencies on earlier application versions
  - Breaking changes
  - Minimum required versions of dependencies

- [COMP019.4-storage] Storage requirements (type, size, IOPS)

- [COMP019.5-database] Database requirements (version, extensions, schema changes)

- [COMP019.6-ingress] External access / ingress requirements

- [COMP019.7-changelog] Changes following [Keep a Changelog](https://keepachangelog.com/) format:
  - New features
  - Improvements
  - Bug fixes
  - Security fixes
  - Breaking changes
  - Deprecations

- [COMP019.8-upgrade-impact] Upgrade impact:
  - Expected upgrade duration
  - Downtime required (if any)
  - Rollback procedure
  - Testing recommendations

- [COMP019.9-security] Security updates and vulnerability fixes
  - CVE numbers
  - Severity levels
  - Mitigation steps

### Communication with External Systems

- [COMP020.1-third-party-services] Communication with third-party services **MUST**:
  - Use only vetted and approved endpoints
  - Handle failures gracefully
  - Avoid leaking sensitive data in logs
  - Include timeouts and retries

- [COMP020.2-cross-environment] Cross-environment data flows **MUST**:
  - Have prior security and privacy assessment
  - Use explicit whitelisting
  - Use encrypted transport (TLS 1.2+)

- [COMP020.3-email-service] Email services **MUST**:
  - Use approved centralized mail relay
  - Use authenticated API or SMTP over TLS
  - No direct unmanaged outbound mail from containers

### Data Security and Privacy

- [COMP021.1-data-in-transit] Data in transit **MUST** use TLS 1.2+

- [COMP021.2-data-at-rest] Data at rest **SHOULD** use encrypted volumes or managed storage encryption

- [COMP021.3-pii-handling] Personally Identifiable Information (PII) **MUST**:
  - Be encrypted if required by policy
  - Follow GDPR and data retention policies
  - Be masked in logs and metrics

- [COMP021.4-least-privilege] Applications **MUST** use least-privilege access
  - Service accounts with minimal permissions
  - No cluster-admin access
  - RBAC enforced

### Interface Documentation [COMP022-ibom]

- [COMP022.1-api-documentation] Applications **MUST** provide API documentation:
  - OpenAPI/Swagger specification
  - Authentication requirements
  - Rate limits
  - Example requests/responses

- [COMP022.2-ibom] Applications **MUST** provide Interface Bill of Materials (IBOM):
  - List all inbound/outbound APIs
  - Events published/consumed
  - Message queues used
  - External service dependencies

### Backward Compatibility [COMP023-compatibility]

- [COMP023.1-api-compatibility] API changes **MUST** maintain backward compatibility
  - Version APIs appropriately (e.g., `/api/v1/`, `/api/v2/`)
  - Deprecation notices before removal
  - Support N-1 version during transition

- [COMP023.2-data-compatibility] Data format changes **MUST** be backward compatible
  - Handle old and new formats during transition
  - Migration paths documented

## Deployment Process

### Deployment Request (Deployment Verzoek)

Deployment requests use a "Deployment Verzoek" form with all relevant deployment data.

### GitOps Workflow

- Git is the single source of truth
- Changes via Pull Requests (branch protection, mandatory reviews, automated checks)
- Merge to main triggers GitOps operator
- Drift detection alerts when live state diverges
- Remediation via commits, not manual `kubectl` (except emergencies with post-factum commit)

### Access Control

- Service providers submit changes via Pull Requests
- Direct cluster access restricted
- Read/list within assigned namespace(s)
- Create/update only for owned resources (least-privilege RBAC)
- No cluster-admin access

## Quality and Maintainability

- [COMP024.1-code-quality] Code **MUST** be:
  - Version controlled
  - Peer reviewed
  - Tested (unit, integration)
  - Free of critical/high vulnerabilities at release

- [COMP024.2-testing] Applications **SHOULD** include:
  - Unit tests (>80% coverage for critical paths)
  - Integration tests
  - End-to-end smoke tests
  - Performance tests for critical workflows

- [COMP024.3-documentation] Applications **MUST** provide:
  - Architecture documentation
  - Deployment documentation
  - Operational runbooks
  - Troubleshooting guides

## Disaster Recovery and Resilience Testing [COMP025-disaster-recovery]

**Regular disaster recovery exercises and resilience testing are critical to ensure systems can recover from failures and meet reliability objectives.**

Based on the Azure Well-Architected Framework Reliability pillar, organizations **MUST** plan, test, and document disaster recovery capabilities.

### Disaster Recovery Planning

- [COMP025.1-recovery-objectives] **Recovery Objectives**: Applications **MUST** define and document:
  - **RTO (Recovery Time Objective)**: Maximum acceptable downtime
    - Critical services: RTO ≤ 1 hour
    - Standard services: RTO ≤ 4 hours
    - Non-critical services: RTO ≤ 24 hours
  - **RPO (Recovery Point Objective)**: Maximum acceptable data loss
    - Critical data: RPO ≤ 15 minutes (continuous backup/replication)
    - Standard data: RPO ≤ 1 hour
    - Non-critical data: RPO ≤ 24 hours

- [COMP025.2-dr-plan] **Disaster Recovery Plan**: Each application **MUST** have a documented DR plan including:
  - Step-by-step recovery procedures
  - Roles and responsibilities during incident
  - Communication protocols and escalation paths
  - Dependency mapping (what must be restored first)
  - Validation steps to confirm successful recovery
  - Fallback procedures if recovery fails

- [COMP025.3-backup-strategy] **Backup Strategy**: Applications **MUST** implement appropriate backup mechanisms:
  - Automated database backups (daily minimum, ideally continuous)
  - Point-in-time recovery capability for databases
  - File storage backups (if applicable)
  - Backup retention policy (typically 30 days minimum)
  - Backup encryption for sensitive data
  - Off-site/geo-redundant backup storage

### Backup Testing and Validation

- [COMP025.4-backup-testing] **Backup Verification**: Backups **MUST** be tested regularly:
  - Automated backup verification (checksum validation)
  - Monthly restore tests to non-production environment
  - Quarterly full disaster recovery simulation
  - Document actual restore times vs. RTO targets
  - Validate data integrity after restore

- [COMP025.5-restore-procedures] **Restore Procedures**: Applications **MUST** document and test:
  - Automated restore scripts where possible
  - Manual restore procedures with step-by-step instructions
  - Data validation queries to confirm successful restore
  - Application startup sequence after restore
  - Smoke tests to verify functionality

### Resilience Testing (Chaos Engineering)

- [COMP025.6-chaos-testing] **Controlled Failure Testing**: Applications **SHOULD** undergo regular resilience testing:
  - **Pod Failures**: Randomly terminate pods to test self-healing
  - **Node Failures**: Simulate node crashes to test failover
  - **Network Latency**: Inject latency to test timeout and retry behavior
  - **Dependency Failures**: Simulate database or API failures to test circuit breakers
  - **Resource Exhaustion**: Test behavior under CPU/memory pressure
  - Start with non-production environments, gradually move to production

- [COMP025.7-chaos-frequency] **Testing Schedule**: Resilience testing **SHOULD** be performed:
  - Development/Test environments: Weekly
  - Acceptance environment: Monthly
  - Production environment: Quarterly (with proper planning and communication)
  - Game Days: Semi-annual full-scale disaster recovery exercises

- [COMP025.8-failure-injection] **Failure Injection Tools**: Consider using:
  - Chaos Mesh for Kubernetes
  - Azure Chaos Studio
  - Gremlin or similar chaos engineering platforms
  - Custom scripts for controlled failure scenarios

### Recovery Time Testing

- [COMP025.9-rto-validation] **RTO Validation**: Applications **MUST**:
  - Measure actual recovery time during DR tests
  - Compare against documented RTO targets
  - Identify and optimize slow recovery steps
  - Update DR documentation with real metrics
  - Report RTO violations to stakeholders

- [COMP025.10-rpo-validation] **RPO Validation**: Applications **MUST**:
  - Verify backup frequency meets RPO requirements
  - Test point-in-time recovery accuracy
  - Measure data loss during simulated failures
  - Validate backup completeness and integrity

### Post-Incident Review

- [COMP025.11-post-mortem] **Post-Incident Analysis**: After DR tests or real incidents:
  - Conduct blameless post-mortem reviews
  - Document lessons learned
  - Create action items for improvements
  - Update runbooks and DR procedures
  - Share findings with development and operations teams
  - Track and verify implementation of improvements

### Documentation Requirements

- [COMP025.12-dr-documentation] **DR Documentation MUST** include:
  - Current RTO and RPO values
  - Last successful DR test date and results
  - Backup schedule and retention policy
  - Restore procedure step-by-step guide
  - Dependency diagram for recovery sequencing
  - Contact information for incident response team
  - Links to monitoring dashboards
  - Links to runbooks for common scenarios

**Example DR Test Checklist:**
```
- [ ] Notify stakeholders of planned DR test
- [ ] Verify latest backup availability
- [ ] Take pre-test snapshot of system state
- [ ] Begin timer for RTO measurement
- [ ] Simulate failure scenario (e.g., delete database)
- [ ] Execute recovery procedure
- [ ] Restore from backup
- [ ] Validate data integrity
- [ ] Run smoke tests
- [ ] Stop timer and record RTO
- [ ] Document any issues encountered
- [ ] Review results with team
- [ ] Update procedures based on findings
```

**Success Criteria:**
- Actual RTO ≤ Target RTO
- Actual RPO ≤ Target RPO  
- All data restored correctly
- All services functioning after recovery
- Zero security vulnerabilities introduced during recovery

Reference: [Azure Well-Architected Framework - Reliability](https://learn.microsoft.com/en-us/azure/well-architected/reliability/)

## Summary of Compliance Codes

| Code | Requirement | Priority |
|------|-------------|----------|
| COMP001-naming | Consistent service and resource naming | MUST |
| COMP002-versioning | Semantic versioning and version API | MUST |
| COMP003-containers | Container delivery and standards | MUST |
| COMP004-timezone | UTC timezone for logging | MUST |
| COMP005-software-supply-chain | Security scanning and SBOM | MUST |
| COMP006-scalability | Horizontal scaling and graceful handling | MUST |
| COMP006.5-upgrade-duration | 15-minute upgrade time limit | MUST |
| COMP007-helm | Helm chart deployment | MUST |
| COMP008-automation | 100% automated deployment | MUST |
| COMP009-communication | Internal communication via APIs | SHOULD |
| COMP010-migrations | Automated database migrations | MUST |
| COMP011-stateless | Stateless applications with external storage | SHOULD |
| COMP012-secrets | Secure secrets management | MUST |
| COMP013-ssl | HTTP endpoints (SSL at gateway) | MUST |
| COMP014-healthchecks | Liveness, readiness, startup probes | MUST |
| COMP015-logging | Structured logging to STDOUT | MUST |
| COMP016-metrics | Prometheus metrics exposure | SHOULD |
| COMP017-resources | Resource requirements specification | MUST |
| COMP018-dns-flexibility | Multi-domain support | MUST |
| COMP019-release-notes | Comprehensive release documentation | MUST |
| COMP020-external-comm | Secure external communication | MUST |
| COMP021-security | Data security and privacy | MUST |
| COMP022-ibom | Interface documentation | MUST |
| COMP023-compatibility | Backward compatibility | MUST |
| COMP024-quality | Code quality and testing | MUST |
| COMP014-system-states | System state definitions and transitions | MUST |
| COMP014A-healthchecks | Enhanced health probe configuration | MUST |
| COMP009A-resilience | Redundancy and fault tolerance | MUST/SHOULD |
| COMP016A-alerting | Comprehensive alerting on critical thresholds | MUST |
| COMP025-disaster-recovery | Disaster recovery planning and testing | MUST |

## References and Further Reading

**Kubernetes Best Practices:**
- [Architecting Applications for Kubernetes | DigitalOcean](https://www.digitalocean.com/community/tutorials/architecting-applications-for-kubernetes)
- [15 Principles for Designing and Deploying Scalable Applications on Kubernetes | Elastisys](https://elastisys.com/designing-and-deploying-scalable-applications-on-kubernetes/)
- [The Twelve-Factor App](https://12factor.net/)

**Monitoring and Observability:**
- [Golden Signals for Kubernetes | Sysdig](https://sysdig.com/blog/golden-signals-kubernetes/)
- [Google SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

**Reliability and Resilience:**
- [Azure Well-Architected Framework - Reliability](https://learn.microsoft.com/en-us/azure/well-architected/reliability/)
- [Kubernetes Health Checks](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/)

**Security:**
- [OWASP Kubernetes Security Cheat Sheet](https://cheatsheetsecurity.com/resources/kubernetes-security-cheat-sheet/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)

**Logging:**
- [JSON Logging Guide | Better Stack](https://betterstack.com/community/guides/logging/json-logging/)

---

**Document Control:**
- This document is maintained in Git repository
- Updates via Pull Requests
- Version history tracked via Git commits
- Review cycle: Quarterly or as needed for significant changes

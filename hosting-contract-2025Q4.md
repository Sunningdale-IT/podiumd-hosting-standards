# PodiumD Hosting Standards 2025Q4

| Version | Date | Author | Comments |
|----|----|----|----|----|
| 2024Q3 | 27/8/2024 | Jim Leitch / SSC | Converted to Markdown |
| 2025Q4 | 27/10/2025 | SSC/Dimpact | Updated with SRE/DevOps best practices and Cap Gemini reliability recommendations |

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

### Redundancy and Fault Tolerance [COMP006A-redundancy]

Applications **MUST** implement redundancy and fault tolerance to meet reliability requirements.

- [COMP006A.1-multi-zone] Applications **SHOULD** support multi-zone deployments:
  - Distribute pods across Azure Availability Zones
  - Use pod anti-affinity rules to prevent co-location on same node/zone
  - Helm charts **SHOULD** include topology spread constraints for production deployments
  - Example configuration:
    ```yaml
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: service-name
    ```

- [COMP006A.2-retry-mechanisms] Applications **MUST** implement retry logic for transient failures:
  - Exponential backoff for retries (e.g., 100ms, 200ms, 400ms, 800ms)
  - Maximum retry attempts (typically 3-5)
  - Only retry idempotent operations or operations designed for safe retries
  - Retry on specific transient error codes (e.g., 429 Too Many Requests, 503 Service Unavailable, network timeouts)
  - Do NOT retry on client errors (4xx except 429) or authentication failures

- [COMP006A.3-circuit-breaker] Applications **SHOULD** implement circuit breaker pattern for external dependencies:
  - Open circuit after threshold of consecutive failures (e.g., 5 failures)
  - Half-open state for testing recovery after timeout period
  - Closed circuit when dependency recovers
  - Fail fast when circuit is open to avoid cascading failures and unnecessary delays
  - Libraries: Resilience4j (Java), Polly (.NET), circuitbreaker (Go), aiobreaker (Python)

- [COMP006A.4-timeouts] Applications **MUST** implement appropriate timeouts:
  - Connection timeout: 5-10 seconds
  - Request timeout: Based on expected response time (typically 30 seconds, max 120 seconds)
  - Idle timeout for persistent connections
  - Prevent resource exhaustion from hung connections

- [COMP006A.5-load-balancing] Applications **MUST** support load balancing:
  - Multiple replicas (minimum 2 for production, 3 recommended)
  - Kubernetes service for traffic distribution
  - Session affinity only when required (prefer stateless)
  - Health probe integration for automatic removal of unhealthy pods

- [COMP006A.6-bulkhead-pattern] Applications **SHOULD** implement resource isolation (bulkhead pattern):
  - Separate connection pools for different dependencies
  - Thread pool isolation for critical vs non-critical operations
  - Prevent resource exhaustion in one component from affecting others
  - Rate limiting to protect against overload

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

Applications **MUST** define and monitor clear operational states for observability and automated response.

- [COMP014.1-state-model] Applications **MUST** implement a health model with the following states:
  - **Healthy/Normal**: System operates within expected parameters, all dependencies available
  - **Degraded**: Reduced functionality but still operational (e.g., cache unavailable, secondary services down)
  - **Failed/Unhealthy**: System not functioning correctly, cannot serve requests
  - **Maintenance**: Planned downtime for updates/repairs (should be signaled via readiness probe)
  - **Recovery**: System returning to normal operation after incident (transitional state)

- [COMP014.2-state-transitions] Applications **SHOULD** log state transitions with:
  - Timestamp (UTC)
  - Previous state
  - New state
  - Reason for transition
  - Affected components

- [COMP014.3-state-exposure] Applications **MUST** expose current operational state via:
  - Health probe endpoints (HTTP status codes reflecting state)
  - Metrics endpoint (state as Prometheus gauge metric)
  - Structured logs with state information

### Health Probes [COMP014-healthchecks]

Applications **MUST** provide probe endpoints for Kubernetes health management that accurately reflect the defined system states.

- [COMP014.4-liveness] **Liveness probe**: Indicates if application is running
  - Should check if the application process is alive and not deadlocked
  - Should NOT check external dependencies (use readiness for that)
  - Failure triggers container restart
  - Recommended endpoint: `/health/live` or `/healthz`
  - Should return:
    - 200 OK: Application is alive
    - 500+ errors: Application should be restarted

- [COMP014.5-readiness] **Readiness probe**: Indicates if application can serve traffic
  - **MUST** check all critical dependencies (database, cache, required downstream services)
  - **SHOULD** reflect system state (healthy vs degraded vs failed)
  - Failure removes pod from load balancer temporarily
  - Recommended endpoint: `/health/ready` or `/readyz`
  - Should return:
    - 200 OK: Healthy state, ready to serve traffic
    - 503 Service Unavailable: Degraded or failed state, not ready for traffic

- [COMP014.6-startup] **Startup probe**: For slow-starting applications
  - Higher failure threshold and interval than liveness
  - Prevents premature container restarts during initialization
  - Recommended endpoint: `/health/startup` or same as liveness
  - Should return 200 OK once application initialization is complete

- [COMP014.7-probe-configuration] Probe configuration in Helm charts **MUST** include:
  - Initial delay appropriate for application startup time
  - Reasonable timeout (typically 1-5 seconds)
  - Appropriate failure threshold (typically 3 failures)
  - Period/interval matching application characteristics (typically 10-30 seconds)

- [COMP014.8-probe-performance] Probes **MUST**:
  - Respond within 1 second for liveness probes
  - Respond within 5 seconds for readiness probes (if checking external dependencies)
  - Not have business logic side-effects
  - Be lightweight (no heavy database queries or complex computations)
  - Use caching for expensive dependency checks (with appropriate TTL)

- [COMP014.9-probe-details] Probe endpoints **SHOULD** provide detailed status information:
  ```json
  {
    "status": "healthy",
    "timestamp": "2025-10-27T12:00:00Z",
    "version": "1.2.3",
    "dependencies": {
      "database": "healthy",
      "cache": "degraded",
      "downstream-api": "healthy"
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

Applications and infrastructure **MUST** have alerts configured for critical thresholds to enable proactive incident response.

- [COMP016A.1-resource-alerts] Resource usage alerts **MUST** be configured for:
  - **CPU Usage**: Alert when >80% for 5+ minutes, critical at >90%
  - **Memory Usage**: Alert when >80% of limit, critical at >90%
  - **Disk Usage**: Alert when >75% for persistent volumes, critical at >85%
  - **Network I/O**: Alert on sustained high throughput or errors
  - **Pod count**: Alert when approaching max replicas (autoscaling limit)

- [COMP016A.2-response-time-alerts] Response time/latency alerts **MUST** include:
  - P95 latency exceeding SLA thresholds (e.g., >500ms for interactive requests)
  - P99 latency for early degradation detection
  - Sudden latency spikes (e.g., 2x normal baseline)
  - Timeout rate increasing

- [COMP016A.3-error-rate-alerts] Error rate alerts **MUST** monitor:
  - HTTP 5xx error rate >1% of total requests
  - HTTP 4xx error rate sudden increases (>5x baseline)
  - Database connection errors
  - Failed health checks
  - Application-specific business logic errors

- [COMP016A.4-availability-alerts] Availability alerts **MUST** include:
  - Service uptime <99.9% (or defined SLA)
  - Pod restart frequency (>3 restarts in 10 minutes)
  - Readiness probe failures
  - Certificate expiration warnings (30, 14, 7 days before expiry)

- [COMP016A.5-dependency-alerts] Dependency health alerts **SHOULD** monitor:
  - Database connection pool exhaustion
  - Cache hit rate degradation
  - External API availability
  - Message queue depth exceeding thresholds

- [COMP016A.6-alert-routing] Alerts **MUST** be routed appropriately:
  - Critical alerts: Immediate notification (PagerDuty, on-call)
  - Warning alerts: Notification during business hours
  - Info alerts: Dashboard/logging only
  - Include runbook links in alert descriptions
  - Avoid alert fatigue through proper threshold tuning

- [COMP016A.7-alert-metadata] Alerts **SHOULD** include:
  - Severity level (critical, warning, info)
  - Service/component name
  - Environment (dev, test, acc, prod)
  - Direct link to relevant dashboard
  - Suggested remediation steps or runbook URL

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

### Disaster Recovery and Testing [COMP025-disaster-recovery]

Applications and infrastructure **MUST** have documented disaster recovery plans and regular testing procedures.

- [COMP025.1-disaster-recovery-plan] Each application **MUST** have a documented disaster recovery plan including:
  - **Recovery Time Objective (RTO)**: Maximum acceptable downtime (e.g., 4 hours)
  - **Recovery Point Objective (RPO)**: Maximum acceptable data loss (e.g., 1 hour)
  - Step-by-step recovery procedures
  - Communication plan during disasters
  - Roles and responsibilities
  - Escalation procedures

- [COMP025.2-backup-strategy] Backup strategy **MUST** include:
  - **Database backups**: Automated daily backups with point-in-time recovery
  - **Configuration backups**: Version-controlled Helm values, ConfigMaps, Secrets
  - **Persistent volume backups**: Regular snapshots for stateful data
  - Backup retention policy (e.g., 30 days for daily, 12 months for monthly)
  - Off-site/geo-replicated backup storage
  - Encryption of backups at rest

- [COMP025.3-backup-testing] Backup verification **MUST** include:
  - **Monthly backup restoration tests**: Verify backups can be restored successfully
  - **Quarterly full recovery drills**: Complete disaster recovery exercise in test environment
  - Document restoration time and compare against RTO
  - Test with production-sized data volumes
  - Validate data integrity after restoration

- [COMP025.4-chaos-engineering] Chaos engineering practices **SHOULD** be implemented:
  - **Controlled failure testing**: Deliberately inject failures to test resilience
  - Pod deletion/eviction tests (simulate node failures)
  - Network latency and partition tests
  - Resource exhaustion tests (CPU/memory pressure)
  - Dependency failure simulation
  - Run in non-production environments first, gradually in production with safety controls
  - Use tools like Chaos Mesh, Litmus, or Azure Chaos Studio

- [COMP025.5-rto-rpo-validation] RTO and RPO validation **MUST** occur:
  - At least quarterly for critical services
  - After major architecture changes
  - After significant version upgrades
  - Document actual recovery times vs target RTO/RPO
  - Update disaster recovery plans based on test results

- [COMP025.6-failover-testing] Failover procedures **SHOULD** be tested:
  - Database failover to replica/standby
  - Zone/region failover capabilities
  - Load balancer failover
  - DNS failover procedures
  - Automated failover mechanisms validation

- [COMP025.7-incident-postmortems] Post-incident reviews **MUST** include:
  - Timeline of events
  - Root cause analysis
  - Impact assessment (affected users, data loss, downtime)
  - Lessons learned
  - Action items to prevent recurrence
  - Update runbooks and disaster recovery plans based on findings

**Reference environments for testing:**
- Use acceptance environment for disaster recovery testing
- Test environment for chaos engineering experiments
- Production-like data volumes for realistic testing

**Success Criteria:**
- All critical services meet defined RTO/RPO targets
- Backup restoration completes successfully in tests
- Chaos engineering tests demonstrate system resilience
- Team members can execute recovery procedures without escalation

## Summary of Compliance Codes

| Code | Requirement | Priority |
|------|-------------|----------|
| COMP001-naming | Consistent service and resource naming | MUST |
| COMP002-versioning | Semantic versioning and version API | MUST |
| COMP003-containers | Container delivery and standards | MUST |
| COMP004-timezone | UTC timezone for logging | MUST |
| COMP005-software-supply-chain | Security scanning and SBOM | MUST |
| COMP006-scalability | Horizontal scaling and graceful handling | MUST |
| COMP006A-redundancy | Redundancy, fault tolerance, circuit breakers | MUST/SHOULD |
| COMP006.5-upgrade-duration | 15-minute upgrade time limit | MUST |
| COMP007-helm | Helm chart deployment | MUST |
| COMP008-automation | 100% automated deployment | MUST |
| COMP009-communication | Internal communication via APIs | SHOULD |
| COMP010-migrations | Automated database migrations | MUST |
| COMP011-stateless | Stateless applications with external storage | SHOULD |
| COMP012-secrets | Secure secrets management | MUST |
| COMP013-ssl | HTTP endpoints (SSL at gateway) | MUST |
| COMP014-system-states | Operational state definitions and monitoring | MUST |
| COMP014-healthchecks | Liveness, readiness, startup probes | MUST |
| COMP015-logging | Structured logging to STDOUT | MUST |
| COMP016-metrics | Prometheus metrics exposure | SHOULD |
| COMP016A-alerting | Critical threshold alerting | MUST |
| COMP017-resources | Resource requirements specification | MUST |
| COMP018-dns-flexibility | Multi-domain support | MUST |
| COMP019-release-notes | Comprehensive release documentation | MUST |
| COMP020-external-comm | Secure external communication | MUST |
| COMP021-security | Data security and privacy | MUST |
| COMP022-ibom | Interface documentation | MUST |
| COMP023-compatibility | Backward compatibility | MUST |
| COMP024-quality | Code quality and testing | MUST |
| COMP025-disaster-recovery | Disaster recovery plans and testing | MUST |

## References and Further Reading

**Kubernetes Best Practices:**
- [Architecting Applications for Kubernetes | DigitalOcean](https://www.digitalocean.com/community/tutorials/architecting-applications-for-kubernetes)
- [15 Principles for Designing and Deploying Scalable Applications on Kubernetes | Elastisys](https://elastisys.com/designing-and-deploying-scalable-applications-on-kubernetes/)
- [The Twelve-Factor App](https://12factor.net/)

**Monitoring and Observability:**
- [Golden Signals for Kubernetes | Sysdig](https://sysdig.com/blog/golden-signals-kubernetes/)
- [Google SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

**Security:**
- [OWASP Kubernetes Security Cheat Sheet](https://cheatsheetsecurity.com/resources/kubernetes-security-cheat-sheet/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)

**Reliability and Resilience:**
- [Azure Well-Architected Framework - Reliability](https://learn.microsoft.com/en-us/azure/well-architected/reliability/)
- [Kubernetes Health Checks](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Circuit Breaker Pattern - Microsoft](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Retry Pattern - Microsoft](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry)
- [Health Endpoint Monitoring Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/health-endpoint-monitoring)

**Logging:**
- [JSON Logging Guide | Better Stack](https://betterstack.com/community/guides/logging/json-logging/)

---

**Document Control:**
- This document is maintained in Git repository
- Updates via Pull Requests
- Version history tracked via Git commits
- Review cycle: Quarterly or as needed for significant changes

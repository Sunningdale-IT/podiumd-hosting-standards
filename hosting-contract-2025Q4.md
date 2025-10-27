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

Each customer/gemeente utilizes a set of Azure Subscriptions - one subscription for each OTAP environment. 

**Isolation Benefits:**
- Simplifies billing and cost allocation
- Provides security demarcation between environments at customer, supplier and OTAP levels
- Enables least-privilege access control
- Supports quota management and cost control
- Facilitates network segregation

**Uniform Management:**
- Network segregation applied consistently
- IAM roles and policies deployed uniformly via Infrastructure as Code
- Not every customer/supplier requires the full set of OTAP environments

### Cloud Resource Provider

We currently use **Microsoft Azure** as our cloud platform. However, the municipality **MUST** retain the ability to migrate to alternative cloud platforms at any time. 

**Vendor Lock-in Avoidance Strategy:**
- Deployment efforts **MUST** use **open standards** and be as cloud-agnostic as possible
- We intentionally avoid PaaS lock-ins unless explicitly agreed and justified
- Any managed service used **MUST** have a documented exit plan:
  - Export/backup format defined
  - Open protocol support verified
  - Migration path documented
- We rely on a coherent suite of **open-source management tools** rather than cloud-specific services
- Infrastructure as Code (IaC) **SHOULD** be portable across cloud providers where feasible

### Open Source Infrastructure Tools

Core platform components are **open source** and **version pinned** to ensure consistency and control:

- **Ingress Controller** - NGINX or similar open-source ingress
- **Service Mesh** - (if applicable) Open-source service mesh solutions
- **GitOps Operator** - ArgoCD, Flux, or similar for declarative deployments
- **Monitoring Stack** - Prometheus, Grafana for metrics and visualization
- **Logging Stack** - Open-source log aggregation and analysis
- **Policy Engine** - OPA (Open Policy Agent) or similar for policy enforcement

**Tool Management:**
- Upgrades follow a controlled change process with rollback plans
- Vulnerability scanning (containers, Helm charts, dependencies) is continuous
- An external market party maintains these management tools, resolves vulnerabilities, and ensures coherence
- Version upgrades are coordinated to maintain compatibility and enable easy Kubernetes version migration

### Documentation Platform

**All platform and component documentation** resides in the internal municipal documentation space:

- [COMP025-documentation-platform] Documentation **MUST** be stored in secured wiki or Git repository
- [COMP025.1-sensitive-data] Sensitive configuration details (IPs, keys, network topologies) **MUST** be stored only internally
- [COMP025.2-no-public-secrets] No sensitive information **MAY** be stored in public repositories
- [COMP025.3-access-control] Documentation access **MUST** be controlled via role-based access

### Azure Resources

Only **generic, cloud-agnostic** Azure resources are used:

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

## Software Placement Criteria

This section defines which software components are permitted on the PodiumD hosting environment ("Haven").

### Permitted Software [COMP026-software-criteria]

#### On Haven: Developed from Common Ground

- [COMP026.1-common-ground-native] Custom or municipality-specific components developed following **Common Ground principles** are permitted:
  - Modularity and separation of concerns
  - API-first design
  - Data ownership at source
  - Interoperability through standard interfaces

#### On Haven: Open Source Aligned with Common Ground

- [COMP026.2-open-source-aligned] Mature **open-source components** aligned with Common Ground architectural tenets may be onboarded if they meet:
  - Security requirements (vulnerability scanning, SBOM)
  - Maintenance criteria (active development, regular updates)
  - Compatibility with existing platform components
  - Documentation standards

### Prohibited Software [COMP026.3-prohibited]

The following types of software are **NOT** permitted on Haven:

- **Closed source black-box solutions** without adequate transparency
- Components with **vendor lock-in risk** or proprietary dependencies
- **Unsupported or end-of-life software** without active maintenance
- Software **failing security or privacy requirements**
- Components incompatible with Common Ground principles

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
  - Regularly refresh base images with security patches

- [COMP003.6-non-root] Containers **MUST NOT** run as root unless explicitly justified and approved

- [COMP003.7-immutable-digests] Production deployments **SHOULD** reference immutable image digests in Helm values
  - Use SHA256 digests in addition to version tags: `image@sha256:abc123...`
  - Ensures exact image reproducibility
  - Prevents tag mutations

### Timezone [COMP004-timezone]

For logging and monitoring purposes, timezone **MUST** be **UTC**. Default database timezone is UTC. All logging and metrics **MUST** use UTC output. End-user interfaces **SHOULD** present data in appropriate local timezone.

### Software Supply Chain Security [COMP005-software-supply-chain]

SSC Hosting scans all containers for vulnerabilities regularly.

- [COMP005.1-scanning] Containers **MUST** be scanned for vulnerabilities by the supplier before delivery

- [COMP005.2-sbom] Each release **MUST** include a Software Bill of Materials (SBOM) in CycloneDX or SPDX format
  - Machine-readable format required
  - Include all dependencies with versions
  - Update SBOM with each release

- [COMP005.3-vulnerability-remediation] Vulnerabilities flagged **MUST** be fixed, mitigated, or receive waiver from system architects (supplier + Dimpact + SSC Hosting) within:
  - Critical: 7 days
  - High: 30 days
  - Medium: 90 days

- [COMP005.4-dependency-updates] Dependencies **SHOULD** be kept current; deprecated APIs removed proactively

- [COMP005.5-signed-images] Container images **SHOULD** be signed using cosign or similar
  - Enables verification of image provenance
  - Protects against supply chain tampering

- [COMP005.6-build-integrity] Implement dependency integrity checks:
  - Checksums for all dependencies
  - Signature verification where available
  - Locked dependency versions (no floating versions)

- [COMP005.7-build-pipeline-security] Build pipelines **MUST** be secured:
  - Isolated build runners
  - Secrets management (no secrets in code or logs)
  - Multi-stage builds to reduce attack surface
  - Audit logging of build activities

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

PodiumD as a whole is deployed in one single **"podiumd"** kubernetes namespace for simplified service discovery and communication.

**Namespace Isolation Requirements:**

- [COMP028-namespace-isolation] Each component (or logical suite) **MUST** be isolated with proper resource controls:
  - Clear **resource quotas** defined (CPU, memory, storage limits)
  - **Network policies** restricting egress/ingress to minimum required
  - Namespace-level RBAC policies for access control
  
- [COMP028.1-resource-quotas] Resource quotas **MUST** prevent resource exhaustion:
  ```yaml
  # Example namespace quota
  resourceQuota:
    hard:
      requests.cpu: "10"
      requests.memory: "20Gi"
      limits.cpu: "20"
      limits.memory: "40Gi"
      persistentvolumeclaims: "10"
  ```

- [COMP028.2-network-policies] Network policies **MUST** restrict network traffic:
  - Default deny ingress/egress
  - Explicit allow rules for required communication
  - Minimize lateral movement risk
  
  ```yaml
  # Example: Allow only necessary component communication
  networkPolicy:
    podSelector:
      matchLabels:
        app: open-zaak
    ingress:
      - from:
        - podSelector:
            matchLabels:
              app: open-forms
        ports:
        - protocol: TCP
          port: 8000
  ```

All PodiumD components share the "podiumd" namespace but are subject to these isolation controls.

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
  - Integration patterns: REST/JSON, async messaging (queues, events), or other standard open protocols

- [COMP009.3-async-preferred] Asynchronous communication **SHOULD** be preferred where latency tolerance exists
  - Events, message queues for decoupling
  - Synchronous calls **MUST** include timeouts, retries with backoff, and circuit breaking

- [COMP009.4-secure-channels] All inter-component communication **MUST** use secure, mutually authenticated channels
  - TLS for encrypted transport
  - Mutual authentication where security policy requires

- [COMP009.5-integration-layer] External exposure occurs **only** via the platform ingress layer with central policies:
  - Rate limiting
  - Web Application Firewall (WAF)
  - Auditing and logging

- [COMP009.6-no-direct-db] Direct database access across components is **prohibited**
  - Only exposed service APIs may be used for inter-component communication

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
  - Support horizontal scaling without shared local state
  - Explicitly document any state handling mechanisms

- [COMP011.2-object-storage] Files **SHOULD** be written to object store (Azure Blob Storage)

- [COMP011.3-file-shares] Files **MAY** be written to cloud file shares

- [COMP011.4-no-local-storage] Persistent files **MUST NOT** be written to local container storage but to Kubernetes PV

- [COMP011.5-permanent-data] Permanent data requiring backup **MUST** be written to file share created outside K8s cluster

- [COMP011.6-session-externalization] Session state **MUST** be externalized (Redis, database, or distributed cache)

- [COMP011.7-concurrency-safety] Applications **MUST** ensure concurrency safety
  - No race conditions causing data corruption
  - Proper locking mechanisms for shared resources
  - Transaction isolation where needed

**File storage preference order:**
1. Object Storage (Azure Blob)
2. SMB file share
3. Persistent volume storage

### Configuration / Secrets

- [COMP012.1-env-vars] All application configuration **SHOULD** use environment variables

- [COMP012.2-configmaps] Environment variables **SHOULD** be set via ConfigMaps in Helm chart

- [COMP012.3-secrets] Secret/sensitive information **MUST** use K8s secrets store
  - Secrets sourced from Azure Key Vault at deployment time
  - Kubernetes Secrets encrypted at rest
  - No secrets in code, container images, or version control

- [COMP012.4-secret-rotation] Applications **SHOULD** support secret rotation without restart
  - Watch for secret changes
  - Reload credentials dynamically where possible

- [COMP012.5-log-verbosity] Logging verbosity **SHOULD** be controlled by environment variable (e.g., `LOG_LEVEL=DEBUG`)

- [COMP012.6-no-secrets-in-images] No secrets **MAY** be embedded in container image layers or VCS
  - Scan images for accidentally committed secrets
  - Use .gitignore and .dockerignore appropriately

### SSL Certificates [COMP013-ssl]

- [COMP013.1-http-endpoints] Application endpoints **MUST** be presented as HTTP (unencrypted)
  - Azure Application Gateway performs SSL termination
  - Secure path to k8s cluster via Azure VWAN and Virtual Hub
  - LetsEncrypt SSL certificates populated at deploy-time

- [COMP013.2-internal-tls] Internal service-to-service communication **MAY** use mutual TLS if required for compliance

### Health Probes [COMP014-healthchecks]

Applications **MUST** provide probe endpoints for Kubernetes health management.

- [COMP014.1-liveness] **Liveness probe**: Indicates if application is running
  - Should check critical dependencies
  - Failure triggers container restart
  - Recommended: `/health/live` or `/healthz`

- [COMP014.2-readiness] **Readiness probe**: Indicates if application can serve traffic
  - Should check all dependencies (database, cache, downstream services)
  - Failure removes pod from load balancer
  - Recommended: `/health/ready` or `/readyz`

- [COMP014.3-startup] **Startup probe**: For slow-starting applications
  - Higher failure threshold and interval
  - Recommended: `/health/startup` or same as liveness

- [COMP014.4-probe-performance] Probes **MUST**:
  - Respond within 1 second
  - Not have business logic side-effects
  - Be lightweight (no heavy database queries)

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
  - Handle failures gracefully (no cascading failures)
  - Avoid leaking sensitive data in logs
  - Include timeouts and retries
  - Be explicitly whitelisted in network policies

- [COMP020.2-cross-environment] Cross-environment data flows **MUST**:
  - Have prior security and privacy assessment
  - Use explicit whitelisting (no blanket cross-environment access)
  - Use encrypted transport (TLS 1.2+)
  - Be documented in architecture diagrams

- [COMP020.3-email-service] Email services **MUST**:
  - Use approved centralized mail relay
  - Use authenticated API or SMTP over TLS
  - No direct unmanaged outbound mail from containers

- [COMP020.4-municipal-environments] Communication with other municipal environments **REQUIRES**:
  - Prior security assessment
  - Privacy impact assessment when handling personal data
  - Explicit network whitelisting
  - Encrypted channels (TLS 1.2+)
  - Documented data flows

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

### User Access and Authentication [COMP027-user-access]

User-facing applications **MUST** follow centralized authentication patterns:

- [COMP027.1-centralized-identity] User access occurs via designated frontend entrypoints and identity provider (IdP) integrations
  - Single Sign-On (SSO) / federated identity preferred
  - Azure Entra ID integration where applicable
  
- [COMP027.2-no-adhoc-auth] No component **MAY** implement ad-hoc authentication bypassing centralized identity management
  - Custom authentication requires explicit waiver
  - Migration plan to centralized identity required for waivers

- [COMP027.3-deployment-strategies] Production rollouts **SHOULD** use risk-appropriate deployment strategies:
  - **Blue/Green deployments** - Full environment switch for high-risk changes
  - **Canary deployments** - Gradual traffic shifting for validation
  - **Rolling updates** - Standard approach for low-risk updates

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

- **Git is the single source of truth** for all infrastructure and application configurations
- **Changes via Pull Requests** with:
  - Branch protection enabled
  - Mandatory code reviews
  - Automated lint and security checks
  - CI/CD validation before merge
- **Merge to main triggers GitOps operator** which reconciles declared desired state to the cluster
- **Drift detection** alerts when live state diverges from Git
- **Remediation via commits**, not manual `kubectl` changes
  - Exception: Emergency hotfixes allowed, but **MUST** be followed by post-factum commit
  - All kubectl changes must be recorded in Git within 24 hours

### Access Control

**Service providers have restricted, controlled access:**

- **Submit changes via Pull Requests** - No direct production changes
- **Direct cluster access restricted** - Read-only where granted
- **Namespace-scoped permissions**:
  - Read/list within assigned namespace(s)
  - Create/update only for owned resources
  - Least-privilege RBAC policies enforced
- **No cluster-admin access** - System-wide changes managed by SSC
- **Audit logging** - All access and changes logged for compliance

## Quality and Maintainability

- [COMP024.1-code-quality] Code **MUST** be:
  - **Version controlled** in Git or equivalent VCS
  - **Peer reviewed** before merge to main branch
  - **Tested** (unit tests, integration tests where feasible)
  - **Free of critical/high vulnerabilities** at release time
  - **Dependencies kept current**; deprecated APIs removed proactively

- [COMP024.2-testing] Applications **SHOULD** include:
  - Unit tests (>80% coverage for critical paths)
  - Integration tests for component interactions
  - End-to-end smoke tests for critical workflows
  - Performance tests for critical workflows
  - Security tests (SAST/DAST where applicable)

- [COMP024.3-documentation] Applications **MUST** provide:
  - Architecture documentation (component diagrams, data flows)
  - Deployment documentation (installation, configuration)
  - Operational runbooks (common tasks, procedures)
  - Troubleshooting guides (common issues, resolutions)
  - API documentation (OpenAPI/Swagger for REST APIs)

## General Principles

The following principles guide all aspects of the PodiumD hosting environment and supplier relationships:

### Security and Privacy by Design

- **Least Privilege Access**: All components, service accounts, and users operate with minimum necessary permissions
- **Defense in Depth**: Multiple layers of security controls (network, application, data)
- **Privacy by Default**: Personal data handling follows GDPR and minimization principles
- **Zero Trust**: No implicit trust; verify all access requests

### Operational Excellence

- **Auditability**: All changes tracked and logged; immutable audit trails
- **Reproducible Builds**: Build process deterministic and repeatable
- **Infrastructure as Code**: All infrastructure declarative and version controlled
- **Automated Testing**: Comprehensive test coverage at all levels

### Open Standards and Portability

- **Open Standards First**: Prefer open protocols and formats over proprietary
- **Cloud Agnostic**: Minimize vendor lock-in; maintain migration capability
- **Interoperability**: Components integrate via standard APIs
- **Open Source Preference**: Leverage and contribute to open source

### Reliability and Resilience

- **Graceful Degradation**: Services degrade gracefully under failure
- **Automated Recovery**: Self-healing systems where possible
- **Chaos Engineering**: Proactively test failure scenarios
- **Observability**: Comprehensive monitoring, logging, and tracing

### Continuous Improvement

- **Feedback Loops**: Regular reviews and retrospectives
- **Metrics-Driven**: Decisions based on data and measurements
- **Learning Culture**: Share knowledge and lessons learned
- **Innovation**: Encourage experimentation within safe boundaries

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
| COMP025-documentation-platform | Internal documentation storage | MUST |
| COMP026-software-criteria | Software placement criteria | MUST |
| COMP027-user-access | Centralized authentication and deployment strategies | MUST |
| COMP028-namespace-isolation | Namespace resource quotas and network policies | MUST |

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

**Logging:**
- [JSON Logging Guide | Better Stack](https://betterstack.com/community/guides/logging/json-logging/)

---

**Document Control:**
- This document is maintained in Git repository
- Updates via Pull Requests
- Version history tracked via Git commits
- Review cycle: Quarterly or as needed for significant changes

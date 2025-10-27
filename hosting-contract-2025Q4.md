# PodiumD Hosting Standards - 2025Q4

| Version | Date | Author | Comments |
|----|----|----|----|----|
| 0.1 | 20/02/2024 | Jim Leitch | Sander vd B, Andrew M |
| 0.8 | 4/3/2024 | Jim Leitch | Stephan Z, Jesse H, Mahmut C, Petra C |
| 1.0 | 22/4/2024 | Jim Leitch | SSC/Maykin/Dimpact |
| 1.1 | 6/5/2024 | Jim Leitch |  |
| 1.2 | 29/5/2024 | Jim Leitch | Dimpact/ICATT | Updated "k8s operators" |
| 2024Q3 | 27/8/2024 | Jim Leitch | SSC | Converted to Markdown |
| 2025Q4 | 27/10/2024 | Jim Leitch | Updated based on WAF, Google SRE, and Cap Gemini assessment |

## Executive Summary

The PodiumD hosting environment runs a collection of Common Ground-Compliant resources currently hosted on Microsoft Azure.

Development, Test, Acceptance and Production (OTAP) environments are provided for multiple Dutch city councils (municipalities) to host the PodiumD application, a replacement for the current E-suite workflow system as used by many municipalities.

PodiumD is a collection of Open-Source components built by external software developers funded by the municipalities.

This document is meant to formalize what SSC (the hosting provider) expects from Dimpact (the contracting party) in terms of what Dimpact in-turn expect from their contracted software developers.

This document describes how SSC expects that software developers should package their software to be integrated into the SSC hosting environment and describes what can be expected from us, SSC Hosting, the hosting provider.

Rather than this document **pre**scribing how developers should deliver their software, it is meant as a document that describes what SSC Hosting and the developers expect from each other. It will develop over time with input from all parties.

The document will be hosted in a GIT repository and will be updated with versioning by means of Git Pull Requests.

## Document Conventions

We will use RFC-style terms to describe the level of requirements. As time goes on various aspects of this contract will be made more and less forceful: "**MUST**"/"**MUST** **NOT**"/"**SHOULD**"/"**SHOULD** **NOT**"/"**MAY**"

([RFC 2119: Key words for use in RFCs to Indicate Requirement Levels (rfc-editor.org)](https://www.rfc-editor.org/rfc/rfc2119))

## Multiple Environments Require Easy Deployments

It should be clear that applications being deployed will be deployed for multiple municipalities in multiple OTAP environments. This means that 100% automated deployment is a hard requirement for any applications being deployed.

## Infrastructure Description

### Subscriptions / Environments

Each customer/gemeente utilises a set of Azure Subscriptions. One subscription for each of the OTAP environments. This simplifies billing and provides security demarcation between the various environments, at customers, suppliers and OTAP level.

Not every customer/supplier requires the full set of OTAP environments.

### Cloud Resource Provider

We currently use Microsoft's Azure as our cloud platform. Our focus is currently on Microsoft Azure; however, we MAY in the future to use alternative cloud platforms. Any efforts made for deployment to our cloud platform should be generic, using open standards, keeping in mind that we may move to an alternative or even multi-cloud platform in the future.

### Azure Resources

The resources currently in our Azure cloud environment, in use and available are described below. In time we may migrate or simultaneously deploy in an alternative cloud environment such as Google / AWS / Private cloud.

Any new Azure resources required can be discussed going forward, this is current list:

- **Kubernetes** (k8s) – AKS

  - We use standard AKS with two autoscaling groups of nodes, one for all application workloads, another for all management-type workloads.

- **Container Registry** – ACR

  - ACR with vulnerability scanning

- **SQL DBMS** - Azure Database for PostgreSQL flexible servers

  - PostgreSQL 14 is our standard database server, one instance per environment with multiple databases inside the server, one per application.

  - Database Timezone is UTC

- **Edge Load Balancer** - Azure Application Gateway

  - Azure Load Balancer is our internet-facing endpoint, providing SSL termination, HTTP -> HTTPS redirection, URL rewriting and distribution of traffic to the appropriate K8S cluster. We have one load balancer for ONTW/TEST and another for ACCP/PROD.

  - We may in future choose to use our on-premises Palo Alto and GW services for this function

- **Storage** - Azure Storage Accounts

  - Azure storage accounts are used for provision of Blob storage and Azure File Shares (SMB and NFS)

- **Secret and Certificate Management** – Azure Key Vault

  - Azure Key Vault is the single source of truth for all certificates and secrets

- **Identity Management** - Active Directory / Entra ID

  - Azure Entra ID SHOULD be used as the only method of Identity Management. If another identity tool is used, a waiver must be obtained with a plan for future migration to Entra.

- **Deployment Orchestration -** Azure DevOps

  - Azure DevOps is our CICD orchestration tool, used for deployment of infrastructure, application deployment and other management tasks. We use a mixture of Terraform and bash scripting with az cli commands to build and maintain our infrastructure.

- **Infra Lifecycle**

  - Most Azure infrastructure resources (storage, databases) have very long lifecycles.

  - Kubernetes versions are handled frequently with the assistance of Azure Fleet Manager and will be updated on a regular monthly cycle ***ontw/test/accp/prod*** with one week between each environment type, meaning that any non-critical k8s patches will have been running in ONTW for 3 weeks before production is patched

  - Upgrades to other resources such as RDBMS and storage will be announced well in advance to allow suppliers to test their applications with the new versions and to work on and plan an upgrade path for the application if required.

## Azure Application Gateway

Azure application gateway can rewrite HTTP headers. At this moment we apply only one header rewrite for HSTS purposes as follows:

We have not made any other changes to the AGW settings. For any other default settings please refer to the MS Azure documentation, any changes to default settings will be reflected in this document.

HTTP2 support is currently DISABLED in the AGW.

### Non – Azure Native Shared Resources

Some resources are not available as Azure native services and are instead deployed inside the Kubernetes cluster:

### SSL Certificates (FrontEnd)

- We use SSL certificates from **LetsEncrypt**. The certificates are automatically requested and downloaded for every web facing service by the cert-manager application inside the cluster. Certificates and keys are then stored in the Azure Key Vault. They are then subsequently deployed to the listeners in the Application Gateway.

- SSL Certificates used internally between apps and legacy system are out-of-scope of this discussion.

### Cost management

In future we may be able to provide extra cost management tooling using the billing data stored.

### Environments (OTAP)

Each municipality and organization has the possibility to run O,T,A and/or P organizations, defined as follows:

#### DEVELOPMENT(ONTWIKKEL)

Ontwikkel (development) environments are meant to verify proper working of the software in a production-similar environment. Developers are free to deploy container versions and new helm charts in this environment. Changes to Azure resources and the k8s cluster are performed by SCC hosting in an updated version of the environment.

To save on cloud costs we switch all dev, test and acceptance environment off in the evening and at weekends. If any individual environments need to be kept on for any reason for a short period of time, this can be accommodated.

#### Test

Test environments are meant for the municipalities to test new application features and provide training facilities to staff.

#### Acceptance

Acceptance environments are 100% like-for-like with production environments. Hardware resources are the same as production and copies of production data.

#### Production

Production is of course the environment per customer/municipality that has maximum uptime, monitoring, and resources.

## Deployment- Azure DevOps Pipelines

Deployment of applications to the k8s cluster are done by an Azure DevOps Pipeline to deploy a Helm Chart. Any other infrastructure preparation is done by Terraform files and/or bash scripts running "az cli" commands.

## What We Expect

We expect that Dimpact will require suppliers to deliver the applications in a way that allows for secure, scalable, stable, manageable, and easily deployable systems.

### Naming Conventions [COMP001-naming]

All application and parameter naming should be uniform to allow for a minimum of repeated configuration. 

**Service Naming Standards:**

- Service names MUST follow a consistent pattern: `{domain}-{function}-service` (e.g., `zaak-processing-service`, `klant-api-service`)
- Service names MUST use lowercase with hyphens as separators
- Service names MUST be descriptive and reflect the business domain
- Service names MUST be consistent across all environments (OTAP)
- Helm chart names MUST match the service name
- Kubernetes namespaces SHOULD follow the pattern: `{customer}-{service-name}` or use a single namespace per recognizable component
- Container image names MUST use the service name as base
- API endpoint paths SHOULD use the service domain name for consistency

**Naming Examples:**
- Good: `klant-management-service`, `zaak-api-service`, `document-storage-service`
- Avoid: `service1`, `api`, `app-x`, `temp-service`

Dimpact will provide examples of parameter naming to be used in the umbrella helm chart.

### Versioning [COMP002-versioning]

PodiumD consists of a suite of sub-applications. The PodiumD version is defined by the versions of the sub-applications.

### Version API Endpoint [COMP002.1-version-api]

- [COMP002.1-version-api] Each application/component MUST expose a standardized version API endpoint at `/api/v1/version` or `/health/version`

- The version endpoint MUST return the following information in JSON format:
  - Application/component name
  - Semantic version number
  - Build date/timestamp
  - Git commit SHA (optional but recommended)
  - Dependencies and their versions (optional but recommended)

- Example response:
```json
{
  "name": "zaak-api-service",
  "version": "2.5.1",
  "buildDate": "2024-10-15T14:30:00Z",
  "gitCommit": "a3f2d1c",
  "dependencies": {
    "postgres": "14.5",
    "redis": "7.0.5"
  }
}
```

- The version endpoint MUST be accessible without authentication
- The version endpoint SHOULD be included in health check monitoring

### Containers

- [COMP003.1-app-as-containers] Applications MUST be delivered as one or a set of application containers

- [COMP003.3-container-registry] These containers MUST be available via a public Container Registry such as **Docker Hub**

- Non-open-source containers MAY be available secured by a key/token.

- [COMP003.3-semantic-versioning] Container labels MUST be "semantically versioned."

- Prelease versions MAY be deployed with version + keyword (e.g., 1.2.3-snapshot)

SSC hosting will copy containers from the supplier's own container registry to a Container Registry in the SSC Azure environment.

### Timezone [COMP004-timezone]

For logging and monitoring purposes, the timezone for applications should be **UTC**. The default timezone for the database is UTC, all logging and metrics should be set to provide UTC output. Of course any end-user interaction should be presented to the client in the appropriate local timezone.

### Software Supply Chain [COMP005-software-supply-chain]

SSC Hosting will scan all containers for vulnerabilities on a regular basis. Containers delivered MUST be already scanned for vulnerabilities by the supplier. Any vulnerabilities that are flagged MUST be fixed, mitigated and/or receive a waiver from all system architects of supplier + Dimpact + SSC Hosting within a specified time limit.

An example of code-level scanning would be SonarQube:

<https://www.sonarsource.com/products/sonarqube/>

An example of container scanning would be GitHub container scanning:

<https://github.com/marketplace/actions/container-scan>

### Scalability and Startup/Shutdown

Apart from where inappropriate, applications SHOULD gracefully handle load-based up and down-scaling of containers.

[COMP006.1-stop-start-safe] Application MUST be able to withstand sudden stops and starts (for example a node failing or container-node migration event)

Applications MAY be deployed with init containers, startup and shutdown scripts.

### Component Upgrade Time Limits [COMP006.2-upgrade-time-limit]

- [COMP006.2-upgrade-time-limit] Any upgrade or deployment of a component MUST complete within **15 minutes maximum**

- This includes:
  - Container image pull time
  - Database migrations
  - Health check stabilization
  - Rolling update completion

- Deployments taking longer than 15 minutes MUST be split into multiple phases or require architectural review

- Zero-downtime deployments are STRONGLY RECOMMENDED using:
  - Rolling updates with appropriate readiness probes
  - Blue/green deployment strategies for major changes
  - Canary deployments for gradual rollouts

- Database migrations that may take longer MUST be handled separately:
  - Run migrations as separate jobs before deployment
  - Use backward-compatible schema changes
  - Implement feature flags for gradual activation

### Helm Charts . Deployment

- Dimpact will curate and provide all required Helm charts in one single repository

- [COMP007.1-helm] All applications MUST be accompanied by Helm Charts. Applications MAY use helm sub-charts and MUST be able to be deployed as a sub-chart themselves to facilitate and contribute to a complete "umbrella" installation of the PodiumD platform.

- [COMP007.2-operators] We do not support the use of applications and components deployed by means of Kubernetes Operators **unless** the installation and use of the operators can be encapsulated inside a single Umbrella Helm chart.

- [COMP007.3-dependancy-apps] Any dependency-applications MUST be deployed as Helm sub-charts.

- The application itself MUST be deployable as a sub chart as part of an over-encompassing PodiumD Helm chart. Over time all parties should work to harmonize variable naming in the Helm charts

- Deployments SHOULD be performed as a rolling update – unless waiver has been granted. Deployments and upgrades that involve database changes are currently exempt from this requirement.

- Helm charts SHOULD be available via the same artifacts' repository as the containers

- [COMP007.4-storage] Helm chart MUST support storage type via SSC's own storage choices (if file storage is required)

- [COMP007.5-ingress] Helm chart MUST support ingress that allows for SSC's own ingress choices (if ingress in required)

- [COMP007.5-database] Helm chart MUST support external cloud database usage (if database is required)

### K8s Namespaces

PodiumD as a whole is deployed in one single "podiumd" kubernetes namespace.

~~SSC hosting uses one single k8s namespace per recognizable (openforms, objects etc.) component. Components must be deployed into its own namespace in Kubernetes cluster.~~

~~The name of the namespace will always be the same as the name of the application as defined in the helm chart.~~

### 100% Automation

[COMP008.1-100percent-automated] PodiumD software will be deployed to many customers and environments, for this reason, all components of the application MUST be possible to deploy completely from the Helm chart.

In this case – 100% deployment means the containers running correctly inside the K8s cluster with the correct deployment parameters and secrets, connecting to other resources such as databases and other PodiumD applications.

Final configuration "inside" of the application is out of this scope.

To help understanding of each area of deployment, development party Maykin Media produced this useful set of definitions:

> **• OTAP-omgevingen**
>
> Ontwikkel-, Test-, Acceptatie, Productie-omgevingen
>
> **• Platform**
>
> Set aan componenten die in samenhang een oplossing bieden (bijv. Open Zaak en ZAC)
>
> **• Installatie**
>
> Het deployen van een component (bijv. Helm chart uitvoeren zodat Open Zaak en ZAC technisch draaien)
>
> **• Configuratie**
>
> Parameters zodat de component kan doen wat het moet doen (bijv. de koppeling tussen ZAC en Open Zaak)
>
> **• Inrichting**
>
> Functionele vulling van het component evt. met hele specifieke configuratie (bijv. zaaktypen, processen)

#### Application Addressing

Dimpact will provide application URLS for all public facing applications.

Dimpact will liase with gemeentes as to the application addressing used and for crestion of appropriate hostnames and cnames.

### Communication Between Components

[COMP009-internal-communication] Communication between components inside the same k8s cluster SHOULD take place inside the cluster. Communication and data transfer between components SHOULD take place via the APIs.

### Database Migrations

[COMP010-sql-migration-scripts] Applications MUST have all database migrations built in or provide config scripts for a DB migration tool such as Flyway or Liquibase.

### Application concurrency / Statelessness

- Applications SHOULD be stateless and commit all data to a database.

- Files SHOULD be written to an object store such as Azure Blob Storage

- Files MAY be written to cloud file shares.

- [COMP011-no-local-storage] Persistent files containing any data MUST NOT be written to local storage on the container but instead to a kubernetes PV.

- [COMP011-permanent-data] Permanent data that requires to be backed up must be written to a file share created outside of the K8s cluster

### FILES

Files MAY be stored in the following storage types, in order of preference:

- Object Storage

- SMB file share shared volume file storage

- Persistent volume storage

### Configuration / Secrets

- All application configurations SHOULD be performed via environment variables.

- Environment variables SHOULD be set via config maps in the helm chart.

- [COMP012-secrets] Secret/sensitive information MUST be written to the K8s secrets store.

- Secrets will be sourced and read from the Azure Key Vault at deployment-time.

- Application logging verbosity SHOULD be enabled by setting a debug environment variable and restarting the container.

### SSL Certificates

[COMP013-ssl] Application end points MUST be presented as HTTP, unencrypted. Our Public-facing Azure Application Gateway performs SSL termination and guarantees a secure path to the k8s cluster via the Azure VWAN and Virtual Hub facilities. The Azure Application Gateway listeners are populated by Lets Encrypt SSL certificates at deploy-time

### Health Checks and Probes

[COMP014-healthchecks] Applications MUST ensure availability of probe endpoints to allow for correct handling of starting, running and failing containers.

**Enhanced Health Check Requirements based on Azure Well-Architected Framework:**

#### Health State Definitions [COMP014.1-health-states]

Applications MUST clearly define and document the following operational states:

1. **Healthy/Normal**: System operates within expected parameters
   - All dependencies available
   - Response times within SLA
   - Error rates below threshold
   - Resources within normal limits

2. **Degraded**: Reduced functionality but still operational
   - Some non-critical dependencies unavailable
   - Response times elevated but acceptable
   - Operating in fallback mode
   - Should trigger warnings but not critical alerts

3. **Unhealthy/Failed**: System not functioning correctly
   - Critical dependencies unavailable
   - Unable to process requests
   - Data consistency issues
   - Should trigger immediate alerts

4. **Starting**: System is initializing
   - Loading configuration
   - Establishing database connections
   - Warming up caches
   - Not ready for traffic

5. **Stopping**: Graceful shutdown in progress
   - Completing in-flight requests
   - Closing connections
   - Cleaning up resources

#### Kubernetes Probe Configuration [COMP014.2-probe-config]

Applications MUST implement all three types of probes:

**Liveness Probe:**
- Determines if container needs to be restarted
- MUST be simple and fast (< 1 second response time)
- Should NOT check dependencies
- Example: Simple HTTP endpoint returning 200 OK if application can accept requests

**Readiness Probe:**
- Determines if container can receive traffic
- SHOULD check critical dependencies (database, required APIs)
- MAY have longer timeout than liveness probe
- Should transition pod out of service gracefully

**Startup Probe:**
- For slow-starting applications
- Disables liveness/readiness checks during startup
- Prevents premature container restarts
- Should have generous initial delay and timeout

**Probe Best Practices:**
- Initial delay should account for realistic startup time
- Failure threshold should avoid false positives from temporary issues
- Success threshold should prevent flapping
- Period should balance responsiveness with load
- Timeout should account for network variability

See [Configure Liveness, Readiness and Startup Probes | Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

### Redundancy and Fault Tolerance [COMP014.3-redundancy]

Based on Azure Well-Architected Framework reliability pillar:

**Multi-Zone Deployment:**
- Production deployments SHOULD spread replicas across multiple Azure Availability Zones
- Minimum 3 replicas for critical services
- Pod anti-affinity rules SHOULD be used to prevent co-location

**Retry Mechanisms:**
- Applications MUST implement exponential backoff for retrying failed requests
- Maximum retry attempts should be configurable
- Circuit breaker pattern SHOULD be implemented for external dependencies

**Load Balancing:**
- Traffic MUST be distributed across multiple instances
- Session affinity SHOULD be avoided where possible for better distribution
- Health checks must remove unhealthy instances from rotation

**Pod Disruption Budgets:**
- [COMP014.4-pdb] Critical services MUST define Pod Disruption Budgets
- Minimum available pods should ensure service continuity during updates
- Should balance update velocity with availability requirements

## Logging / Monitoring / Metrics / Alerting

### Monitoring

Monitoring is the combination of aggregating logs and metrics to ascertain if the application is working as expected, hopefully before the end user is affected and allow us to react in time to fix the problem and/or underlying issue.

### Infra monitoring

The infrastructure (resources such as k8s, edge gateway, databases etc.) SHALL be monitored for errors as exported to Azure. Other metrics such as CPU capacity, disk IO should be used to ensure rightsizing and autoscaling of resources.

### Logging

- [COMP015.1-logging-to-stdout] Applications MUST stream all logging to STDOUT. These logs will be captured and sent to the appropriate logging endpoint.

- Application logging MUST be configurable via a LOGLEVEL environment variable following standard usage for DEBUG, INFO, WARN, ERROR, and FATAL log levels.

- Applications logs SHOULD be written in JSON format to allow proper parsing in the Logging system. (see <https://betterstack.com/community/guides/logging/json-logging/>)

- Applications MAY write relevant logging information in a form that can be read in the application, but this MUST be in addition to the logs written to STDOUT.

- [COMP016.2-utc-logs] Log lines MUST be written with timestamps in UTC format, no time zone.

### Metrics - Golden Signals [COMP017-golden-signals]

Based on Google SRE best practices, applications MUST provide metrics for the four golden signals:

**1. Latency [COMP017.1-latency-metrics]**
- Request duration/response time
- Separate success and error latencies
- Percentile distributions (p50, p95, p99)
- Example metrics:
  - `http_request_duration_seconds`
  - `api_response_time_milliseconds`

**2. Traffic [COMP017.2-traffic-metrics]**
- Requests per second
- Concurrent connections
- Active users
- Example metrics:
  - `http_requests_total`
  - `active_connections`
  - `concurrent_users`

**3. Errors [COMP017.3-error-metrics]**
- Error rate (errors/total requests)
- Error types and codes
- Failed transactions
- Example metrics:
  - `http_requests_failed_total`
  - `transaction_errors_total`
  - `error_rate_percentage`

**4. Saturation [COMP017.4-saturation-metrics]**
- Resource utilization (CPU, memory, disk, network)
- Queue depth
- Thread pool usage
- Database connection pool usage
- Example metrics:
  - `cpu_usage_percentage`
  - `memory_usage_bytes`
  - `queue_depth`
  - `db_connection_pool_active`

**Additional Application Metrics:**

Applications SHOULD provide metrics that make sense for the application in hand, for example:

- Number of Transactions
- Transaction latency
- Incomplete transactions
- Failed transactions
- Any other relevant metrics per application

**Metrics Implementation:**

- Metrics SHOULD be exposed in Prometheus format at `/metrics` endpoint
- Metrics MUST include appropriate labels for filtering (environment, version, instance)
- Metrics collection SHOULD have minimal performance impact (< 1% overhead)
- Counter metrics SHOULD be monotonically increasing
- Gauge metrics SHOULD represent current values
- Histogram metrics SHOULD be used for latency measurements

The metrics can then be scraped by (for example) Prometheus to display in graph form, for analysis and for alerting purposes.

Metrics are an ongoing discussion and will always be updated and improved throughout the lifecycle of the application.

The following article gives a thorough discussion on the use of metrics usage in Kubernetes:

<https://sysdig.com/blog/golden-signals-kubernetes/>

### Alerting [COMP018-alerting]

Based on Google SRE principles:

**Alert Design Principles:**
- Alerts MUST only notify when human intervention is required
- Alerts SHOULD be based on symptoms (user impact) not causes
- Every alert MUST be actionable
- Alert descriptions MUST include:
  - What is broken
  - Impact on users/business
  - Next steps for investigation
  - Relevant runbook links

**Alert Severity Levels:**
1. **Critical/P1**: Production down, major functionality broken, immediate action required
2. **Warning/P2**: Degraded performance, potential issue, action required within business hours
3. **Info/P3**: Informational, no immediate action required

**SLO-Based Alerting [COMP018.1-slo-alerting]:**
- Alerts SHOULD be tied to Service Level Objectives (SLOs)
- Use error budget burn rate for alerting thresholds
- Fast burn alerts (1-2 hour windows) for immediate issues
- Slow burn alerts (24-72 hour windows) for trend issues

**Alert Threshold Guidelines:**
- Error rate: Alert when > 1% of requests fail (or based on SLO)
- Latency: Alert when p95 > SLO threshold
- Saturation: Alert when > 80% resource utilization
- Availability: Alert when uptime < SLO (e.g., 99.9%)

**Alert Fatigue Prevention:**
- Avoid alerting on predicted problems
- Use multi-window, multi-burn-rate alerting
- Regularly review and tune alert thresholds
- Remove alerts that don't result in action

### Service Level Objectives (SLO) [COMP019-slo]

**SLO Definition Requirements:**

Each production service MUST define:

1. **Service Level Indicators (SLIs):**
   - Availability: Percentage of successful requests
   - Latency: Request response time (p95, p99)
   - Throughput: Requests processed per second
   - Error rate: Percentage of failed requests

2. **Service Level Objectives (SLOs):**
   - Target values for each SLI
   - Measurement window (e.g., 30 days)
   - Example SLOs:
     - Availability: 99.9% (43.2 minutes downtime/month)
     - Latency: 95% of requests < 200ms
     - Error rate: < 0.1% of requests

3. **Error Budgets:**
   - Calculated from SLO (e.g., 0.1% = error budget)
   - Used to balance velocity vs. reliability
   - When budget exhausted, freeze feature releases and focus on reliability

**SLO Documentation:**
- SLOs MUST be documented in service documentation
- SLOs SHOULD be reviewed quarterly
- SLO violations MUST trigger post-incident reviews

### Resource requirements

[COMP020-resource-recommendations] As part of the Helm chart, applications MUST provide an estimate of CPU/Disk/Memory for a certain baseline, with resources required to operate the application

**Resource Specification Requirements:**

Applications MUST specify:

**Requests (Guaranteed Resources):**
- Minimum CPU (in millicores)
- Minimum Memory (in MB/GB)
- These represent guaranteed allocation

**Limits (Maximum Resources):**
- Maximum CPU (in millicores)
- Maximum Memory (in MB/GB)
- Prevents resource hogging

For Example:

For application **X** to be able to process **Y** transactions per minute, we recommend:

- Requests: 1000 milliCPU, 512MB RAM
- Limits: 2000 milliCPU, 1GB RAM
- 3 container replicas (minimum for high availability)
- File storage type and size requirements
- Network bandwidth requirements (if applicable)

**Autoscaling Configuration:**
- Horizontal Pod Autoscaler (HPA) thresholds
- Target CPU utilization (typically 70-80%)
- Minimum and maximum replica counts
- Scale-up and scale-down behavior

### Disaster Recovery [COMP021-disaster-recovery]

Based on Azure Well-Architected Framework:

**Recovery Objectives:**

Each application MUST define:

1. **Recovery Time Objective (RTO):**
   - Maximum acceptable downtime
   - Time to restore service after incident
   - Should align with business requirements

2. **Recovery Point Objective (RPO):**
   - Maximum acceptable data loss
   - Frequency of backups/replication
   - Should align with business requirements

**Backup and Restore:**
- Database backup strategy MUST be documented
- Backup frequency MUST meet RPO requirements
- Restore procedures MUST be tested quarterly
- Backup retention policy MUST be defined

**Disaster Recovery Testing:**
- DR procedures MUST be tested at least annually
- Chaos engineering tests SHOULD be performed regularly
- Failover and failback procedures MUST be documented
- DR test results MUST be documented

### DNS Naming Conventions

Applications MUST support access from *ANY* URL simultaneously. There should be no restiction on changing application URLS at any point in time. If a gemeente decide to change the URL that citizens access the application from, this should not require any database updates. 

Applications MUST be accessible via:

- technical domains: app.test.gemeente.dimpact.nl
- vanity name domains: burgerformulieren.gemeente.nl
- kubernetes internal DNS naming: openzaak.podiumd.svc.cluster.local
- relative path domain name: gemeente.nl/formulieren 

More information can be found here: https://dimpact.atlassian.net/wiki/spaces/PCP/pages/175865864/Toegang+API+s+via+verschillende+domeinnamen+waaronder+interne+toegang

## Deployment Verzoek

Deployment requests take the form of a "Deployment Verzoek", a form filled in with all relevant data required for deployment.

### Release notes

[COMP022-release-notes] The following information MUST be contained in the application release notes:

- Full description of parameters needed for HELM deploy.

- Full description of possible output parameters (eg API keys generated at deploy time)

- Dependencies on earlier versions of the application

- Storage Requirements

- Database requirements

- External access / Ingress requirements

- **Breaking changes** clearly marked and explained

- **Security fixes** with CVE numbers if applicable

- **Migration steps** if required for upgrade

- **Rollback procedures** in case of issues

- **Performance impact** of changes

- **Known issues** in this release

Release notes SHOULD follow the "Keep a Changelog" format:
- <https://keepachangelog.com/>

## Security Requirements [COMP023-security]

### Container Security

- Containers MUST run as non-root user
- Containers MUST use read-only root filesystem where possible
- Containers MUST drop all unnecessary Linux capabilities
- Container images MUST be scanned for vulnerabilities before deployment
- High/Critical CVEs MUST be addressed within 7 days

### Network Security

- Network policies MUST be defined to restrict pod-to-pod communication
- Only necessary ports SHOULD be exposed
- All external communication MUST use TLS 1.2 or higher
- API authentication and authorization MUST be implemented

### Secret Management

- Secrets MUST NEVER be committed to source control
- Secrets MUST be stored in Azure Key Vault
- Secrets SHOULD be rotated regularly (at least annually)
- Application SHOULD support secret rotation without restart where possible

### SBOM (Software Bill of Materials) [COMP024-sbom]

- Applications SHOULD provide SBOM in SPDX or CycloneDX format
- SBOM SHOULD include all direct and transitive dependencies
- SBOM SHOULD be updated with each release
- SBOM helps with vulnerability tracking and compliance

## Integration and API Best Practices [COMP025-api-practices]

### API Design Standards

**RESTful Principles:**
- Use standard HTTP methods (GET, POST, PUT, PATCH, DELETE)
- Use plural nouns for collections (`/users`, `/orders`)
- Use nested resources for relationships (`/users/{id}/orders`)
- Return appropriate HTTP status codes

**Versioning:**
- API versions MUST be included in URL path (`/api/v1/resource`)
- Major version changes MUST be backward incompatible only
- Old API versions SHOULD be supported for at least 12 months after deprecation

**Error Handling:**
- Errors MUST return consistent JSON format
- Include error code, message, and details
- Use appropriate HTTP status codes

**Rate Limiting:**
- APIs SHOULD implement rate limiting
- Rate limit headers SHOULD be returned
- Rate limits SHOULD be documented

**Documentation:**
- APIs MUST provide OpenAPI/Swagger documentation
- Documentation MUST be kept up-to-date
- Example requests and responses SHOULD be provided

### Service Mesh Considerations [COMP026-service-mesh]

For complex microservice environments:

- Service mesh (like Istio or Linkerd) MAY be used for:
  - Mutual TLS between services
  - Traffic management and routing
  - Observability and tracing
  - Circuit breaking and retry logic

- If service mesh is used:
  - Applications MUST NOT implement their own service discovery
  - Applications SHOULD rely on mesh for retry and timeout logic
  - Applications MUST expose metrics for mesh integration

## Documentation Requirements [COMP027-documentation]

### Technical Documentation

Each application MUST provide:

1. **Architecture Documentation:**
   - System architecture diagram
   - Component interactions
   - Data flow diagrams
   - External dependencies

2. **Deployment Documentation:**
   - Helm chart values explanation
   - Environment-specific configurations
   - Deployment sequence/dependencies
   - Rollback procedures

3. **Operations Documentation:**
   - Runbooks for common issues
   - Troubleshooting guides
   - Log interpretation guide
   - Metrics dashboard descriptions

4. **API Documentation:**
   - OpenAPI/Swagger specification
   - Authentication/authorization details
   - Rate limiting information
   - Example requests/responses

5. **Security Documentation:**
   - Security controls implemented
   - Authentication/authorization model
   - Data encryption details
   - Compliance requirements

### Documentation Maintenance

- Documentation MUST be updated with each release
- Documentation SHOULD be version-controlled
- Documentation SHOULD be accessible to all stakeholders
- Outdated documentation MUST be clearly marked

## Compliance and Audit [COMP028-compliance]

### Audit Logging

- All authentication attempts MUST be logged
- All authorization failures MUST be logged
- All data access SHOULD be logged (where applicable)
- Logs MUST be retained per regulatory requirements
- Logs MUST be tamper-proof

### Compliance Requirements

Applications handling personal data MUST:
- Comply with GDPR requirements
- Implement data retention policies
- Support right to erasure (right to be forgotten)
- Support data portability
- Maintain audit trails

## Performance Requirements [COMP029-performance]

### Response Time Targets

- Interactive requests: < 200ms (p95)
- API calls: < 500ms (p95)
- Batch operations: Document expected duration
- Database queries: < 100ms (p95)

### Throughput Requirements

- Applications MUST document expected throughput
- Load testing results SHOULD be provided
- Performance degradation under load MUST be documented

### Caching Strategy

- Applications SHOULD implement caching where appropriate
- Cache invalidation strategy MUST be documented
- Cache hit rate SHOULD be monitored

## Cost Optimization [COMP030-cost-optimization]

### Resource Efficiency

- Applications SHOULD use resources efficiently
- Unnecessary resource requests waste money
- Autoscaling SHOULD be used to match demand
- Idle resources SHOULD be scaled to minimum during off-hours

### Cost Visibility

- Resource requirements SHOULD be documented
- Cost per transaction/user SHOULD be estimated
- Optimization opportunities SHOULD be identified

## References

[Architecting Applications for Kubernetes | DigitalOcean](https://www.digitalocean.com/community/tutorials/architecting-applications-for-kubernetes)

[Free Guide: Our 15 Principles for Designing and Deploying Scalable Applications on Kubernetes (elastisys.com)](https://elastisys.com/designing-and-deploying-scalable-applications-on-kubernetes/)

<https://sysdig.com/blog/golden-signals-kubernetes/>

[Microsoft Well-Architected Framework for Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-kubernetes-service)

[Google SRE: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

[Google SRE: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)

[Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)

[Azure Best Practices for AKS](https://learn.microsoft.com/en-us/azure/aks/best-practices)

# TITLE HERE

| Version | Date | Author | Comments |  |
|----|----|----|----|----|
| 0.1 | 20/02/2024 | Jim Leitch | Sander vd B, Andrew M |  |
| 0.8 | 4/3/2024 | Jim Leitch | Stephan Z, Jesse H, Mahmut C, Petra C |  |
| 1.0 | 22/4/2024 | Jim Leitch | SSC/Maykin/Dimpact |  |
| 1.1 | 6/5/2024 | Jim Leitch |  |  |
| 1.2 | 29/5/2024 | Jim Leitch | Dimpact/ICATT | Updated “k8s operators” |
| 2024Q3 | 27/8/2024 | Jim Leitch | SSC | Converted to Markdown |

## Executive Summary

The PodiumD hosting environment runs a collection of Common
Ground-Compliant resources currently hosted on Microsoft Azure.

Development, Test, Acceptance and Production (OTAP) environments are
provided for multiple Dutch city councils (municipalities) to host the
PodiumD application, a replacement for the current E-suite workflow
system as used by many municipalities.

PodiumD is a collection of Open-Source components built by external
software developers funded by the municipalities.

This document is meant to formalize what SSC (the hosting provider)
expects from Dimpact (the contracting party) in terms of what Dimpact
in-turn expect from their contracted software developers.

This document describes how SSC expects that software developers should
package their software to be integrated into the SSC hosting environment
and describes what can be expected from us, SSC Hosting, the hosting
provider.

Rather than this document **pre**scribing how developers should deliver
their software, it is meant as a document that describes what SSC
Hosting and the developers expect from each other. It will develop over
time with input from all parties.

The document will be hosted in a GIT repository and will be updated with
versioning by means of Git Pull Requests.

## Document Conventions

We will use RFC-style terms to describe the level of requirements. As
time goes on various aspects of this contract will be made more and less
forceful: “**MUST**“/“**MUST** **NOT**“/“**SHOULD**“/“**SHOULD**
**NOT**“/“**MAY**“

([RFC 2119: Key words for use in RFCs to Indicate Requirement Levels
(rfc-editor.org)](https://www.rfc-editor.org/rfc/rfc2119))

## Multiple Environments Require Easy Deployments

It should be clear that applications being deployed will be deployed for
multiple municipalities in multiple OTAP environments. This means that
100% automated deployment is a hard requirement for any applications
being deployed.

## Infrastructure Description

### Subscriptions / Environments

Each customer/gemeente utilises a set of Azure Subscriptions. One
subscription for each of the OTAP environments. This simplifies billing
and provides security demarcation between the various environments, at
customers, suppliers and OTAP level.

Not every customer/supplier requires the full set of OTAP environments.

### Cloud Resource Provider

We currently use Microsoft’s Azure as our cloud platform. Our focus is
currently on Microsoft Azure; however, we MAY in the future to use
alternative cloud platforms. Any efforts made for deployment to our
cloud platform should be generic, using open standards, keeping in mind
that we may move to an alternative or even multi-cloud platform in the
future.

### Azure Resources

The resources currently in our Azure cloud environment, in use and
available are described below. In time we may migrate or simultaneously
deploy in an alternative cloud environment such as Google / AWS /
Private cloud.

Any new Azure resources required can be discussed going forward, this is
current list:

- **Kubernetes** (k8s) – AKS

  - We use standard AKS with two autoscaling groups of nodes, one for
    all application workloads, another for all management-type
    workloads.

- **Container Registry** – ACR

  - ACR with vulnerability scanning

- **SQL DBMS** - Azure Database for PostgreSQL flexible servers

  - PostgreSQL 14 is our standard database server, one instance per
    environment with multiple databases inside the server, one per
    application.

  - Database Timezone is UTC

- **Edge Load Balancer** - Azure Application Gateway

  - Azure Load Balancer is our internet-facing endpoint, providing SSL
    termination, HTTP -\> HTTPS redirection, URL rewriting and
    distribution of traffic to the appropriate K8S cluster. We have one
    load balancer for ONTW/TEST and another for ACCP/PROD.

  - We may in future choose to use our on-premises Palo Alto and GW
    services for this function

- **Storage** - Azure Storage Accounts

  - Azure storage accounts are used for provision of Blob storage and
    Azure File Shares (SMB and NFS)

- **Secret and Certificate Management** – Azure Key Vault

  - Azure Key Vault is the single source of truth for all certificates
    and secrets

- **Identity Management** - Active Directory / Entra ID

  - Azure Entra ID SHOULD be used as the only method of Identity
    Management. If another identity tool is used, a waiver must be
    obtained with a plan for future migration to Entra.

- **Deployment Orchestration -** Azure DevOps

  - Azure DevOps is our CICD orchestration tool, used for deployment of
    infrastructure, application deployment and other management tasks.
    We use a mixture of Terraform and bash scripting with az cli
    commands to build and maintain our infrastructure.

- **Infra Lifecycle**

  - Most Azure infrastructure resources (storage, databases) have very
    long lifecycles.

  - Kubernetes versions are handled frequently with the assistance of
    Azure Fleet Manager and will be updated on a regular monthly cycle
    ***ontw/test/accp/prod*** with one week between each environment
    type, meaning that any non-critical k8s patches will have been
    running in ONTW for 3 weeks before production is patched

  - Upgrades to other resources such as RDBMS and storage will be
    announced well in advance to allow suppliers to test their
    applications with the new versions and to work on and plan an
    upgrade path for the application if required.

## Azure Application Gateway

Azure application gateway can rewrite HTTP headers. At this moment we
apply only one header rewrite for HSTS purposes as follows:

We have not made any other changes to the AGW settings. For any other
default settings please refer to the MS Azure documentation, any changes
to default settings will be reflected in this document.

HTTP2 support is currently DISABLED in the AGW.

### Non – Azure Native Shared Resources

Some resources are not available as Azure native services and are
instead deployed inside the Kubernetes cluster:

### SSL Certificates (FrontEnd)

- We use SSL certificates from **LetsEncrypt**. The certificates are
  automatically requested and downloaded for every web facing service by
  the cert-manager application inside the cluster. Certificates and keys
  are then stored in the Azure Key Vault. They are then subsequently
  deployed to the listeners in the Application Gateway.

- SSL Certificates used internally between apps and legacy system are
  out-of-scope of this discussion.

### Cost management

In future we may be able to provide extra cost management tooling using
the billing data stored.

### Environments (OTAP)

Each municipality and organization has the possibility to run O,T,A
and/or P organizations, defined as follows:

#### DEVELOPMENT(ONTWIKKEL)

Ontwikkel (development) environments are meant to verify proper working
of the software in a production-similar environment. Developers are free
to deploy container versions and new helm charts in this environment.
Changes to Azure resources and the k8s cluster are performed by SCC
hosting in an updated version of the environment.

To save on cloud costs we switch all dev, test and acceptance
environment off in the evening and at weekends. If any individual
environments need to be kept on for any reason for a short period of
time, this can be accommodated.

#### Test

Test environments are meant for the municipalities to test new
application features and provide training facilities to staff.

#### Acceptance

Acceptance environments are 100% like-for-like with production
environments. Hardware resources are the same as production and copies
of production data.

#### Production

Production is of course the environment per customer/municipality that
has maximum uptime, monitoring, and resources.

## Deployment- Azure DevOps Pipelines

Deployment of applications to the k8s cluster are done by an Azure
DevOps Pipeline to deploy a Helm Chart. Any other infrastructure
preparation is done by Terraform files and/or bash scripts running “az
cli” commands.

## What We Expect

We expect that Dimpact will require suppliers to deliver the
applications in a way that allows for secure, scalable, stable,
manageable, and easily deployable systems.

### Naming Conventions [COMP001-naming]

All application and parameter naming should be uniform to allow for a
minimum of repeated configuration. Dimpact will provide examples of
parameter naming to be used in the umbrella helm chart.

### Versioning [COMP002-versioning]

PodiumD consists of a suite of sub-applications. The PodiumD version is
defined by the versions of the sub-applications.

### Containers

- [COMP003.1-app-as-containers] Applications MUST be delivered as one or a set of application
  containers

- [COMP003.3-container-registry]These containers MUST be available via a public Container Registry
  such as **Docker Hub**

- Non-open-source containers MAY be available secured by a key/token.

- [COMP003.3-semantic-versioning] Container labels MUST be “semantically versioned.”

- Prelease versions MAY be deployed with version + keyword (e.g.,
  1.2.3-snapshot)

SSC hosting will copy containers from the supplier’s own container
registry to a Container Registry in the SSC Azure environment.

### Timezone [COMP004-timezone]

For logging and monitoring purposes, the timezone for applications
should be **UTC**. The default timezone for the database is UTC, all
logging and metrics should be set to provide UTC output. Of course any
end-user interaction should be presented to the client in the
appropriate local timezone.

### Software Supply Chain [COMP005-software-supply-chain]
SSC Hosting will scan all containers for vulnerabilities on a regular
basis. Containers delivered MUST be already scanned for vulnerabilities
by the supplier. Any vulnerabilities that are flagged MUST be fixed,
mitigated and/or receive a waiver from all system architects of
supplier + Dimpact + SSC Hosting within a specified time limit.

An example of code-level scanning would be SonarQube:

<https://www.sonarsource.com/products/sonarqube/>

An example of container scanning would be GitHub container scanning:

<https://github.com/marketplace/actions/container-scan>

### Scalability and Startup/Shutdown

Apart from where inappropriate, applications SHOULD gracefully handle
load-based up and down-scaling of containers.

[COMP006.1-stop-start-safe] Application MUST be able to withstand sudden stops and starts (for
example a node failing or container-node migration event)

Applications MAY be deployed with init containers, startup and shutdown
scripts.

### Helm Charts . Deployment

- Dimpact will curate and provide all required Helm charts in one single
  repository

- [COMP007.1-helm] All applications MUST be accompanied by Helm Charts. Applications MAY
  use helm sub-charts and MUST be able to be deployed as a sub-chart
  themselves to facilitate and contribute to a complete “umbrella”
  installation of the PodiumD platform.

- [COMP007.2-operators] We do not support the use of applications and components deployed by
  means of Kubernetes Operators **unless** the installation and use of
  the operators can be encapsulated inside a single Umbrella Helm chart.

- [COMP007.3-dependancy-apps] Any dependency-applications MUST be deployed as Helm sub-charts.

- The application itself MUST be deployable as a sub chart as part of an
  over-encompassing PodiumD Helm chart. Over time all parties should
  work to harmonize variable naming in the Helm charts

- Deployments SHOULD be performed as a rolling update – unless waiver
  has been granted. Deployments and upgrades that involve database
  changes are currently exempt from this requirement.

- Helm charts SHOULD be available via the same artifacts’ repository as
  the containers

- [COMP007.4-storage] Helm chart MUST support storage type via SSC’s own storage choices (if file storage is required)

- [COMP007.5-ingress] Helm chart MUST support ingress that allows for SSC’s own ingress
  choices (if ingress in required)

- [COMP007.5-database] Helm chart MUST support external cloud database usage (if database is required)

### K8s Namespaces

PodiumD as a whole is deployed in one single "podiumd" kubernetes namespace.

~~SSC hosting uses one single k8s namespace per recognizable (openforms,
objects etc.) component. Components must be deployed into its own
namespace in Kubernetes cluster.~~

~~The name of the namespace will always be the same as the name of the
application as defined in the helm chart.~~

### 100% Automation

[COMP008.1-100percent-automated] PodiumD software will be deployed to many customers and environments,
for this reason, all components of the application MUST be possible to
deploy completely from the Helm chart.

In this case – 100% deployment means the containers running correctly
inside the K8s cluster with the correct deployment parameters and
secrets, connecting to other resources such as databases and other
PodiumD applications.

Final configuration “inside” of the application is out of this scope.

To help understanding of each area of deployment, development party
Maykin Media produced this useful set of definitions:

> **• OTAP-omgevingen**
>
> Ontwikkel-, Test-, Acceptatie, Productie-omgevingen
>
> **• Platform**
>
> Set aan componenten die in samenhang een oplossing bieden (bijv. Open
> Zaak en ZAC)
>
> **• Installatie**
>
> Het deployen van een component (bijv. Helm chart uitvoeren zodat Open
> Zaak en ZAC technisch draaien)
>
> **• Configuratie**
>
> Parameters zodat de component kan doen wat het moet doen (bijv. de
> koppeling tussen ZAC en Open Zaak)
>
> **• Inrichting**
>
> Functionele vulling van het component evt. met hele specifieke
> configuratie (bijv. zaaktypen, processen)

#### Application Addressing

Dimpact will provide application URLS for all public facing
applications.

Dimpact will liase with gemeentes as to the application addressing used
and for crestion of appropriate hostnames and cnames.

### Communication Between Components

[COMP009-internal-communication] Communication between components inside the same k8s cluster SHOULD take
place inside the cluster. Communication and data transfer between
components SHOULD take place via the APIs.

### Database Migrations

[COMP010-sql-migration-scripts] Applications MUST have all database migrations built in or provide
config scripts for a DB migration tool such as Flyway or Liquibase.

### Application concurrency / Statelessness

- Applications SHOULD be stateless and commit all data to a database.

- Files SHOULD be written to an object store such as Azure Blob Storage

- Files MAY be written to cloud file shares.

- [COMP011-no-local-storage] Persistent files containing any data MUST NOT be written to local
  storage on the container but instead to a kubernetes PV.

- [COMP011-permanent-data] Permanent data that requires to be backed up must be written to a file share created outside of the K8s cluster

### FILES

Files MAY be stored in the following storage types, in order of
preference:

- Object Storage

- SMB file share shared volume file storage

- Persistent volume storage

### Configuration / Secrets

- All application configurations SHOULD be performed via environment
  variables.

- Environment variables SHOULD be set via config maps in the helm chart.

- [COMP012-secrets] Secret/sensitive information MUST be written to the K8s secrets store.

- Secrets will be sourced and read from the Azure Key Vault at
  deployment-time.

- Application logging verbosity SHOULD be enabled by setting a debug
  environment variable and restarting the container.

### SSL Certificates

[COMP013-ssl] SSLApplication end points MUST be presented as HTTP, unencrypted. Our
Public-facing Azure Application Gateway performs SSL termination and
guarantees a secure path to the k8s cluster via the Azure VWAN and
Virtual Hub facilities. The Azure Application Gateway listeners are
populated by Lets Encrypt SSL certificates at deploy-time

### Probes

[COMP014-healthchecks] Applications MUST ensure availability of probe endpoints to allow for
correct handling of starting, running and failing containers.

See [Configure Liveness, Readiness and Startup Probes \|
Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

## Logging / Monitoring / Metrics / Alerting

### Monitoring

Monitoring is the combination of aggregating logs and metrics to
ascertain if the application is working as expected, hopefully before
the end user is affected and allow us to react in time to fix the
problem and/or underlying issue.

### Infra monitoring

The infrastructure (resources such as k8s, edge gateway, databases etc.)
SHALL be monitored for errors as exported to Azure. Other metrics such
as CPU capacity, disk IO should be used to ensure rightsizing and
autoscaling of resources.

### Logging

- [COMP015.1-logging-to-stdout] Applications MUST stream all logging to STDOUT. These logs will be
  captured and sent to the appropriate logging endpoint.

- Application logging MUST be configurable via a LOGLEVEL environment
  variable following standard usage for DEBUG, INFO, WARN, ERROR, and
  FATAL log levels.

- Applications logs SHOULD be written in JSON format to allow proper
  parsing in the Logging system. (see
  <https://betterstack.com/community/guides/logging/json-logging/>)

- Applications MAY write relevant logging information in a form that can
  be read in the application, but this MUST be in addition to the logs
  written to STDOUT.

- [COMP016.2-utc-logs] Log lines MUST be written with timestamps in UTC format, no time zone.

### Metrics

Applications SHOULD provide the metrics that make sense for the
application in hand for example:

- Number of Transactions

- Transaction latency

- Incomplete transactions

- Failed transactions

- Any other relevant metrics per application

- The metrics can then be scraped by (for example) Prometheus to display
  in graph form, for analysis and for alerting purposes.

Metrics are an ongoing discussion and will always be updated and
improved throughout the lifecycle of the application.

The following article gives a thorough discussion on the use of metrics
usage in Kubernetes:

<https://sysdig.com/blog/golden-signals-kubernetes/>

### Resource requirements

[COMP017-resource-recommendations] As part of the Helm chart, applications MUST provide an estimate of
CPU/Disk/Memory for a certain baseline, with resources required to
operate the application

For Example:

For application **X** to be able to process **Y** transactions per
minute, we recommend:

- 2000 milliCPU

- 1GB RAM

- 3 container replicas

- File storage type

### DNS Naming Conventions

Applications MUST support access from *ANY* URL simultaneously. There should be no restiction on changing application URLS at any point in time. If a gemeente decide to change the URL that citizens access the application from, this should not require any database updates. 

Applications MUST be accessible via:

- technical domains: app.test.gemeente.dimpact.nl
- vanity name domains: burgerformulieren.gemeente.nl
- kubernetes internal DNS naming: openzaak.podiumd.svc.cluster.local
- relative path domain name: gemeente.nl/formulieren 

More infomration cna be found here: https://dimpact.atlassian.net/wiki/spaces/PCP/pages/175865864/Toegang+API+s+via+verschillende+domeinnamen+waaronder+interne+toegang


## Deployment Verzoek

Deployment requests take the form of a “Deployment Verzoek”, a form
filled in with all relevant data required for deployment.

### Release notes

[COMP018-release-notes] The following information MUST be contained in the application release
notes:

- Full description of parameters needed for HELM deploy.

- Full description of possible output parameters (eg API keys generated
  at deploy time)

- Dependancies on earlier versions of the application

- Storage Requirements

- Database requirements

- External access / Ingress requirements

## References

[Architecting Applications for Kubernetes \|
DigitalOcean](https://www.digitalocean.com/community/tutorials/architecting-applications-for-kubernetes)

[Free Guide: Our 15 Principles for Designing and Deploying Scalable
Applications on Kubernetes
(elastisys.com)](https://elastisys.com/designing-and-deploying-scalable-applications-on-kubernetes/)

<https://sysdig.com/blog/golden-signals-kubernetes/>

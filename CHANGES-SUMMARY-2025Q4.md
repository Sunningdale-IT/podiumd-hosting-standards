# Summary of Changes: Hosting Contract 2025Q4

## Document Information
- **Source Document**: hosting-contract-2024Q3.md
- **Target Document**: hosting-contract-2025Q4.md
- **Update Date**: October 27, 2024
- **Primary References**: 
  - Capgemini PodiumD Assessment Report (cap-gemini-nl.txt)
  - Microsoft Azure Well-Architected Framework
  - Google SRE Documentation
  - Industry Best Practices for Kubernetes Hosting

## Executive Summary

This update transforms the hosting contract from a basic infrastructure requirements document into a comprehensive SRE/DevOps best practices guide. The changes incorporate recommendations from the Capgemini assessment (which scored the current reliability at 41/200) and align with industry standards from Microsoft and Google.

## Major Additions and Changes

### 1. Service Naming Conventions [COMP001-naming] - ENHANCED

**Why**: The Capgemini report emphasized the need for consistency in multi-tenant environments. Industry best practices require DNS-compliant naming.

**What Changed**:
- Added explicit DNS label standards (RFC 1123 compliance)
- Defined service naming patterns for consistency
- Added Kubernetes DNS naming format documentation: `{service-name}.{namespace}.svc.cluster.local`
- Included multi-tenant naming considerations
- Added labeling requirements for tenant tracking

**Impact**: Ensures consistency across all components, enables automation, and prevents naming collisions in multi-tenant deployments.

### 2. Deployment Time Requirements [COMP007.7-deployment-time] - NEW

**Why**: The issue specifically requested that upgrades should never last more than 15 minutes. This aligns with SRE best practices for minimal disruption windows.

**What Added**:
- Hard requirement: Component upgrades MUST complete within 15 minutes
- Breakdown of what's included in the 15-minute window:
  - Container pull time
  - Application startup time
  - Readiness probe success
  - Traffic migration
- Requirement for architectural review if consistently exceeded

**Impact**: Sets clear expectations for deployment performance, enabling better capacity planning and ensuring minimal service disruption.

### 3. Zero-Downtime Deployment Strategy [COMP007.6-rolling-updates] - ENHANCED

**Why**: Modern Kubernetes applications must support continuous availability. The previous contract allowed waivers; the new standard makes this mandatory.

**What Changed**:
- Changed from "SHOULD be performed as rolling update" to "MUST be performed as rolling update with zero downtime"
- Added specific configuration: `maxUnavailable: 0` and `maxSurge: 1` or higher
- Removed waiver clause for database changes (these must now also be zero-downtime)

**Impact**: Eliminates planned downtime for deployments, improving overall system availability.

### 4. Version Information Endpoint [COMP019-version-endpoint] - NEW

**Why**: The issue specifically requested "each app should have an API where the version of the app/component can be easily read out." This is essential for operational visibility and troubleshooting.

**What Added**:
- Mandatory `/version` or `/api/version` endpoint
- Structured JSON response format with:
  - Semantic version number
  - Build/commit information
  - Build timestamp
  - Optional dependency versions
- Requirement for unauthenticated access (operational tool)
- Clear separation: version endpoint is NOT for health checks

**Impact**: Enables automated version discovery, improves troubleshooting, and supports compliance tracking across deployments.

### 5. Health State Definitions [COMP014-healthchecks] - SIGNIFICANTLY ENHANCED

**Why**: This directly addresses the Capgemini report's highest priority recommendation (score 41/200 for reliability). The report emphasized: "We must clearly define and document the different operational states of our systems."

**What Added**:
- Five operational states defined based on Azure Well-Architected Framework:
  1. **Healthy/Normal State** - Full operation
  2. **Degraded State** - Reduced functionality but operational
  3. **Failed/Unhealthy State** - Cannot function correctly
  4. **Maintenance State** - Planned downtime
  5. **Recovery State** - Returning to normal after incident

- Enhanced probe configuration requirements:
  - Liveness vs Readiness probe distinctions clarified
  - What each probe should/shouldn't check
  - When failures should trigger restarts vs traffic removal
  - Startup probe guidance for slow-starting apps

- Health check endpoint standards:
  - Lightweight and fast (< 1 second)
  - Structured JSON responses with status and dependency checks
  - Example response format provided

**Impact**: Dramatically improves monitoring accuracy, enables automated responses, and provides clarity for incident management. This directly addresses the Capgemini report's primary concern.

### 6. Service Level Objectives (SLOs) and Indicators (SLIs) [COMP020-slo-sli] - NEW

**Why**: Google SRE best practices emphasize SLOs as fundamental to reliability engineering. The issue requested alignment with "Google SRE documentation."

**What Added**:
- Definition and requirement for SLIs (measurable service health):
  - Availability (99.9%+ target)
  - Latency (95th percentile thresholds)
  - Error Rate (< 0.1% target)
  - Throughput capacity

- SLO documentation requirements:
  - Explicit targets for critical user journeys
  - Error budget concepts
  - Balance between reliability and feature velocity

- Example SLO documentation format provided

- Monitoring requirements:
  - Prometheus format metrics
  - Request counters, latency histograms, error counters
  - Proper labeling (endpoint, method, status)

**Impact**: Provides measurable reliability targets, enables data-driven decision making, and aligns with industry-leading SRE practices.

### 7. Redundancy and Fault Tolerance [COMP021-redundancy] - NEW

**Why**: Capgemini report recommendation #3 emphasized implementing redundancy to meet reliability requirements.

**What Added**:
- Multi-zone deployment requirements:
  - Distribution across Azure Availability Zones
  - Pod anti-affinity rules
  - Minimum 3 replicas for critical services

- Retry mechanisms:
  - Exponential backoff for transient failures
  - Appropriate timeout values
  - Cascading failure prevention

- Circuit breaker pattern:
  - For external service calls
  - Graceful degradation
  - Prevent unnecessary delays

- Load balancing requirements:
  - Stateless design for effective distribution
  - Minimize session affinity

**Impact**: Increases system resilience, reduces outage impact, and ensures business continuity.

### 8. Alerting and Critical Thresholds [COMP022-alerting] - NEW

**Why**: Capgemini report recommendation #4 emphasized setting alerts for critical thresholds to proactively address problems.

**What Added**:
- Resource usage alert thresholds:
  - CPU > 80% for 5 minutes
  - Memory > 85% for 5 minutes
  - Disk > 80%

- Performance alerts:
  - SLO threshold violations
  - Latency and error rate alerts
  - Availability monitoring

- Application-specific alerts:
  - Queue depth
  - Connection pool exhaustion
  - Dependency failures
  - Background job failures

- Alert best practices:
  - Must be actionable
  - Include context
  - Avoid alert fatigue

**Impact**: Enables proactive problem detection, reduces mean time to detection (MTTD), and improves operational response.

### 9. Disaster Recovery [COMP023-disaster-recovery] - NEW

**Why**: Capgemini report recommendation #5 emphasized planning and executing regular disaster recovery exercises.

**What Added**:
- Backup requirements:
  - Document data backup needs
  - Define RPO (Recovery Point Objective)
  - Define RTO (Recovery Time Objective)

- Recovery procedures:
  - Step-by-step documentation
  - Quarterly testing requirement
  - Runbooks for common failures

- Chaos engineering:
  - Test failure scenarios
  - Non-production testing
  - Validate fault tolerance

**Impact**: Ensures preparedness for disasters, reduces recovery time, and validates resilience mechanisms.

### 10. Infrastructure Enhancements

**Why**: Align with Microsoft Azure Well-Architected Framework recommendations for high availability.

**What Added**:
- Availability Zones requirement for AKS clusters
- Multi-zone deployment emphasis throughout document
- Regional redundancy considerations

**Impact**: Improves infrastructure resilience and reduces blast radius of failures.

### 11. Resource Requirements Enhancement [COMP017-resource-recommendations]

**Why**: Proper resource allocation is essential for stability and cost optimization.

**What Enhanced**:
- Added requirement for both `requests` and `limits`
- Guideline: limits should be 1.5-2x requests for burst capacity
- Requirement for different load profiles (low, medium, high)

**Impact**: Better resource planning, improved cost management, and prevention of resource contention.

### 12. Release Notes Enhancement [COMP018-release-notes]

**Why**: Operational teams need comprehensive information for safe deployments.

**What Added to Required Information**:
- Breaking changes and migration procedures
- Performance impact of changes
- Security fixes included in release

**Impact**: Reduces deployment risks and improves change management.

### 13. Well-Architected Framework Alignment Section - NEW

**Why**: The issue specifically requested "make recommendations according to microsoft well architected framework standards."

**What Added**:
- New section mapping requirements to the five WAF pillars:
  1. **Reliability**: Multi-zone, health states, SLOs, disaster recovery
  2. **Security**: Scanning, secrets, network isolation, identity
  3. **Cost Optimization**: Right-sizing, quotas, scheduling
  4. **Operational Excellence**: Automation, logging, alerting, versioning
  5. **Performance Efficiency**: Scaling, resources, monitoring, load balancing

**Impact**: Demonstrates comprehensive alignment with Microsoft's cloud architecture standards and provides framework for continuous improvement.

### 14. Enhanced References Section

**Why**: Provide suppliers with authoritative sources for best practices.

**What Added**:
- Google SRE Book links (Service Level Objectives)
- Google SRE Workbook (Implementing SLOs)
- Microsoft Azure Well-Architected Framework
- Azure AKS Best Practices
- Kubernetes DNS documentation

**Impact**: Enables suppliers to deep-dive into topics and understand the rationale behind requirements.

## Changes by Compliance Code

### New Compliance Codes
- **COMP019-version-endpoint**: Version information API requirement
- **COMP020-slo-sli**: Service Level Objectives and Indicators
- **COMP021-redundancy**: Redundancy and fault tolerance requirements
- **COMP022-alerting**: Alerting and critical thresholds
- **COMP023-disaster-recovery**: Disaster recovery procedures

### Enhanced Compliance Codes
- **COMP001-naming**: Added DNS standards, Kubernetes naming, multi-tenant guidance
- **COMP007.6-rolling-updates**: Changed from SHOULD to MUST, added specific configuration
- **COMP007.7-deployment-time**: New requirement for 15-minute deployment window
- **COMP014-healthchecks**: Comprehensive health state definitions and probe requirements
- **COMP017-resource-recommendations**: Added requests/limits guidance and load profiles
- **COMP018-release-notes**: Added breaking changes, performance impact, security fixes

## Integration of Capgemini Recommendations

The Capgemini report (cap-gemini-nl.txt) provided a reliability assessment score of **41/200** and identified five critical areas for improvement. Here's how each was addressed:

### Capgemini Priority #1: Define System States (Score: 41/200)
**Addressed by**: COMP014-healthchecks enhancement
- Added five operational states (Healthy, Degraded, Failed, Maintenance, Recovery)
- Defined probe configuration requirements
- Provided health check endpoint standards
**Target**: Improve score to minimum 120/200

### Capgemini Priority #2: Configure Health Probes
**Addressed by**: COMP014-healthchecks enhancement
- Liveness vs Readiness probe distinctions
- Startup probe guidance
- Dependency checking recommendations

### Capgemini Priority #3: Redundancy and Fault Tolerance
**Addressed by**: COMP021-redundancy (NEW)
- Multi-zone deployment requirements
- Retry mechanisms
- Circuit breaker patterns
- Load balancing requirements

### Capgemini Priority #4: Alerts at Critical Thresholds
**Addressed by**: COMP022-alerting (NEW)
- Resource usage thresholds
- Performance alerts
- Application-specific alerts
- Alert best practices

### Capgemini Priority #5: Disaster Recovery Exercises
**Addressed by**: COMP023-disaster-recovery (NEW)
- Backup requirements with RPO/RTO
- Recovery procedure documentation
- Quarterly testing requirement
- Chaos engineering recommendations

## Benefits of the Updates

### For SSC Hosting (Platform Team)
1. **Improved Reliability**: Clear standards enable better monitoring and faster incident response
2. **Automation**: Consistent naming and versioning enable automated tooling
3. **Operational Visibility**: Version endpoints and structured health checks improve troubleshooting
4. **Risk Reduction**: Disaster recovery and redundancy requirements reduce outage impact

### For Suppliers (Development Teams)
1. **Clear Expectations**: Specific requirements reduce ambiguity
2. **Best Practices**: Alignment with industry standards (Microsoft, Google) provides proven patterns
3. **Better Support**: Well-defined SLOs and health states make support more effective
4. **Quality Improvement**: Security scanning and testing requirements improve overall quality

### For Municipalities (End Users)
1. **Higher Availability**: Zero-downtime deployments and redundancy reduce service interruptions
2. **Faster Recovery**: Defined recovery procedures minimize downtime duration
3. **Better Performance**: SLO monitoring ensures consistent service quality
4. **Transparency**: Version information enables tracking of deployed versions

## Migration Path from 2024Q3 to 2025Q4

### Immediate Requirements (MUST implement)
1. Add `/version` endpoint to all applications
2. Implement zero-downtime rolling updates
3. Define and document SLOs for critical services
4. Ensure 15-minute deployment window compliance

### Short-term Requirements (3-6 months)
1. Enhance health checks with state definitions
2. Implement retry mechanisms and circuit breakers
3. Deploy across multiple availability zones
4. Configure alerting on critical thresholds

### Medium-term Requirements (6-12 months)
1. Document and test disaster recovery procedures
2. Implement comprehensive SLI monitoring
3. Conduct chaos engineering tests
4. Achieve Well-Architected Framework compliance

## Compliance Matrix Impact

The updates affect the compliance tracking in app-compliance-2024Q3.md:

### Enhanced Requirements
- COMP001-naming: Now requires DNS compliance check
- COMP014-healthchecks: Requires state definition documentation
- COMP017-resource-recommendations: Requires requests/limits specification

### New Requirements
- COMP019-version-endpoint: New compliance check needed
- COMP020-slo-sli: New compliance check needed
- COMP021-redundancy: New compliance check needed
- COMP022-alerting: New compliance check needed
- COMP023-disaster-recovery: New compliance check needed

## Conclusion

This update represents a significant maturation of the PodiumD hosting standards, transforming from basic infrastructure requirements to comprehensive SRE/DevOps best practices. The changes directly address the Capgemini assessment findings, particularly the critical need for health state definitions (improving from 41/200 to target 120/200).

By incorporating Microsoft Azure Well-Architected Framework and Google SRE principles, the document now provides suppliers with industry-standard guidance that will result in more reliable, scalable, and maintainable systems.

The phased migration path allows existing components to adopt new requirements progressively while ensuring all new components meet the enhanced standards from the start.

### Key Metrics for Success
- **Reliability Score**: Target improvement from 41/200 to 120/200 (Capgemini assessment)
- **Deployment Time**: All upgrades complete within 15 minutes
- **Availability**: 99.9% or higher per SLO definitions
- **Recovery Time**: Documented and tested RTO/RPO for all critical services
- **Zero Downtime**: 100% of deployments executed without service interruption

This update positions PodiumD as a modern, well-architected platform that meets the highest standards of reliability, security, and operational excellence.

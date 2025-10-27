# Summary of Changes: Cap Gemini Review Integration

**Document:** hosting-contract-2025Q4.md  
**Date:** October 27, 2025  
**Review Source:** Bijlage3_translation_EN.txt (Cap Gemini Review)

## Overview

This document summarizes the changes made to the hosting-contract-2025Q4.md based on recommendations from the Cap Gemini review. The review provided valuable insights into security, operational excellence, and best practices for the PodiumD hosting environment.

## Major Changes

### 1. Cloud Resource Provider Strategy (Enhanced)

**What Changed:**
- Strengthened vendor lock-in avoidance requirements
- Made exit strategy mandatory for all managed services
- Added explicit requirements for portable Infrastructure as Code

**Why:**
- Ensures municipality can migrate to alternative cloud platforms at any time
- Reduces dependency on Azure-specific services
- Aligns with open standards philosophy

**Key Requirements Added:**
- Any managed service MUST have documented exit plan
- Export/backup formats must be defined
- Open protocol support must be verified

### 2. Open Source Infrastructure Tools (New Section)

**What Changed:**
- Added new section documenting required open-source tooling
- Specified version pinning and controlled upgrade processes
- Documented vulnerability scanning requirements for infrastructure

**Why:**
- Transparency and auditability of platform components
- Easier migration between Kubernetes versions
- Reduces vendor dependency

**Components Documented:**
- Ingress controllers (NGINX)
- GitOps operators (ArgoCD, Flux)
- Monitoring stack (Prometheus, Grafana)
- Logging stack
- Policy engines (OPA)

### 3. Documentation Platform Requirements (New Section)

**What Changed:**
- Added COMP025 compliance code for documentation platform
- Mandated internal storage for sensitive documentation
- Prohibited public repository storage of secrets

**Why:**
- Protects sensitive configuration details
- Ensures proper access control
- Prevents accidental exposure of security-critical information

**Key Requirements:**
- Documentation in secured wiki or Git repository
- Sensitive data (IPs, keys, topologies) stored only internally
- Role-based access control required

### 4. Software Placement Criteria (New Section)

**What Changed:**
- Added COMP026 compliance code
- Defined explicit criteria for permitted software
- Listed prohibited software categories

**Why:**
- Clarity on what software can be hosted on Haven
- Enforces Common Ground principles
- Prevents security and compatibility issues

**Categories:**
- **Permitted:** Common Ground-compliant components, aligned open source
- **Prohibited:** Closed-source black boxes, vendor lock-in risks, EOL software

### 5. Enhanced Integration and Connectivity

**What Changed:**
- Expanded COMP009 with additional sub-requirements
- Added requirements for secure channels and mutual authentication
- Specified central policy enforcement at ingress layer
- Explicitly prohibited direct database access

**Why:**
- Stronger security posture
- Better audit capabilities
- Prevents tight coupling between components

**Key Additions:**
- COMP009.4: Secure, mutually authenticated channels
- COMP009.5: Central ingress policies (rate limiting, WAF, auditing)
- COMP009.6: Prohibition of direct cross-component database access

### 6. User Access and Authentication (New Section)

**What Changed:**
- Added COMP027 compliance code
- Mandated centralized identity management
- Prohibited ad-hoc authentication implementations
- Specified deployment strategies (blue/green, canary)

**Why:**
- Consistent authentication across all components
- Better security through centralized identity provider
- Risk-appropriate deployment approaches

**Key Requirements:**
- SSO/federated identity required
- Azure Entra ID integration preferred
- No custom authentication without waiver
- Blue/green and canary deployment strategies documented

### 7. Enhanced Container Image Security

**What Changed:**
- Added COMP003.7 for immutable digest requirements
- Strengthened minimal image requirements
- Added base image refresh requirements

**Why:**
- Prevents tag mutation attacks
- Ensures exact reproducibility
- Reduces attack surface
- Maintains security patch levels

**Key Additions:**
- SHA256 digest references in production
- Regular base image updates
- Enhanced multi-stage build requirements

### 8. Namespace Isolation and Network Policies (Enhanced)

**What Changed:**
- Added COMP028 compliance code with detailed sub-requirements
- Added resource quota examples
- Added network policy examples
- Documented isolation requirements within shared namespace

**Why:**
- Prevents resource exhaustion
- Limits lateral movement in security incidents
- Provides defense in depth

**Key Requirements:**
- COMP028: Namespace isolation
- COMP028.1: Resource quotas (CPU, memory, storage, PVC limits)
- COMP028.2: Network policies (default deny, explicit allow)

### 9. Enhanced External Communication

**What Changed:**
- Expanded COMP020 with more specific requirements
- Added COMP020.4 for municipal environment communication
- Added explicit network whitelisting requirements
- Added privacy impact assessment requirements

**Why:**
- Better control over external dependencies
- Clearer security assessment process
- Documented data flows for compliance

**Key Additions:**
- Explicit network whitelisting for third-party services
- Privacy assessment for cross-environment data flows
- No cascading failures from external service issues

### 10. Enhanced Software Supply Chain Security

**What Changed:**
- Expanded COMP005 with additional sub-requirements
- Added build pipeline security requirements (COMP005.7)
- Added dependency integrity checks (COMP005.6)
- Made SBOM machine-readable requirement explicit

**Why:**
- Prevents supply chain attacks
- Ensures reproducible builds
- Protects against dependency confusion

**Key Additions:**
- COMP005.6: Checksum verification, signature verification, locked versions
- COMP005.7: Isolated build runners, secrets management, audit logging

### 11. Enhanced Concurrency and Statelessness

**What Changed:**
- Added COMP011.7 for concurrency safety
- Explicit documentation of state handling required
- Added requirements for transaction isolation

**Why:**
- Prevents race conditions and data corruption
- Ensures safe horizontal scaling
- Clarifies state management responsibilities

**Key Requirements:**
- No race conditions causing data corruption
- Proper locking for shared resources
- Transaction isolation where needed

### 12. Enhanced Configuration and Secrets

**What Changed:**
- Added COMP012.6 for secret scanning
- Enhanced secret rotation requirements
- Added encryption at rest requirement for Kubernetes Secrets

**Why:**
- Prevents accidental secret commits
- Supports dynamic credential updates
- Ensures secrets are protected even in etcd

**Key Additions:**
- Image scanning for accidentally committed secrets
- .gitignore and .dockerignore usage
- Watch for secret changes and reload dynamically

### 13. Enhanced GitOps Workflow

**What Changed:**
- Restructured section with clearer bullet points
- Added CI/CD validation requirement
- Added 24-hour commit requirement for emergency hotfixes
- Emphasized Git as single source of truth

**Why:**
- Stronger governance
- Better auditability
- Prevents configuration drift

**Key Requirements:**
- Automated validation before merge
- Emergency changes must be committed within 24 hours
- Drift detection and remediation

### 14. Enhanced Access Control

**What Changed:**
- Restructured with clearer organization
- Added audit logging requirement
- Emphasized least-privilege RBAC

**Why:**
- Better security posture
- Compliance with audit requirements
- Clear separation of responsibilities

**Key Additions:**
- All access and changes logged
- Namespace-scoped permissions detailed
- System-wide changes managed by SSC only

### 15. Enhanced Quality and Maintainability

**What Changed:**
- Expanded COMP024.1 with more specific requirements
- Added security testing to COMP024.2
- Enhanced documentation requirements in COMP024.3

**Why:**
- Higher code quality standards
- Better security posture
- More comprehensive documentation

**Key Additions:**
- Dependencies kept current requirement
- Security tests (SAST/DAST)
- API documentation requirement (OpenAPI/Swagger)

### 16. General Principles (New Section)

**What Changed:**
- Added comprehensive General Principles section before compliance codes
- Organized into 5 key areas with specific principles

**Why:**
- Provides philosophical foundation for all requirements
- Guides decision-making in edge cases
- Aligns all stakeholders on core values

**Principles Added:**

1. **Security and Privacy by Design:**
   - Least Privilege Access
   - Defense in Depth
   - Privacy by Default
   - Zero Trust

2. **Operational Excellence:**
   - Auditability
   - Reproducible Builds
   - Infrastructure as Code
   - Automated Testing

3. **Open Standards and Portability:**
   - Open Standards First
   - Cloud Agnostic
   - Interoperability
   - Open Source Preference

4. **Reliability and Resilience:**
   - Graceful Degradation
   - Automated Recovery
   - Chaos Engineering
   - Observability

5. **Continuous Improvement:**
   - Feedback Loops
   - Metrics-Driven
   - Learning Culture
   - Innovation

### 17. Updated Compliance Codes Table

**What Changed:**
- Added 4 new compliance codes to summary table
- Maintained consistent formatting

**New Codes:**
- COMP025-documentation-platform: Internal documentation storage
- COMP026-software-criteria: Software placement criteria
- COMP027-user-access: Centralized authentication and deployment strategies
- COMP028-namespace-isolation: Namespace resource quotas and network policies

## Impact Analysis

### Benefits

1. **Stronger Security Posture:**
   - Enhanced supply chain security
   - Better secret management
   - Network isolation and policies
   - Centralized authentication

2. **Better Vendor Independence:**
   - Explicit exit strategies
   - Open source tooling emphasis
   - Cloud-agnostic requirements

3. **Improved Operational Excellence:**
   - Clearer governance
   - Better auditability
   - Reproducible builds
   - Automated testing

4. **Enhanced Compliance:**
   - Documentation requirements
   - Privacy considerations
   - Audit logging

5. **Clearer Expectations:**
   - Software placement criteria
   - Deployment strategies
   - Quality standards

### Compatibility

- All changes are **additive** or **clarifying**
- No breaking changes to existing requirements
- Enhanced existing requirements with more specific guidance
- New requirements align with industry best practices

### Migration Path

For existing components:
1. Review against new COMP025-028 requirements
2. Implement namespace isolation controls (COMP028)
3. Add image digest references (COMP003.7)
4. Implement or document centralized authentication (COMP027)
5. Document software criteria compliance (COMP026)
6. Update documentation storage practices (COMP025)

## Recommendations for Suppliers

1. **Immediate Actions:**
   - Review software against COMP026 placement criteria
   - Implement image digest references in Helm charts
   - Document exit strategies for any managed services used

2. **Short-term (30 days):**
   - Implement namespace resource quotas
   - Add network policies
   - Migrate to centralized authentication
   - Enhance build pipeline security

3. **Medium-term (90 days):**
   - Achieve full compliance with enhanced requirements
   - Update documentation per COMP025
   - Implement deployment strategies (blue/green or canary)

## Conclusion

The Cap Gemini review provided valuable recommendations that significantly strengthen the PodiumD hosting standards. The changes enhance security, operational excellence, vendor independence, and compliance while maintaining compatibility with existing components.

These updates align the hosting contract with industry best practices and Common Ground principles, ensuring a robust, secure, and maintainable platform for Dutch municipalities.

---

**Total Requirements Added:** 4 new compliance codes (COMP025-028)  
**Total Requirements Enhanced:** 15+ existing requirements strengthened  
**Lines Changed:** ~283 additions, ~29 deletions  
**Document Version:** 2025Q4  
**Review Date:** October 27, 2025

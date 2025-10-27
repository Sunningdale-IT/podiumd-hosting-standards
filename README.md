# PodiumD Hosting Standards

This repository contains the hosting standards and requirements for the PodiumD platform - a Common Ground-compliant application platform for Dutch municipalities.

## Current Version

**Latest:** [hosting-contract-2025Q4.md](hosting-contract-2025Q4.md) - October 2024

This version incorporates best practices from:
- Microsoft Azure Well-Architected Framework
- Google Site Reliability Engineering (SRE) documentation
- Cap Gemini PodiumD Assessment recommendations
- Current industry standards for Kubernetes and cloud-native applications

## What's New in 2025Q4

See [CHANGES-SUMMARY-2025Q4.md](CHANGES-SUMMARY-2025Q4.md) for a comprehensive summary of all changes and the rationale behind them.

Key additions include:
- **Service naming conventions** and consistency standards
- **Component upgrade time limits** (15 minutes maximum)
- **Version API endpoint** requirements for all components
- **Enhanced health check** and state definitions
- **Golden Signals monitoring** framework (Latency, Traffic, Errors, Saturation)
- **Service Level Objectives (SLO)** and error budgets
- **Disaster recovery** requirements with RTO/RPO
- **Security hardening** requirements
- **API design standards** for better integration
- **Comprehensive documentation** requirements

## Previous Versions

- [hosting-contract-2024Q3.md](hosting-contract-2024Q3.md) - August 2024
- [app-compliance-2024Q3.md](app-compliance-2024Q3.md) - Application compliance tracking

## Document Purpose

This document formalizes what SSC (the hosting provider) expects from Dimpact (the contracting party) and their contracted software developers.

It describes:
- How software should be packaged for integration into the SSC hosting environment
- What developers can expect from SSC Hosting
- Standards for secure, scalable, stable, and manageable deployments

## How to Use This Repository

1. **Software Suppliers**: Review the hosting contract to understand requirements for delivering PodiumD components
2. **SSC Hosting**: Use as the standard for evaluating and onboarding new components
3. **Dimpact**: Reference when contracting with suppliers
4. **Municipalities**: Understand the quality standards for the platform

## Contributing

This is a living document that evolves with input from all parties:
- SSC Hosting
- Dimpact
- Software suppliers
- Municipalities

Updates are managed through Git Pull Requests with version control.

## Support Documents

- **cap-gemini-nl.txt**: Cap Gemini assessment report (Dutch)
- **Bijlage3_full_cleaned.txt**: Additional assessment materials (Dutch)

## Questions or Feedback?

Please open an issue in this repository or contact the SSC Hosting team.

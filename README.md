# PodiumD Hosting Standards

This repository contains the hosting standards documentation for the PodiumD platform. These standards describe the requirements that software suppliers must adhere to when delivering software components as part of our integrated software environment.

## Current Standards

**Latest Version:** [Hosting Contract 2025Q4](hosting-contract-2025Q4.md)

The 2025Q4 version includes comprehensive SRE/DevOps/Platform Engineering best practices for:
- Service naming conventions
- Version API requirements
- Upgrade time limits (15 minutes max)
- Security and supply chain standards
- Kubernetes deployment requirements
- Monitoring, logging, and observability
- Integration and operational best practices

## Previous Versions

- [Hosting Contract 2024Q3](hosting-contract-2024Q3.md) - Original comprehensive version (Aug 2024)
- Hosting Contract 2024Q4 - Placeholder
- Hosting Contract 2025Q1 - Placeholder
- Hosting Contract 2025Q2 - Placeholder
- Hosting Contract 2025Q3 - Placeholder

Note: Versions marked as "Placeholder" have not been published yet. The 2025Q4 version supersedes all previous versions with updated requirements.

## Compliance Tracking

[Application Compliance 2024Q3](app-compliance-2024Q3.md) - Tracks compliance of individual components against standards

## Overview

PodiumD is a collection of Open-Source components built by external software developers for Dutch municipalities. This hosting environment runs on Microsoft Azure Kubernetes Service (AKS) with multiple OTAP (Development, Test, Acceptance, Production) environments.

### Key Requirements

- **100% Automation**: All components must be fully deployable via Helm charts
- **Security**: Container scanning, SBOM, vulnerability management
- **Scalability**: Horizontal scaling support, graceful shutdown
- **Observability**: Structured logging, Prometheus metrics, health probes
- **Version Management**: Semantic versioning with accessible version API
- **Fast Upgrades**: Maximum 15 minutes for standard component upgrades

## Document Updates

This document is maintained in Git with versioning via Pull Requests. Updates are reviewed quarterly or as needed for significant changes.

## Contributing

Updates to these standards should:
1. Be submitted via Pull Request
2. Include rationale for changes
3. Consider impact on existing components
4. Be reviewed by SSC, Dimpact, and supplier representatives

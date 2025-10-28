# GitHub Copilot Instructions for PodiumD Hosting Standards

## Repository Overview

This repository contains documentation and standards for the PodiumD hosting environment, which runs Common Ground-Compliant resources on Microsoft Azure for Dutch municipalities.

## Repository Purpose

- Define hosting standards and requirements for PodiumD applications
- Track compliance of applications against defined standards
- Provide clear expectations for software developers building applications for the PodiumD platform
- Serve as a living document that evolves through Git Pull Requests

## File Structure

- `hosting-contract-2024Q3.md` - Main hosting contract document defining requirements and expectations
- `app-compliance-2024Q3.md` - Compliance tracking matrix for individual applications
- `Bijlage3_full_cleaned.txt` - Reference material (attachment 3)
- `cap-gemini-nl.txt` - Reference material
- `README.md` - Repository overview

## Documentation Standards

### Markdown Format
All documentation files use Markdown format (`.md` extension).

### RFC-Style Requirement Levels
When describing requirements, use RFC 2119 terminology:
- **MUST** / **MUST NOT** - Absolute requirements
- **SHOULD** / **SHOULD NOT** - Strong recommendations
- **MAY** - Optional items

Reference: [RFC 2119: Key words for use in RFCs to Indicate Requirement Levels](https://www.rfc-editor.org/rfc/rfc2119)

### Document Versioning
Documents include a version history table with:
- Version number
- Date
- Author
- Comments/reviewers

Example format:
```markdown
| Version | Date | Author | Comments |  |
|----|----|----|----|----|
| 0.1 | 20/02/2024 | Author Name | Reviewer Names |  |
```

### Compliance Codes
The compliance tracking document uses standardized codes (e.g., COMP001-naming, COMP002-versioning) with levels:
- `Compliant` - Meets requirements
- `NonCompliant` - Does not meet requirements
- `n/a` - Not applicable
- `JIM-STILL-TO-CHECK` - Pending verification

## Content Guidelines

### Hosting Contract Document
- Describes OTAP (Development, Test, Acceptance, Production) environments
- Defines expectations for software packaging and integration
- Uses executive summary for high-level overview
- Includes technical specifications for containerization, Helm charts, and Kubernetes

### App Compliance Document
- Tracks multiple applications against compliance codes
- Uses table format for easy scanning
- Groups by application name and version
- Links compliance codes to requirements in hosting contract

## Making Changes

1. All updates should be made through Git Pull Requests
2. Maintain version history in document headers
3. Keep compliance tracking current when standards change
4. Ensure RFC-style requirement language is used consistently
5. Update both hosting contract and compliance tracking when adding new requirements

## Best Practices

- Keep documentation clear and accessible for developers
- Balance prescriptive requirements with collaborative development
- Reference external standards (RFCs, Common Ground specifications)
- Use tables for structured data (versioning, compliance tracking)
- Maintain consistency in terminology and formatting across documents

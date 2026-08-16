# Security Policy

## Supported versions

Only the latest GitHub Release receives security fixes during the MVP phase.

## Reporting a vulnerability

Use [GitHub private vulnerability reporting](https://github.com/april-jk/dsh-mobile-suite/security/advisories/new) for vulnerabilities affecting the coordinated product. Do not open a public issue with credentials, personal data, exploit details, or a working proof of concept.

Include the affected component and version, deployment model, reproduction steps, expected impact, and any suggested mitigation. Reports are acknowledged on a best-effort community-maintained basis.

For a vulnerability isolated to one component, the corresponding component repository may also be used if it has private reporting enabled.

## MVP limitations

The current release relies on HTTPS/WSS transport encryption and does not provide application-level end-to-end encryption. A Relay operator can inspect in-flight DSH tunnel traffic. Treat Relay operators as trusted infrastructure and use a privately deployed Relay when that trust boundary is not acceptable.

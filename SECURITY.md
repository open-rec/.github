# Security Policy

## Reporting a vulnerability

Do not open a public issue for a suspected vulnerability. Use GitHub's private vulnerability
reporting feature on the affected OpenRec repository. If the affected repository does not expose
private reporting, contact the organization maintainers privately through the contact method listed
on the OpenRec organization profile.

Include the affected component and version, impact, prerequisites, reproduction steps, and any
suggested mitigation. Do not access data that is not yours, disrupt shared services, or publish the
report before maintainers have coordinated a fix.

Maintainers will acknowledge a complete report as soon as practical, validate its scope, coordinate
fixes across affected repositories, and publish an advisory when users can take corrective action.

## Supported versions

Until OpenRec publishes its first stable release, only the latest distribution manifest on the
default branch receives security fixes. After `v1.0.0`, the distribution release notes will identify
the supported release lines and end-of-support dates.

## Deployment responsibility

Example credentials and exposed ports are for isolated development only. Operators must replace
sample secrets, restrict network access, enable authentication where provided, validate TLS
certificates, and follow the deployment security notes before using OpenRec with real data.

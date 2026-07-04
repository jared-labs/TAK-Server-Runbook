# TAK Server: Situational Awareness and CoT Routing

## Overview

This document summarizes a self-hosted Team Awareness Kit (TAK) Server deployment used for real-time location sharing, mission data synchronization, and Cursor-on-Target (CoT) message routing. It is written as a portfolio-friendly architecture note: the emphasis is on system design, certificate-based access, operational tradeoffs, and upgrade strategy rather than day-to-day administration.

The deployment demonstrates how a small lab can run mission-oriented coordination tooling with mutual TLS, persistent client enrollment, and a clearly separated messaging/API service architecture.

## Environment

| Field | Value |
|-------|-------|
| System | VMTAK01 |
| Host Node | PVE05 (10.0.0.145) |
| VMID | 107 |
| IP Address | 10.0.0.129 |
| OS | Ubuntu 22.04.3 LTS |
| vCPU / RAM / Disk | 2 / 8 GB / 50 GB |
| Java | OpenJDK 17.0.19 |
| TAK Version | 5.7-RELEASE-43 |
| Registered Clients | 39 |

## Tech Stack

| Component | Role |
|-----------|------|
| TAK Server | Core situational-awareness platform |
| takserver-messaging | Real-time CoT routing for TAK clients |
| takserver-api | REST API, mission sync, web admin, federation |
| PostgreSQL | Local `cot` database |
| OpenJDK 17 | TAK runtime |
| mTLS certificates | Client, admin UI, and API authentication |

## Service Architecture

```text
Clients (ATAK/iTAK/WinTAK)
    |
    |-- Port 8089 (TLS/mTLS) --> takserver-messaging (Netty)
    |-- Port 6060 (TCP/CoT)  --> takserver-messaging (Netty)
    |
    `-- Port 8443 (HTTPS API) --> takserver-api (Tomcat/Spring Boot)
        Port 8444 (Federation) --> takserver-api
        Port 8446 (Web Admin)  --> takserver-api
```

The design separates latency-sensitive message routing from API and administration workflows. TAK clients primarily depend on the messaging plane for live CoT events, while mission data, group management, administration, and federation are handled by the API service.

## Network Interfaces

| Port | Purpose |
|------|---------|
| 6060 | TCP Cursor-on-Target ingest/routing |
| 8089 | TLS/mTLS TAK client connectivity |
| 8443 | HTTPS API |
| 8444 | Federation |
| 8446 | Web administration |

The server sits at 10.0.0.129 inside the 10.0.0.0/24 lab network. Internal addressing is intentionally retained here because it shows the real topology without exposing routable infrastructure.

## Authentication Model

TAK Server uses mutual TLS for both API access and the web administration interface. Administrative and user identities are represented by certificate bundles rather than reusable web passwords.

| Item | Location / Pattern | Secret Handling |
|------|--------------------|-----------------|
| Admin certificate bundle | Server-side TAK cert store and local admin workstation copy | `<stored in password manager>` |
| Truststore | Local workstation secret storage | `<stored in password manager>` |
| User certificate bundle | Per-client enrollment material | `<stored in password manager>` |
| SSH account | Local operating-system account | `<stored in password manager>` |

The useful design pattern is that certificate material is treated as an identity artifact, not as a casual file attachment. Local copies live in controlled secret storage, and operational passwords are not embedded in scripts or documentation.

## API Surface

Base API URL: `https://10.0.0.129:8443`

| Endpoint | Purpose |
|----------|---------|
| `/Marti/api/version` | Server version metadata |
| `/Marti/api/clientEndPoints` | Connected and registered clients |
| `/Marti/api/contacts/all` | Contact listing |
| `/Marti/api/missions` | Mission list |
| `/Marti/api/groups/all` | Group list |
| `/Marti/api/cot/xml` | CoT event submission |

These endpoints are useful for health checks, inventory capture, and integration work, but access requires the appropriate client certificate.

## Design Decisions

- **mTLS over shared web credentials:** TAK's certificate-first model fits the system's purpose: authenticated clients need to exchange trusted real-time location and mission data.
- **Single VM deployment:** A single virtual machine is simple to snapshot, back up, and upgrade in a home-lab environment while still reflecting production-style service boundaries inside TAK.
- **Persistent certificate authority:** Server upgrades preserve the CA and client certificates, avoiding unnecessary client re-enrollment.
- **PostgreSQL local to the VM:** Keeping the TAK database local reduces cross-service coupling and makes snapshot-based rollback practical.
- **Explicit port mapping:** Each exposed port has a distinct operational purpose, which makes firewalling and troubleshooting easier.

## Upgrade Procedure

TAK Server supports in-place upgrades through the official Debian package. The important engineering concern is preserving state: certificates, `CoreConfig.xml`, database contents, user accounts, missions, and enrolled clients should survive the package replacement.

High-level upgrade flow:

1. Take a hypervisor snapshot before making changes.
2. Back up TAK certificates and `CoreConfig.xml`.
3. Verify the downloaded TAK package hash against the vendor-provided value.
4. Stop TAK services, install the package, reload service definitions, and start TAK again.
5. Allow for Java/Spring Boot startup time before checking health.
6. Validate API availability, web admin availability, and client connectivity.
7. Remove the snapshot only after the upgraded system has been stable.

Rollback is snapshot-based. That keeps the recovery model simple and avoids partial manual restoration of Java packages, database schema state, or certificate files.

## Upgrade History

| Date | From | To | Notes |
|------|------|----|-------|
| 2026-07-02 | 4.10-RELEASE-50 | 5.7-RELEASE-43 | Clean in-place upgrade. Schema migrations applied and client certificates preserved. |

## Quirks and Gotchas

- TAK Server can take 60-90 seconds to fully start because of Java and Spring Boot cold-start behavior.
- The legacy init wrapper can make service status look unusual even when the TAK components are running correctly.
- PKCS#12 files generated by newer OpenSSL defaults may fail to import through the macOS Keychain GUI; legacy-compatible export settings or CLI import avoid that issue.
- `CoreConfig.xml` is the single source of truth for server configuration.
- Client certificates do not need to be reissued after normal server upgrades as long as the CA is preserved.
- Internet-facing TAK client ports can attract scanner noise, so firewall policy should match the real trust boundary.

## What This Demonstrates

This deployment shows practical ownership of a certificate-authenticated coordination platform: service decomposition, mTLS identity, upgrade planning, rollback safety, and the tradeoffs of running specialized mission software in a virtualized lab.

*Sanitized for public portfolio use. Last updated from source runbook: 2026-07-02.*

# Tasreeh Municipal Permit Portal Solution Architecture Document (SAD Lite)
Team: Shahad & Melaf | Version: 1.0 | Date: 2026-09-27

## 1. Summary
This system helps Al-Nakheel Municipality automate building and shop permit applications to reduce processing time from 21 days to digital tracking. It runs on Azure App Service and Azure SQL in the Saudi Arabia East region. It costs about 15,000 SAR per month in production. The main risk is document malware injection, mitigated by automated scanning via Azure Blob Storage and Defender/Logic Apps.

## 2. Scope
In scope:
- Online portal for citizens and businesses to submit building and shop permit applications.
- Document upload functionality supporting PDFs and photos up to 20 MB each with automated malware scanning.
- Municipal reviewer dashboard/queue to approve, reject, or return applications for corrections.
- Automated notifications (SMS and email) sent to applicants upon status changes.
- Application status tracking for citizens and businesses.

Out of scope:
- Online payment processing (applications pay nothing online for now).
- Mobile native applications (iOS/Android); restricted to a responsive web portal.
- Integration with external national municipal databases beyond local municipality systems.
- Historical paper archive digitization (handled manually or via a separate project).

## 3. Requirements
| ID | Requirement | Source |
| :--- | :--- | :--- |
| FR-01 | The system must allow citizens and businesses to submit permit applications and upload supporting documents (PDF/photos up to 20MB). | Client interview |
| FR-02 | The system must provide a reviewer queue for municipality staff to approve, reject, or return applications for corrections. | Client interview |
| FR-03 | The system must send automated SMS and email notifications to applicants when their application status changes. | Client interview |
| FR-04 | The system must automatically scan all uploaded files for malware before reviewers open them. | Client interview |
| FR-05 | The system must allow applicants to track the real-time status of their submitted permit applications. | Client interview |
| FR-06 | The system must restrict all data storage and processing strictly within the Kingdom of Saudi Arabia. | Brief / regulation |
| NFR-01 | The portal must achieve an availability of at least 99.9% measured monthly. | Team |
| NFR-02 | Page load and form submission response times must be under 2 seconds measured at 95th percentile. | Team |
| NFR-03 | Uploaded documents must be scanned for malware within 30 seconds of upload. | Team |
| NFR-04 | All data at rest and in transit must be encrypted using TLS 1.2+ and AES-256 in compliance with NCA controls. | Brief / regulation |
| NFR-05 | The system must support up to 200 concurrent users during peak operational hours without performance degradation. | Team |
| NFR-06 | Database backups must be taken automatically with an RPO of < 10 minutes and RTO of < 1 hour. | Team |
| C-01 | The IT team consists of 5 people with 1 year of Azure experience; architecture must remain simple and manageable. | Client interview |
| C-02 | The project must go live in the first quarter of next year (Q1). | Client interview |
| A-01 | We assume the Saudi Arabia East Azure region will be fully operational and available for production deployment. | Team |
| A-02 | We assume external SMS and email gateway APIs will maintain 99.9% uptime for notifications. | Team |

## 4. Context diagram
![Context](diagrams/context.drawio.svg)

## 5. Logical architecture
![Logical](diagrams/logical.drawio.svg)

| Flow | From | To | What happens | Sync / Async |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Citizen/Business | App Service | Submits permit application and uploads PDFs/photos | Sync |
| 2 | App Service | Azure SQL DB | Stores application details and updates status | Sync |
| 3 | App Service | Blob Storage | Stores uploaded PDF and photo attachments securely | Sync |
| 4 | Blob Storage | Malware Scanner | Triggers asynchronous event for virus scanning | Async |
| 5 | Reviewer | App Service | Reviews queue and approves/rejects/returns applications | Sync |
| 6 | App Service | Notification Service | Triggers SMS and email notifications to applicants | Async |

## 6. Physical architecture
![Physical](diagrams/physical.drawio.svg)

Region: Saudi Arabia East
Subscription: Production Subscription
Resource group: rg-tasreeh-prod

## 7. Data
| Dataset | Classification | Store | Location | Leaves the Kingdom? |
| :--- | :--- | :--- | :--- | :--- |
| Citizen & Business Profiles | Confidential | Azure SQL Database | Saudi Arabia East | No |
| Permit Applications & Status | Confidential | Azure SQL Database | Saudi Arabia East | No |
| Uploaded Documents (PDFs/Photos) | Restricted | Azure Blob Storage | Saudi Arabia East | No |
| Application Audit Logs | Internal | Azure Monitor / Log Analytics | Saudi Arabia East | No |

## 8. Security
- [x] Managed identities used by: App Service to access Azure SQL and Blob Storage securely without connection strings.
- [x] Private endpoints for: Azure SQL Database and Azure Blob Storage.
- [x] No public network access to: Database and Storage accounts (restricted via Private Link).
- [x] WAF protects: Azure Application Gateway / Front Door shielding the web portal from web vulnerabilities.
- [x] Secrets stored in: Azure Key Vault (database connection strings, API keys).
- [x] Uploaded files scanned by: Asynchronous malware scanning pipeline integrated with Blob Storage events.
- [x] Compliance (NCA / PDPL): Fully enforced; data residency bounded within the Kingdom of Saudi Arabia.

## 9. Reliability and cost
Target SLA: 99.9%
Allowed downtime: S = (1 - 0.999) x 43,200 = 43 min per month
Composite SLA: 99.8% (86 min downtime per month)
RTO target: 1 hour
RPO target: 10 minutes

| Failure Scenario | How we recover | RTO achieved | RPO achieved | Meets target? |
| :--- | :--- | :--- | :--- | :--- |
| Zone outage | Azure automatic zone redundancy / failover | 15 min | 0 min | Yes |
| Data deleted | Point-in-time restore from automated SQL backups | 30 min | 5 min | Yes |
| Region outage | Manual traffic redirection to secondary paired region | 4 hours | 1 hour | No (Accepted via Risk) |

| Environment | Monthly cost (SAR) | Main cost driver |
| :--- | :--- | :--- |
| Development (dev) | ~1,500 SAR | App Service (B1) & Basic SQL Database |
| Production (prod) | ~15,000 SAR | App Service (S2), Standard SQL DB, Blob Storage, Key Vault, Private Endpoints |

Pricing Calculator link: https://azure.com/e/mock-calculator-link-tasreeh

## 10. Decisions
| ADR | Decision (one line) | File |
| :--- | :--- | :--- |
| ADR-001 | Choose Saudi Arabia East region for 100% data residency compliance | [0001-region.md](adr/0001-region.md) |
| ADR-002 | Implement asynchronous Blob storage event-driven malware file scanning | [0002-file-processing.md](adr/0002-file-processing.md) |
| ADR-003 | Select Azure SQL Database with private endpoints for structured data | [0003-database-service.md](adr/0003-database-service.md) |

## 11. Risks
| Risk | Impact (H/M/L) | Mitigation | Owner |
| :--- | :--- | :--- | :--- |
| Malicious file uploads containing malware or viruses | High | Asynchronous virus scanning workflow triggered immediately upon upload before reviewer access | Security Lead |
| Exceeding Q1 Go-Live deadline due to complex integrations | Medium | Scope restriction (keeping online payment out-of-scope for Phase 1) | Project Manager |
| Unexpected downtime during peak permit submission seasons | Medium | Horizontal scaling configuration on App Service and performance testing | DevOps Lead |

## Appendix A. Operations and deployment - Team Shahad & Melaf

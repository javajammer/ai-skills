# Functional Specification Design (FSD)
## Sistem Manajemen Aset dan Monitoring Infrastruktur (SAMAHI)

**Project Name:** SAMAHI (Sistem Manajemen Aset dan Monitoring Infrastruktur)
**Version:** 1.1 (Improved from 1.0 Draft)
**Date:** 17 Mei 2026
**Author:** AI Technical Architect (Gemini-3-Flash-Preview)
**Status:** Review-Ready
**Audience:** VP Infrastructure, Tech Lead, DevOps Team

---

## 1. Project Context

### 1.1 Background
PT Nusantara Digital mengelola infrastruktur hibrida yang signifikan (200+ server) di 3 lokasi geografis. Saat ini, ketergantungan pada pemantauan manual (spreadsheet) mengakibatkan *Mean Time To Detect* (MTTD) yang tinggi (4-6 jam) dan risiko *human error* yang besar. Proyek SAMAHI bertujuan untuk memodernisasi tumpukan pemantauan (*monitoring stack*) menjadi sistem yang proaktif, otomatis, dan terpusat.

### 1.2 Objectives
1. **Otomatisasi Monitoring:** Mengganti proses manual dengan *automated scraping* dan *log ingestion*.
2. **Real-time Visibility:** Menyediakan *single pane of glass* melalui Grafana untuk seluruh infrastruktur.
3. **Proactive Alerting:** Mengurangi MTTD dari > 4 jam menjadi < 15 menit melalui sistem alert terintegrasi (Slack).
4. **Data-Driven Planning:** Menyediakan histori performa untuk *capacity planning* yang lebih akurat.

### 1.3 Stakeholders
| Role | Responsibility |
|---|---|
| **VP Infrastructure** | Project Sponsor & Approval |
| **Tech Lead** | Architecture Review & Technical Guidance |
| **DevOps/Infra Team** | Implementation & Day-2 Operations |
| **Management** | Business Continuity & Capacity Reporting |

### 1.4 Timeline & Methodology
- **Methodology:** Agile (Scrum) - 6 Sprints (2 weeks per sprint).
- **Total Duration:** 12 Weeks.

---

## 2. Methodology Notes (Agile Alignment)

Proyek ini akan dijalankan dengan disiplin Agile:
- **Sprints:** 6 Sprint selama 2 minggu masing-masing.
- **Ceremonies:** Daily Standup, Sprint Planning, Sprint Review, dan Retrospective.
- **Artifacts:** Product Backlog (Jira), Sprint Backlog, dan Definition of Done (DoD).
- **Phased Delivery:** Monitoring server akan aktif di Sprint 2, Dashboard dasar di Sprint 4, dan Full Alerting di Sprint 5.

---

## 3. Requirements

### 3.1 Functional Requirements (FR)

| ID | Requirement | Priority | Acceptance Criteria | Owner |
|---|---|---|---|---|
| **FR-001** | Metric Collection | P1 | Given server target, Prometheus must scrape metrics every 15s with 99.9% success rate. | Infra |
| **FR-002** | Centralized Logging | P1 | Given application logs, Fluentd must ingest logs to Elasticsearch with < 10s latency. | DevOps |
| **FR-003** | Real-time Dashboards | P1 | Given stored metrics, Grafana must display CPU/Mem/Disk/Network graphs with < 5s load time. | Infra |
| **FR-004** | Threshold Alerting | P1 | Given metric exceeds threshold, Alertmanager must send Slack notification in < 2 mins. | DevOps |
| **FR-005** | Historical Reporting | P2 | System must store metric history for at least 90 days for trend analysis. | Infra |
| **FR-006** | LDAP Integration | P2 | Users must be able to login to Grafana using corporate LDAP credentials. | Security |

### 3.2 Non-Functional Requirements (NFR)

| ID | Category | Requirement | Target |
|---|---|---|---|
| **NFR-001** | Availability | Uptime sistem monitoring | 99.9% (High Availability Setup) |
| **NFR-002** | Performance | Time to Detect (TTD) | < 15 Menit |
| **NFR-003** | Scalability | Support tambahan server | Mampu menangani hingga 500 server tanpa degradasi |
| **NFR-004** | Security | Network Access | Hanya dapat diakses dari VPN/Internal Network |
| **NFR-005** | Compliance | Log Filtering | PII (Email, Phone, Credit Card) harus difilter sebelum masuk ES |

---

## 4. Scope

### 4.1 In Scope
- Monitoring 200+ Server (Linux & Windows).
- Monitoring Database (PostgreSQL, MySQL, MongoDB).
- Centralized Log Management (Full-text search).
- Dashboard Visualization (Grafana).
- Automated Alerting (Slack).

### 4.2 Out of Scope
- Application Performance Monitoring (APM/Tracing).
- Security Vulnerability Scanning.
- Backup & Disaster Recovery Monitoring (Phase 2).
- Public Cloud Monitoring (AWS/GCP).

---

## 5. Proposed Solution Architecture

### 5.1 Tech Stack (SAMAHI Stack)
- **Collector:** Prometheus (Metrics), Fluentd (Logs), Node Exporter/WMI Exporter.
- **Database:** TimescaleDB (TSDB for metrics), Elasticsearch (Log storage).
- **Visualization:** Grafana.
- **Alerting:** Alertmanager (Routing), Slack Webhook (Delivery).
- **Deployment:** Docker Compose on 2 High-Spec Nodes (Primary/Standby).

### 5.2 Data Flow
1. **Pull Mechanism:** Prometheus melakukan scrape metrics dari exporter di setiap node via HTTP (Port 9100/9182).
2. **Push Mechanism:** Fluentd di setiap node mengirim log secara asinkron ke Elasticsearch Cluster.
3. **Storage:** Metrics disimpan di TimescaleDB; Logs di Elasticsearch dengan indeks per-hari.
4. **Alerting Loop:** Prometheus mengevaluasi rules setiap 1 menit -> Trigger Alertmanager -> Slack.

---

## 6. Implementation Plan (Agile Sprints)

| Sprint | Goal | Key Deliverables |
|---|---|---|
| **Sprint 1** | Foundation | Setup High-Spec Monitoring Nodes, LDAP Integration, Docker Orchestration. |
| **Sprint 2** | Metrics Ingestion | Deployment of Node Exporters to 200 servers, Prometheus configuration. |
| **Sprint 3** | Logging Ingestion | Deployment of Fluentd, Elasticsearch Cluster setup, Log rotation policy. |
| **Sprint 4** | Visualization | Creation of Overview, Detail, and Database Dashboards in Grafana. |
| **Sprint 5** | Alerting & Tuning | Configuration of Alertmanager rules, Slack integration, False-positive tuning. |
| **Sprint 6** | UAT & Handover | User Acceptance Testing, Operational Manual, Final Go-Live. |

---

## 7. Security & Risk Management

### 7.1 Security Controls
- **Authentication:** LDAP/Active Directory integration for all components.
- **Authorization:** RBAC (Admin, Editor, Viewer) di Grafana.
- **Network:** Komunikasi Prometheus-Exporter diproteksi dengan IP Whitelisting/Firewall.
- **Logging Privacy:** Fluentd filter rules untuk me-redact informasi sensitif.

### 7.2 Risk Assessment
| Risk | Impact | Mitigation |
|---|---|---|
| **Resource Contention** | High | Dedicated monitoring servers (tidak dicampur dengan production apps). |
| **Log Explosion** | Medium | Implementasi Index Lifecycle Management (ILM) di Elasticsearch (30 hari retensi). |
| **Slack API Down** | Low | Secondary alerting via Email sebagai backup. |

---

## 8. Assumptions & Open Questions

### 8.1 Assumptions
1. Seluruh 200 server memiliki akses jaringan ke monitoring server pusat pada port yang diperlukan.
2. Tim Security akan menyediakan akses LDAP/AD dalam minggu ke-1.
3. Kapasitas penyimpanan awal (2TB) mencukupi untuk retensi metrics 90 hari dan logs 30 hari.

### 8.2 Open Questions
1. Apakah diperlukan integrasi dengan ITSM tool (seperti Jira/ServiceNow) untuk ticketing otomatis dari alert?
2. Apakah ada limitasi throughput pada Slack Webhook yang dimiliki perusahaan?
3. Apakah diperlukan monitoring untuk network devices (Switches/Routers) via SNMP di Phase 1?

---

## 9. Completeness Check

| Section | Status | Notes |
|---|---|---|
| Project Context | COMPLETE | Detailed background and objectives added. |
| Requirements (FR/NFR) | COMPLETE | Structured table with Priority and AC. |
| Architecture & Data Flow | COMPLETE | Clear tech stack and flow description. |
| Implementation Plan | COMPLETE | Mapped to Agile Sprints. |
| Security & Risks | COMPLETE | Controls and mitigation identified. |
| Assumptions & Questions | COMPLETE | Added to clarify scope. |

---
**Security & Confidentiality Note:** Dokumen ini bersifat INTERNAL. Redact data sensitif (seperti IP Address spesifik atau LDAP Secret) sebelum didistribusikan ke pihak ketiga.

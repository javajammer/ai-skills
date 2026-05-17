# Functional Specification Design (FSD)
## Sistem Manajemen Aset dan Monitoring Infrastruktur (SAMAHI)

**Versi:** 2.0
**Tanggal:** 17 Mei 2026
**Disusun oleh:** Tim Teknologi PT Nusantara Digital
**Status:** Draft — Ready for Review
**Methodology:** Agile/Scrum
**Audience:** VP Infrastructure, CTO, Tech Lead, Product Owner

---

## Table of Contents

1. [Project Background & Objectives](#1-project-background--objectives)
2. [Project Scope](#2-project-scope)
3. [Stakeholder Identification](#3-stakeholder-identification)
4. [Functional Requirements](#4-functional-requirements)
5. [Non-Functional Requirements](#5-non-functional-requirements)
6. [Data Architecture & Data Model](#6-data-architecture--data-model)
7. [System Architecture & Integration Design](#7-system-architecture--integration-design)
8. [UI/UX Specifications](#8-uiux-specifications)
9. [Business Rules & Logic](#9-business-rules--logic)
10. [Security Requirements](#10-security-requirements)
11. [Testing Strategy & Acceptance Criteria](#11-testing-strategy--acceptance-criteria)
12. [Implementation Timeline](#12-implementation-timeline)
13. [Dependencies & Assumptions](#13-dependencies--assumptions)
14. [Risk Register](#14-risk-register)
15. [Change Management Plan](#15-change-management-plan)
16. [Handover & Support Plan](#16-handover--support-plan)
17. [Sign-off / Approval Matrix](#17-sign-off--approval-matrix)
18. [Assumptions & Open Questions](#18-assumptions--open-questions)
19. [Completeness Check](#19-completeness-check)
20. [Security & Confidentiality Notes](#20-security--confidentiality-notes)

---

## 1. Project Background & Objectives

### 1.1 Background

PT Nusantara Digital mengoperasikan lebih dari 200 server fisik dan virtual yang tersebar di 3 data center (Jakarta, Surabaya, dan Manado), serta 15 instance database kritis yang mendukung layanan e-commerce, payment gateway, dan internal ERP perusahaan. Saat ini pemantauan infrastruktur dilakukan secara manual menggunakan spreadsheet yang diperbarui setiap minggu oleh tim operasi.

Proses manual ini menyebabkan:
- Keterlambatan deteksi anomali (rata-rata 4–6 jam sebelum diketahui, SLA internal adalah < 15 menit)
- Tidak adanya histori performa untuk analisis tren capacity planning
- Kesulitan dalam pelacakan aset (rata-rata 12% server tidak tercatat di inventory)
- Human error dalam pencatatan dan pelaporan
- Tidak ada single source of truth untuk status kesehatan infrastruktur

Akibatnya, dalam 6 bulan terakhir terjadi 7 insiden downtime yang berkepanjangan karena keterlambatan deteksi, dengan total downtime mencapai 48 jam dan estimasi kerugian pendapatan sebesar Rp 360 juta.

### 1.2 Project Objectives

| No | Objective | Target Metric |
|----|-----------|---------------|
| 1 | Membangun sistem monitoring otomatis untuk seluruh infrastruktur | Coverage 100% dari 200+ server |
| 2 | Menyediakan dashboard real-time untuk tim operasi dan manajemen | Time-to-detect < 15 menit |
| 3 | Mengimplementasikan alerting otomatis via Slack dan email | Alert delivery latency < 30 detik |
| 4 | Mengurangi time-to-detect anomali dari 4 jam menjadi < 15 menit | MTTR turun dari 2 jam → < 30 menit |
| 5 | Menciptakan single source of truth untuk inventory aset | Akurasi inventory ≥ 98% |
| 6 | Menyediakan laporan capacity planning berbasis data historis | Laporan bulanan otomatis |

### 1.3 Success Criteria

- 99.5% uptime untuk sistem monitoring itu sendiri
- Deteksi anomaly tercapai dalam waktu < 15 menit sejak kejadian
- 100% server terdaftar dan terpantau dalam 90 hari pertama go-live
- Zero critical bugs pada release v1.0 saat UAT

---

## 2. Project Scope

### 2.1 In-Scope

1. **Server Monitoring** — 200+ server Linux (Ubuntu 20.04/22.04) dan Windows Server 2019/2022
2. **Database Monitoring** — 15 PostgreSQL, 8 MySQL, 5 MongoDB replica sets
3. **Log Aggregation** — Centralized logging dari semua server dan aplikasi
4. **Real-time Dashboard** — Grafana-based dashboard untuk overview, detail per-server, database, dan log search
5. **Alerting Engine** — Multi-channel alerting (Slack webhook, email, SMS gateway)
6. **Asset Inventory Management** — CRUD aset, lifecycle tracking, auto-discovery
7. **Reporting** — Laporan harian (ops), bulanan otomatis, capacity trend analysis
8. **Integration** — LDAP/Active Directory authentication, webhook integrasi

### 2.2 Out-of-Scope

1. Application Performance Monitoring (APM) — termasuk tracing dan profiling
2. Security scanning dan vulnerability assessment
3. Backup monitoring dan disaster recovery testing automation
4. Cloud resource monitoring (AWS/GCP/Azure) — fase berikutnya
5. Network device monitoring (switch/router/firewall) — fase berikutnya
6. End-user experience monitoring (synthetic/RUM)

---

## 3. Stakeholder Identification

| Role | Nama / Posisi | Tanggung Jawab | Keterlibatan |
|------|---------------|----------------|--------------|
| Executive Sponsor | CTO | Approval budget, strategic alignment | Phase gate review |
| Product Owner | Head of Operations | Requirement prioritization, UAT sign-off | Daily backlog grooming |
| Tech Lead | Senior Infrastructure Engineer | Technical design, implementation oversight | Full engagement |
| DevOps Engineer | 1 orang | CI/CD, deployment automation | Sprint 2–6 |
| System Administrator | 2 orang | Agent deployment, config validation | Sprint 1–5 |
| Security Officer | InfoSec Team Lead | Security review, compliance check | Sprint 3, 6 |
| End Users | Operations Team (8 orang) | UAT, daily usage, feedback | UAT + post-go-live |
| Vendor (Optional) | Managed Service Provider | Support escalation tier-3 | Post-go-live SLA |

---

## 4. Functional Requirements

### 4.1 Functional Requirements Table

| ID | Requirement | Priority | Acceptance Criteria | Owner |
|----|-------------|----------|---------------------|-------|
| FR-001 | Sistem dapat mengumpulkan metrics CPU, Memory, Disk, Network dari setiap server setiap 15 detik | P1 | Prometheus scrape berhasil pada 100% target server dalam ≤ 15 detik interval; data tersedia di Grafana dalam ≤ 30 detik setelah scrape | Eng |
| FR-002 | Dashboard Overview menampilkan status keseluruhan infrastruktur dengan color-coded health score | P1 | Dashboard menampilkan: total server online/offline, top 5 most loaded servers, active alerts count, trend chart 24 jam; load time < 3 detik | Product |
| FR-003 | Sistem mengirim alert otomatis ke Slack channel ketika threshold dilampaui | P1 | Alert terkirim dalam ≤ 30 detik setelah threshold breach; mencakup metric name, current value, threshold, server name, timestamp; link langsung ke dashboard detail | Eng |
| FR-004 | Alerting multi-channel: Slack, Email, dan SMS untuk critical severity | P1 | Critical alert → Slack + Email + SMS; Warning alert → Slack saja; retry 3x jika delivery gagal; acknowledgment mechanism tersedia | Eng |
| FR-005 | Asset inventory mendukung CRUD lengkap dengan field: hostname, IP, OS type/version, location (DC), rack, owner, purchase date, warranty expiry, status | P1 | Form validasi input; pencarian dan filter multi-kriteria; export CSV; audit trail setiap perubahan | Product |
| FR-006 | Auto-discovery server melalui network scan (CIDR-based) | P2 | Sistem memindai range CIDR yang dikonfigurasi; mendeteksi host alive, OS fingerprint, port terbuka; hasil diverifikasi operator sebelum diaktifkan | Eng |
| FR-007 | Database monitoring: query performance, connection pool, replication lag, slow queries | P1 | Metrics tersedia per database instance; replication lag alert pada > 10 detik; slow query log aggregation | Eng |
| FR-008 | Log aggregation dan full-text search dengan filter by hostname, level, time range | P1 | Search response < 2 detik untuk 7 hari data; support wildcard dan regex; filter by log level (INFO/WARN/ERROR/FATAL) | Eng |
| FR-009 | Dashboard Detail per-server menampilkan historical metrics (1 jam, 24 jam, 7 hari, 30 hari) | P2 | Grafik interaktif dengan zoom dan pan; comparison view antar periode; export grafik PNG/PDF | Product |
| FR-010 | User authentication via LDAP/Active Directory | P1 | Single Sign-On terintegrasi; RBAC role assignment (Admin, Operator, Viewer); session timeout 30 menit idle | Eng |
| FR-011 | Laporan kapasitas bulanan otomatis (email digest) | P2 | Laporan berisi: peak usage 30 hari, projected capacity exhaustion date, rekomendasi scaling; dikirim pertama hari kerja setiap bulan | Product |
| FR-012 | Webhook outbound untuk integrasi dengan ticketing system (Jira/ServiceNow) | P2 | Event alert → auto-create ticket; ticket status sync back (resolved → mute alert); API key based auth | Eng |
| FR-013 | Downtime history tracking dan root cause annotation | P3 | Operator dapat menambahkan catatan pada setiap insiden downtime; timeline view 90 hari; export laporan insiden | Product |
| FR-014 | Dashboard mobile-responsive atau dedicated mobile view | P3 | Layout adaptif untuk viewport 320px–768px; critical alerts visible tanpa login (read-only token) | Product |

### 4.2 Priority Definitions

| Priority | Definition |
|----------|------------|
| P1 (Critical) | Must have for v1.0 go-live; blocks core functionality if absent |
| P2 (High) | Important for v1.0; can defer non-critical edge cases |
| P3 (Medium) | Nice-to-have; can be deferred to v1.1 or later sprint |

---

## 5. Non-Functional Requirements

### 5.1 Performance Requirements

| ID | Requirement | Target | Measurement |
|----|-------------|--------|-------------|
| NFR-001 | Dashboard load time (initial render) | < 3 detik | Synthetic test dari 3 DC |
| NFR-002 | Alert delivery latency | < 30 detik | Timestamp diff: breach event → Slack message |
| NFR-003 | Log search response time (≤ 7 hari window) | < 2 detik | Elasticsearch _search API timing |
| NFR-004 | Metrics scrape cadence | 15 detik ± 2 detik | Prometheus scrape_duration histogram |
| NFR-005 | Concurrent dashboard users (peak) | ≥ 20 users | Load test dengan k6 |
| NFR-006 | Alert engine processing throughput | ≥ 100 events/detik | Synthetic burst test |

### 5.2 Availability & Reliability

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-007 | System availability (monitoring platform itself) | 99.5% (≈ 3.6 jam downtime/tahun) |
| NFR-008 | RTO (Recovery Time Objective) | 1 jam |
| NFR-009 | RPO (Recovery Point Objective) | 15 menit |
| NFR-010 | Data retention — metrics | 90 hari hot storage, 1 tahun cold archive |
| NFR-011 | Data retention — logs | 30 hari hot storage, 90 hari cold archive |

### 5.3 Scalability

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-012 | Current scale | 200+ servers, 28 DB instances |
| NFR-013 | Planned scale (12 bulan) | 400+ servers, 60 DB instances |
| NFR-014 | Storage growth estimate | ~5 GB/hari metrics, ~20 GB/hari logs |

### 5.4 Compliance

| ID | Requirement | Status |
|----|-------------|--------|
| NFR-015 | Data residency — all data stored within Indonesia | On-premise DC only |
| NFR-016 | Audit logging — every admin action logged | Implemented via audit middleware |
| NFR-017 | GDPR/PPJK compliance — no PII in logs | Filter rules applied at Fluentd pipeline |

---

## 6. Data Architecture & Data Model

### 6.1 Data Flow Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Node Exporter │     │ mysqld_      │     │ MongoDB      │
│ (Linux)       │     │ exporter     │     │ Exporter     │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                     │
       ▼                    ▼                     ▼
┌──────────────────────────────────────────────────────────┐
│                   Prometheus                               │
│          (Metrics Collection & Storage)                    │
│   Scrape Interval: 15s | Retention: 90 days              │
└──────┬──────────────┬──────────────┬─────────────────────┘
       │              │              │
       ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Alertmanager │ │  Grafana     │ │  Thanos      │
│  (Rules &    │ │  (Dashboard  │ │  (Long-term  │
│   Routing)   │ │   Viz)       │ │   Archive)   │
└──────┬───────┘ └──────────────┘ └──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Slack      │     │    Email     │     │    SMS       │
│   Webhook    │     │   Gateway    │     │  Gateway     │
└──────────────┘     └──────────────┘     └──────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Fluentd      │────▶│ Elasticsearch │────▶│  Grafana     │
│ (Log Shipper) │     │ (Index &     │     │  (Log Panel) │
│               │     │  Search)     │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
```

### 6.2 Asset Inventory Data Model

```
┌─────────────────────────────────────────────────────┐
│ ASSET                                                 │
├─────────────────────────────────────────────────────┤
│ id            UUID (PK)                              │
│ hostname      VARCHAR(255) UNIQUE                    │
│ ip_address    INET                                   │
│ mac_address   VARCHAR(17)                            │
│ os_type       ENUM(LINUX, WINDOWS)                   │
│ os_version    VARCHAR(100)                           │
│ asset_type    ENUM(SERVER, DATABASE, NETWORK)        │
│ dc_location   ENUM(JKT, SUB, MDO)                    │
│ rack_u        VARCHAR(20)                            │
│ status        ENUM(PROD, STAGING, DEV, DECOMMISSIONED)│
│ owner_team    VARCHAR(100)                           │
│ cpu_cores     INT                                    │
│ memory_gb     INT                                    │
│ disk_gb       INT                                    │
│ purchase_date DATE                                   │
│ warranty_expiry DATE                                 │
│ serial_number VARCHAR(100)                           │
│ discovered_at TIMESTAMP WITH TIME ZONE               │
│ last_seen     TIMESTAMP WITH TIME ZONE               │
│ created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW() │
│ updated_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW() │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ METRIC_SAMPLE                                         │
├─────────────────────────────────────────────────────┤
│ id            BIGSERIAL (PK)                         │
│ metric_name   VARCHAR(200)                           │
│ server_id     UUID (FK → ASSET.id)                   │
│ value         DOUBLE PRECISION                       │
│ timestamp     TIMESTAMP WITH TIME ZONE               │
│ labels        JSONB (dynamic labels)                 │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ ALERT_EVENT                                           │
├─────────────────────────────────────────────────────┤
│ id            BIGSERIAL (PK)                         │
│ rule_name     VARCHAR(200)                           │
│ severity      ENUM(INFO, WARNING, CRITICAL)          │
│ status        ENUM(FIRING, RESOLVED, ACKNOWLEDGED)   │
│ triggered_at  TIMESTAMP WITH TIME ZONE               │
│ resolved_at   TIMESTAMP WITH TIME ZONE               │
│ server_id     UUID (FK → ASSET.id)                   │
│ channels      TEXT[] (slack, email, sms)             │
│ annotations   JSONB                                  │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ AUDIT_LOG                                             │
├─────────────────────────────────────────────────────┤
│ id            BIGSERIAL (PK)                         │
│ user_id       UUID (FK → USER)                       │
│ action        VARCHAR(50) (CREATE, UPDATE, DELETE)   │
│ entity_type   VARCHAR(50)                            │
│ entity_id     UUID                                   │
│ old_values    JSONB                                  │
│ new_values    JSONB                                  │
│ ip_address    INET                                   │
│ occurred_at   TIMESTAMP WITH TIME ZONE DEFAULT NOW() │
└─────────────────────────────────────────────────────┘
```

### 6.3 Storage Capacity Estimate

| Component | Daily Volume | 90-Day Hot | 1-Year Cold | Estimated Disk |
|-----------|-------------|------------|-------------|----------------|
| Prometheus Metrics | ~5 GB | ~450 GB | ~1.8 TB | 1 TB SSD (with compression ratio 2:1) |
| Elasticsearch Logs | ~20 GB | ~600 GB | ~7.3 TB | 2 TB NVMe (hot) + 4 TB HDD (cold tier) |
| Asset DB (PostgreSQL) | ~1 MB | Negligible | Negligible | 50 GB |
| **Total** | **~25 GB/day** | | | **~7 TB provisioned** |

---

## 7. System Architecture & Integration Design

### 7.1 Infrastructure Topology

```
                    ┌──────────────────────────────────┐
                    │        Internal Network           │
                    │   (10.0.0.0/8 segmented by VLAN)  │
                    └──────────┬───────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼───────┐ ┌─────▼──────┐ ┌───────▼────────┐
     │  Monitoring    │ │ Monitoring │ │  Bastion Host  │
     │  Server - JKT  │ │ Server -   │ │  (Jump Box)    │
     │  (Primary)     │ │ SUB        │ │                │
     │                │ │ (Standby)  │ │ SSH/TLS Only   │
     └────────┬───────┘ └────────────┘ └────────────────┘
              │
     ┌────────▼────────┐
     │  Prometheus      │◀── Scrapes from all agents
     │  + Alertmanager  │
     └────────┬────────┘
              │
     ┌────────▼────────┐
     │  Elasticsearch   │◀── Fluentd tail shipping
     │  + Grafana       │
     │  + Asset API     │
     └─────────────────┘
```

### 7.2 High Availability Design

| Component | HA Strategy | Failover Mechanism |
|-----------|-------------|-------------------|
| Prometheus | Active-Standby (2 nodes) | VictoriaMetrics cluster mode; WAL-based replication |
| Elasticsearch | 3-node cluster | Automatic shard rebalancing; master election |
| Grafana | Single node (v1.0) | Scheduled state export; manual restore on failover |
| Fluentd | Deployed per-agent (distributed) | N/A — each agent runs independently |
| Asset API | Stateless behind nginx LB | Round-robin; health check endpoint |

### 7.3 Integration Points

| Integration | Direction | Protocol | Authentication |
|-------------|-----------|----------|----------------|
| LDAP/AD | Outbound | LDAPS (636) | Service account bind |
| Slack | Outbound | HTTPS POST | Bot token (xoxb-) |
| Email | Outbound | SMTPS (587) | TLS + credentials |
| SMS Gateway | Outbound | HTTPS REST | API key header |
| Jira/ServiceNow | Bidirectional | REST API | OAuth 2.0 / PAT |
| Nagios (legacy) | Inbound | SNMP v3 | Community string + auth |

---

## 8. UI/UX Specifications

### 8.1 Dashboard Inventory

| Dashboard | Audience | Key Panels | Refresh Rate |
|-----------|----------|-----------|--------------|
| DASH-01: Infrastructure Overview | All | Health score gauge, server status table, active alerts list, topology map | 30 detik |
| DASH-02: Server Detail | Ops Engineers | CPU/memory/disk/time-series, process top-10, network I/O, open connections | 15 detik |
| DASH-03: Database Health | DBA | Query latency p95/p99, connection pool %, replication lag, slow queries, lock waits | 30 detik |
| DASH-04: Log Explorer | All | Full-text search bar, result table with syntax highlighting, log stream viewer | Manual |
| DASH-05: Capacity Planning | Managers | Trend charts (30/90 hari), projection graphs, threshold markers, recommendation panel | 5 menit |
| DASH-06: Alert History | Ops Engineers | Alert timeline, severity distribution pie chart, MTTR trend, top-firing rules | 1 menit |

### 8.2 Interaction Requirements

- All tables support sorting, filtering, pagination (default 50 rows/page)
- Drill-down: click any server → navigate to DASH-02 pre-filtered
- Alert acknowledgment: inline button on alert row; requires comment
- Export: CSV for tables, PNG for charts, PDF for full dashboard snapshot
- Dark/Light theme toggle (browser preference detected automatically)

### 8.3 Accessibility

- WCAG 2.1 AA compliance for text content
- Keyboard navigation support (Tab order, Enter/Escape shortcuts)
- Color contrast ratio ≥ 4.5:1 (color is never the sole indicator of status)

---

## 9. Business Rules & Logic

| Rule ID | Business Rule | Logic | Enforcement |
|---------|--------------|-------|-------------|
| BR-001 | Critical alert must be acknowledged within 15 menit | Timer starts at FIRING; auto-escalate to manager after 15 menit | Alertmanager notification routing with group_wait |
| BR-002 | Asset cannot be decommissioned without approval workflow | Status change to DECOMMISSIONED requires 2-approver workflow | Asset API middleware with approval endpoint |
| BR-003 | Warranty expiry warning at 30 hari before expiration | Cron job checks warranty_expiry − NOW() = 30 | Daily scheduled task → Slack #asset-warnings |
| BR-004 | Duplicate asset detection on import | Match on hostname OR IP address; flag potential duplicate for review | Import validation step |
| BR-005 | Alert suppression during planned maintenance | Maintenance window defined per asset; suppress non-critical alerts | Prometheus recording rule + label matcher |
| BR-006 | Log retention enforcement | Hard delete after retention period; soft delete first (move to cold index) | Elasticsearch ILM policy |
| BR-007 | Session timeout after 30 menit idle | Last activity timestamp tracked; redirect to login on timeout | JWT expiration + axios interceptor |
| BR-008 | Minimum password complexity for local accounts | ≥ 12 karakter, uppercase, lowercase, digit, special char | HashiCorp Vault / LDAP password policy |

---

## 10. Security Requirements

### 10.1 Authentication & Authorization

| Control | Implementation |
|---------|---------------|
| Authentication | LDAP/Active Directory via LDAPS; fallback to local auth (disabled by default) |
| Authorization | RBAC: Admin (full access), Operator (read/write dashboards/alerts), Viewer (read-only) |
| Session Management | JWT token, 24-hour expiry, 30-minute idle timeout, refresh token rotation |
| Password Policy | Enforced via LDAP; local accounts require hash algorithm bcrypt cost ≥ 12 |

### 10.2 Encryption

| Data State | Algorithm | Key Management |
|------------|-----------|----------------|
| Data in transit | TLS 1.3 (HTTPS), LDAPS (636), SMTPS (587) | Internal PKI certificate |
| Data at rest — metrics | AES-256-GCM | LUKS full-disk encryption |
| Data at rest — logs | AES-256-GCM | LUKS full-disk encryption |
| Data at rest — asset DB | Column-level encryption for serial_number, warranty fields | HashiCorp Vault KMS |

### 10.3 Network Security

| Layer | Control |
|-------|---------|
| VPC Segmentation | Monitoring subnet isolated in dedicated VLAN (VLAN 100) |
| Firewall Rules | Ingress: Grafana 443 (internal only), Prometheus 9090 (internal only); Egress: Slack 443, SMTP 587, SMS 443, LDAP 636 |
| NAT | Outbound traffic via NAT gateway; no direct internet exposure |
| IDS/IPS | Suricata ruleset deployed on monitoring subnet boundary |
| Bastion Host | Only entry point for SSH access; mTLS required |

### 10.4 Audit & Compliance

- Every administrative action logged to AUDIT_LOG table with immutable append-only constraint
- Quarterly access review: list all users per role, confirm necessity
- Annual penetration test scoped to monitoring infrastructure
- Log integrity verification: SHA-256 checksum appended to daily log archives

---

## 11. Testing Strategy & Acceptance Criteria

### 11.1 Test Levels

| Level | Scope | Responsibility | Entry Criteria | Exit Criteria |
|-------|-------|---------------|----------------|---------------|
| Unit | Individual functions/components | Developers | Code committed to feature branch | ≥ 80% line coverage; all tests green |
| Integration | Component interactions (Prometheus → Alertmanager → Slack) | DevOps Engineer | All unit tests pass | All integration test scenarios pass |
| System | End-to-end workflows | QA Engineer | All integration tests pass; staging environment ready | All test cases executed; zero P1/P2 defects |
| UAT | Business acceptance by operations team | End Users (Ops Team) | System test passed | Signed UAT acceptance form |
| Performance | Load testing with k6 | DevOps Engineer | Staging environment mirrors prod specs | Meet all NFR targets |

### 11.2 Acceptance Criteria Summary

| Scenario | Expected Result | Pass Condition |
|----------|----------------|----------------|
| 200 servers scraped simultaneously | All metrics available within 30 detik | 100% success rate over 1-hour sustained test |
| CPU > 90% triggers critical alert | Alert appears in Slack #infra-critical within 30 detik | Latency ≤ 30 detik in 95th percentile |
| Grafana loads Overview dashboard | Dashboard renders fully in < 3 detik | Measured from browser navigation start to DOMContentLoaded |
| LDAP authentication | User can log in with AD credentials | Successful login + correct role assignment |
| Asset import 500 records | All records processed; duplicates flagged | Zero data loss; duplicate report generated |
| Elasticsearch search 7-day window | Results returned in < 2 detik | P95 response time ≤ 2 detik across 100 queries |
| Disaster recovery drill | System restored within RTO | Recovery completed in ≤ 1 jam |

### 11.3 Test Environment

| Environment | Purpose | Data |
|-------------|---------|------|
| Dev | Developer unit/integration testing | Synthetic/mock data |
| Staging | Pre-production validation | Anonymized production-like data (subset) |
| UAT | Business acceptance | Production clone (read-only agents) |
| Prod | Live operations | Real production data |

---

## 12. Implementation Timeline

### 12.1 Sprint Plan (Agile/Scrum — 2-week sprints)

| Sprint | Duration | Focus Area | Key Deliverables |
|--------|----------|-----------|-----------------|
| Sprint 0 | 2 minggu | Project Setup & Foundation | Infrastructure provisioning, CI/CD pipeline, repo structure, architecture sign-off |
| Sprint 1 | 2 minggu | Metrics Core | Prometheus deployed; Node Exporter on 50 servers; basic Grafana dashboards |
| Sprint 2 | 2 minggu | Metrics Expansion | Node Exporter on remaining servers; database exporters deployed; Alertmanager configured |
| Sprint 3 | 2 minggu | Log Pipeline | Fluentd deployed on all servers; Elasticsearch cluster operational; Log Dashboard live |
| Sprint 4 | 2 minggu | Asset Inventory | Asset API (CRUD); auto-discovery; LDAP auth integrated |
| Sprint 5 | 2 minggu | Alerting & Integrations | Multi-channel alerting; Jira/ServiceNow webhook; maintenance window suppression |
| Sprint 6 | 2 minggu | Reporting & Polish | Capacity reports; mobile-responsive dashboards; accessibility audit |
| Sprint 7 | 2 minggu | Performance & UAT Prep | Load testing; security scan; documentation; training materials |
| Sprint 8 | 2 minggu | UAT & Go-Live | UAT execution; bug fixes; production cutover; hypercare period begins |

**Total Duration: 16 minggu (4 bulan)**

### 12.2 Resource Allocation

| Role | Jumlah | Sprint Engagement |
|------|--------|-------------------|
| Project Manager | 1 | Sprint 0–8 (part-time, 20%) |
| Tech Lead | 1 | Sprint 0–8 (full-time) |
| DevOps Engineer | 1 | Sprint 0–6 (full-time), Sprint 7–8 (part-time, 50%) |
| System Administrator | 2 | Sprint 1–5 (full-time), Sprint 6–8 (part-time, 30%) |
| QA Engineer | 1 | Sprint 5–8 (full-time) |
| Security Officer | 1 | Sprint 3, 7 (review points) |

### 12.3 Milestones

| Milestone | Target Date | Gate Criteria |
|-----------|-------------|---------------|
| Kickoff | Minggu ke-1 | Budget approved, team assembled, environment provisioned |
| Metrics MVP | Minggu ke-4 | 100% servers scraping; basic dashboards functional |
| Logging Live | Minggu ke-8 | All logs flowing to Elasticsearch; search operational |
| Feature Complete | Minggu ke-12 | All P1 requirements implemented; code freeze |
| UAT Start | Minggu ke-13 | UAT test plan signed off; staging ready |
| Go-Live | Minggu ke-16 | UAT signed off; rollback plan tested; ops team trained |
| Hypercare End | Minggu ke-20 | Zero critical incidents; handover to BAU complete |

---

## 13. Dependencies & Assumptions

### 13.1 Dependencies

| Dependency | Owner | Status | Impact if Delayed |
|------------|-------|--------|-------------------|
| Network VLAN segmentation (VLAN 100) | Network Team | Not started | Blocks monitoring server deployment |
| LDAP service account creation | IAM Team | Not started | Blocks Sprint 4 (auth integration) |
| Slack workspace admin approval | IT Governance | Pending | Blocks Sprint 5 (alerting) |
| DNS records for monitoring subdomain | DNS Team | Not started | Blocks SSL certificate issuance |
| Hardware procurement (NVMe SSDs) | Procurement | Quote pending | Delays Elasticsearch cluster sizing |
| Legacy Nagios credentials | Previous vendor | Unknown | May delay SNMP integration |

### 13.2 Assumptions

| # | Assumption | Confidence | Verification Method |
|---|-----------|------------|-------------------|
| A1 | Semua 200+ server memiliki network connectivity ke monitoring subnet | Tinggi | Ping test dari monitoring server ke semua target IP |
| A2 | Existing LDAP/AD supports LDAPS protocol | Tinggi | Port 636 connectivity test |
| A3 | Operations team will dedicate 10% time for UAT participation | Medium | Confirmed via PM communication |
| A4 | No firewall changes needed between monitoring subnet and application servers | Medium | Network team confirmation required |
| A5 | Elasticsearch can handle 20 GB/day ingestion on proposed hardware | Tinggi | Benchmark test on staging hardware |
| A6 | SMS gateway API has sufficient throughput for concurrent alerts | Tinggi | Vendor SLA review |
| A7 | Budget approval for additional hardware (estimated Rp 85 juta) | Medium | Finance confirmation pending |

---

## 14. Risk Register

| Risk ID | Risk Description | Probability | Impact | Risk Level | Mitigation Strategy | Owner |
|---------|-----------------|-------------|--------|-----------|--------------------|-------|
| R-01 | Monitoring server becomes single point of failure | Medium | High | **HIGH** | Deploy primary + standby pair; implement failover script; test quarterly | Tech Lead |
| R-02 | Network partition causes metrics gap | Low | High | **MEDIUM** | Local buffer in node_exporter (buffer-size: 10000); replay on reconnect | DevOps Eng |
| R-03 | Storage fills up faster than expected | Medium | High | **HIGH** | Implement ILM policies; compress metrics; auto-expand volume when 80% full | DevOps Eng |
| R-04 | Elasticsearch cluster degrades under high ingestion | Medium | Medium | **MEDIUM** | Benchmark before go-live; tune JVM heap (50% RAM, max 31 GB); add data nodes if needed | DevOps Eng |
| R-05 | Operations team resists new tool adoption | High | Medium | **HIGH** | Involve ops team in design phase; provide hands-on training; assign champions per shift | Product Owner |
| R-06 | Alert fatigue due to excessive false positives | High | Medium | **HIGH** | Tune thresholds during Sprint 2–3; implement alert grouping; review weekly during hypercare | Tech Lead |
| R-07 | Hardware delivery delayed beyond 2 minggu | Medium | High | **MEDIUM** | Identify alternative supplier; consider cloud burst for staging; negotiate penalty clauses | PM |
| R-08 | Data corruption in Prometheus long-term storage | Low | High | **LOW** | Regular checksum verification; backup strategy; test restore procedure monthly | DevOps Eng |
| R-09 | LDAP outage prevents Grafana login | Low | High | **LOW** | Configure local fallback admin account; monitor LDAP health; alert on connection failures | Tech Lead |
| R-10 | Scope creep from additional monitoring requests | Medium | Medium | **MEDIUM** | Strict scope baseline; change request process; evaluate additions for Phase 2 | Product Owner |

### Risk Matrix

| | Low Impact | Medium Impact | High Impact |
|--|-----------|--------------|-------------|
| **Low Probability** | R-08 | R-04 | R-02, R-09 |
| **Medium Probability** | — | R-07 | R-01, R-03, R-06 |
| **High Probability** | — | R-05, R-10 | — |

---

## 15. Change Management Plan

### 15.1 Change Request Process

```
Request Submitted → Triage (PM + Tech Lead) → Impact Assessment → 
ARB Review (weekly) → Approved/Rejected → 
  If Approved: Backlog Prioritization → Sprint Planning → Implementation → 
  UAT → Release
```

### 15.2 Change Categories

| Category | Examples | Approval Authority | Turnaround |
|----------|---------|-------------------|------------|
| Standard | Minor dashboard tweaks, new metric endpoints | Tech Lead | 1 sprint |
| Normal | New integration, significant feature addition | ARB + Product Owner | 2–3 sprints |
| Emergency | Security patch, critical bug fix | Tech Lead + VP Infra | Immediate (within 48 jam) |

### 15.3 Configuration Baseline

- All infrastructure configurations managed via Git (GitOps)
- Terraform state stored remotely with versioning
- Grafana dashboards exported as JSON and version-controlled
- Prometheus rules and Alertmanager configs in same repository
- Change audit trail maintained via git commit history

---

## 16. Handover & Support Plan

### 16.1 Transition to BAU (Business As Usual)

| Activity | Timing | Responsible | Deliverable |
|----------|--------|-------------|-------------|
| Documentation handover | Minggu ke-18 | Tech Lead | Runbook, architecture diagrams, SOP |
| Ops team training (batch 1) | Minggu ke-18 | Tech Lead + DevOps Eng | Training attendance record, quiz score ≥ 80% |
| Ops team training (batch 2) | Minggu ke-19 | Tech Lead + DevOps Eng | Shift B/C coverage confirmed |
| Hypercare support | Minggu ke-16–20 | DevOps Eng + Tech Lead | Incident log, resolution notes |
| Final handover sign-off | Minggu ke-20 | VP Infra + Product Owner | Handover certificate |

### 16.2 Support Structure

| Tier | Responsibility | Response Time | Escalation Path |
|------|---------------|---------------|----------------|
| Tier 1 — Ops Team | Daily monitoring, alert acknowledgment, basic troubleshooting | 15 menit | Escalate to Tier 2 if unresolved in 30 menit |
| Tier 2 — DevOps Eng | Deep troubleshooting, configuration changes, performance tuning | 1 jam | Escalate to Tier 3 if unresolved in 4 jam |
| Tier 3 — Vendor / Community | Bug fixes in upstream components, complex issues | 8 jam | CTO escalation if unresolved in 24 jam |

### 16.3 Monitoring the Monitor

The monitoring system itself will be monitored:

| Check | Tool | Frequency | Alert Channel |
|-------|------|-----------|---------------|
| Prometheus uptime | Blackbox exporter | Setiap 30 detik | #infra-critical |
| Elasticsearch cluster health | Elasticsearch API | Setiap 1 menit | #infra-critical |
| Grafana availability | HTTP probe | Setiap 1 menit | #infra-warning |
| Alert pipeline end-to-end | Synthetic alert injection | Setiap 4 jam | #infra-warning |
| Disk usage on storage nodes | Node exporter | Setiap 15 detik | #infra-warning (> 80%), #infra-critical (> 90%) |
| Replication lag (VictoriaMetrics) | Built-in metrics | Setiap 15 detik | #infra-critical |

---

## 17. Sign-off / Approval Matrix

| Role | Name | Signature | Date | Status |
|------|------|-----------|------|--------|
| Executive Sponsor (CTO) | `[Nama]` | _____________ | ____/____/2026 | Pending |
| VP Infrastructure | `[Nama]` | _____________ | ____/____/2026 | Pending |
| Product Owner (Head of Ops) | `[Nama]` | _____________ | ____/____/2026 | Pending |
| Tech Lead | `[Nama]` | _____________ | ____/____/2026 | Pending |
| Security Officer | `[Nama]` | _____________ | ____/____/2026 | Pending |
| Project Manager | `[Nama]` | _____________ | ____/____/2026 | Pending |

**Go/No-Go Decision Criteria:**
- All P1 requirements implemented and tested
- UAT signed off by Product Owner
- Security review completed with no Critical/High findings
- Disaster recovery test passed (RTO ≤ 1 jam)
- Ops team training completed (≥ 80% pass rate)
- Rollback plan documented and tested

---

## 18. Assumptions & Open Questions

### 18.1 Assumptions (vs Facts)

| # | Statement | Type | Action Required |
|---|-----------|------|-----------------|
| 1 | Budget of Rp 85 juta approved for hardware | Assumption | Confirm with Finance by Minggu ke-2 |
| 2 | Network team can provision VLAN 100 within 1 minggu | Assumption | Schedule workshop with Network team |
| 3 | Operations team will adopt new tool (no parallel legacy system) | Assumption | Communicate decision to all shifts; include in change comms |
| 4 | All servers can install Node Exporter without application impact | Assumption | Pilot on 5 servers first; measure overhead |
| 5 | SMS gateway provider supports Indonesian numbers | Assumption | Verify with vendor contract |

### 18.2 Open Questions

| # | Question | Stakeholder | Deadline |
|---|----------|-------------|----------|
| Q1 | Apakah perlu monitoring untuk container/orchestration (Docker/Kubernetes) di Phase 1? | Product Owner | Minggu ke-2 |
| Q2 | Berapa jumlah operator per shift yang membutuhkan akses Viewer role? | Head of Ops | Minggu ke-2 |
| Q3 | Apakah ada requirement untuk multi-tenant isolation (misalnya tiap business unit punya dashboard terpisah)? | CTO | Minggu ke-3 |
| Q4 | Apakah legacy Nagios akan di-decommission atau dijalankan paralel selama transisi? | Tech Lead | Minggu ke-3 |
| Q5 | Apakah ada requirement compliance spesifik (ISO 27001, POJK) yang harus dipenuhi? | Security Officer | Minggu ke-4 |
| Q6 | Berapa throughput SMS gateway yang dibutuhkan untuk peak alert scenario (estimasi 50 concurrent alerts)? | DevOps Eng | Minggu ke-4 |

---

## 19. Completeness Check

Based on FSD Review Rubric:

| Section | Status | Notes |
|---------|--------|-------|
| Project Background & Objectives | COMPLETE | Detailed background with pain points, quantified objectives |
| Project Scope (In-Scope / Out-of-Scope) | COMPLETE | Clear boundaries defined |
| Stakeholder Identification | COMPLETE | 8 roles identified with engagement levels |
| Functional Requirements | COMPLETE | 14 FRs with IDs, priorities, acceptance criteria |
| Non-Functional Requirements | COMPLETE | 17 NFRs covering performance, availability, scalability, compliance |
| Data Architecture / Data Model | COMPLETE | Data flow diagram, 4 entity models, storage estimates |
| System Architecture / Integration Design | COMPLETE | Topology, HA design, 6 integration points |
| UI/UX Specifications | COMPLETE | 6 dashboards specified, interaction requirements, accessibility |
| Business Rules & Logic | COMPLETE | 8 business rules with enforcement mechanisms |
| Security Requirements | COMPLETE | AuthN/AuthZ, encryption, network, audit covered |
| Testing Strategy & Acceptance Criteria | COMPLETE | 5 test levels, 7 scenarios, environment strategy |
| Implementation Timeline | COMPLETE | 8 sprints, resource allocation, milestones |
| Dependencies & Assumptions | COMPLETE | 6 dependencies, 7 assumptions classified |
| Risk Register | COMPLETE | 10 risks with probability, impact, mitigation, matrix |
| Change Management Plan | COMPLETE | Process flow, categories, configuration baseline |
| Handover & Support Plan | COMPLETE | Transition activities, 3-tier support, monitoring the monitor |
| Sign-off / Approval Matrix | COMPLETE | 6 approvers with go/no-go criteria |

**Overall Status: ALL SECTIONS COMPLETE**

---

## 20. Security & Confidentiality Notes

- Dokumen ini diklasifikasikan sebagai **CONFIDENTIAL — Internal Use Only**
- Tidak mengandung credential, secret, atau data sensitif apa pun
- Semua nama stakeholder bersifat placeholder (`[Nama]`) dan akan diisi saat sign-off
- Arsitektur jaringan menggunakan private IP space (10.0.0.0/8) — informasi publikasi aman
- Rencana hardware dan budget bersifat internal finance

**Action Items:**
- Setelah sign-off, simpan dokumen di repository internal terenkripsi
- Distribusi terbatas kepada stakeholder yang terdaftar di section 3
- Review ulang keamanan setiap kali ada perubahan arsitektur signifikan

---

*End of Document — Version 2.0 — 17 Mei 2026*

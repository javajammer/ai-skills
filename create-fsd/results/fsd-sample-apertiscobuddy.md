# Functional Specification Design (FSD)
## SAMAHI — Sistem Manajemen Aset dan Monitoring Infrastruktur

**Doc Type**: FSD (Functional Specification Design)
**Version**: 2.0
**Status**: Draft (Review-ready)
**Date**: 16 Mei 2026
**Owner/Authoring Team**: Tim Teknologi PT Nusantara Digital
**Methodology**: Agile (Scrum)
**Primary Audience**: VP Infrastructure, CTO, Tech Lead, Operations Lead, Security Lead, Project Manager

---

## 0. Project Context

### 0.1 Project Name
SAMAHI — Sistem Manajemen Aset dan Monitoring Infrastruktur.

### 0.2 Background / Problem Statement
PT Nusantara Digital mengoperasikan >200 server fisik dan virtual yang tersebar di 3 data center (Jakarta, Surabaya, Manado). Monitoring infrastruktur saat ini dilakukan manual dengan spreadsheet yang diperbarui mingguan, plus alert email dari Nagios yang tidak selalu terbaca.

Dampak utama current state:
- Deteksi anomali terlambat (rata-rata 4–6 jam sebelum diketahui)
- Tidak ada histori performa untuk analisis tren dan post-incident analysis
- Capacity planning lemah (kurang data time-series)
- Risiko human error dalam pencatatan dan eskalasi
- Log tersebar (tidak ada centralized logging), sehingga MTTR memburuk

### 0.3 Goals & Success Metrics
**Business/Operational Goals**
1. Monitoring otomatis untuk seluruh infrastruktur on-premise (server & database).
2. Dashboard real-time untuk tim operasi dan manajemen.
3. Alerting otomatis via chat (Slack) dan proses eskalasi yang jelas.

**Quantitative Success Metrics**
- **MTTD** (Mean Time To Detect) anomali turun dari 4 jam → **< 15 menit**.
- **Coverage**: 100% server target dan 100% DB target ter-monitor (lihat scope).
- **Alert delivery reliability**: >99% alert kritikal terkirim ke Slack channel yang ditetapkan.
- **Data availability**: metrik tersedia untuk query dashboard minimal 99.5% pada jam kerja.

### 0.4 Stakeholders & RACI (high level)
| Stakeholder | Role | Responsibility |
|---|---|---|
| VP Infrastructure | Executive Sponsor | Approve scope, budget, risk acceptance |
| CTO | Executive Approver | Final sign-off, alignment dengan roadmap teknologi |
| Tech Lead (Platform/Infra) | Technical Owner | Arsitektur, kualitas implementasi, DoD |
| Operations Lead / NOC Lead | Business Owner (Ops) | Kebutuhan operasional, SOP incident, UAT |
| Security Lead | Security Approver | Review akses, logging policy, hardening |
| Project Manager | Delivery Owner | Timeline, dependency, komunikasi, change control |

### 0.5 Timeline & Milestones (baseline)
Total estimasi: **12 minggu (±3 bulan)**.

| Phase | Duration | Milestone / Exit Criteria |
|---|---:|---|
| Setup monitoring servers | 1 minggu | Environment siap (OS, network, DNS, TLS) |
| Install Prometheus + exporters | 2 minggu | 80% server onboarded + baseline dashboards |
| Setup logging (Fluentd → Elasticsearch) | 2 minggu | Log dari 80% server masuk + query dasar berjalan |
| Dashboard development (Grafana) | 3 minggu | Dashboard ops & mgmt tersedia + role-based access |
| Alerting configuration | 1 minggu | Rules P1/P2 + routing + on-call channel valid |
| UAT & tuning | 2 minggu | Pass criteria terpenuhi (lihat Testing & Acceptance) |
| Go-live | 1 minggu | Cutover + handover + operational readiness |

---

## 1. Scope

### 1.1 In Scope
1. Monitoring server Linux dan Windows (200+ server).
2. Monitoring database: PostgreSQL, MySQL, MongoDB.
3. Dashboard real-time (grafik, heatmap, scorecard) menggunakan Grafana.
4. Alerting via webhook ke Slack (routing, severity, deduplication).
5. Centralized log aggregation dari seluruh server untuk pencarian dan korelasi incident.
6. Asset inventory minimum untuk target monitoring (ID aset, lokasi DC, owner service, environment).

### 1.2 Out of Scope
1. Application Performance Monitoring (APM) end-user (tracing, RUM).
2. Security scanning / vulnerability assessment.
3. Backup monitoring.
4. Cloud resource monitoring (saat ini semua on-premise).

### 1.3 Target Environment Coverage
| Component | Target |
|---|---|
| Linux servers | ±120 (Ubuntu 20.04/22.04) |
| Windows servers | ±80 (Windows Server 2019) |
| PostgreSQL | ±15 instance |
| MySQL | ±8 instance |
| MongoDB | ±5 replica set |
| Data centers | Jakarta, Surabaya, Manado |

---

## 2. Current State (As-Is)

| Capability | Current | Pain Point |
|---|---|---|
| Server monitoring | Spreadsheet manual | Data stale, human error, tidak real-time |
| DB monitoring | Manual check (pg_stat_activity dsb.) | Tidak scalable, tidak ada alert konsisten |
| Log management | Tidak ada centralized logging | Troubleshooting lambat, bukti incident sulit |
| Alerting | Email dari Nagios | Mudah terlewat, routing/eskalasi tidak konsisten |
| Dashboard | Tidak ada | Tidak ada single source of truth |

---

## 3. Proposed Solution (To-Be)

### 3.1 Solution Overview
Stack yang diusulkan:
- **Metrics collection**: Prometheus + exporters (node exporter, windows exporter, DB exporters)
- **Alerting**: Alertmanager → Slack webhook (routing per severity/team)
- **Visualization**: Grafana
- **Metrics storage**: TimescaleDB (time-series)
- **Log ingestion**: Fluentd
- **Log storage/search**: Elasticsearch
- **Deployment model**: Docker Compose pada dedicated monitoring servers

**Catatan**: Dokumen ini adalah FSD; detail desain low-level, sizing final, dan runbook deployment akan diperdalam pada TSD (jika dibuat terpisah).

### 3.2 High-Level Architecture & Integrations
**Actors / Systems**:
- Infra Nodes (Linux/Windows)
- Database nodes (PostgreSQL/MySQL/MongoDB)
- Monitoring servers (Prometheus, Alertmanager, Grafana, Fluentd, Elasticsearch, TimescaleDB)
- Slack workspace (channel ops + on-call)
- LDAP/Directory Service untuk autentikasi dashboard

**Integration points**:
- LDAP auth untuk Grafana
- Slack incoming webhook untuk alert notifications

### 3.3 Data Flow (Logical)
1. Exporters di tiap node expose metrics.
2. Prometheus scrape metrics periodik (default 15 detik) dan menyimpan/forward ke storage.
3. Fluentd mengumpulkan log (file/syslog/eventlog) dan mengirim ke Elasticsearch.
4. Grafana query TimescaleDB (metrics) dan Elasticsearch (logs) untuk dashboard.
5. Alertmanager mengevaluasi alert rules (via Prometheus) dan mengirim notifikasi ke Slack sesuai routing.

---

## 4. Functional Requirements (FR)

### 4.1 Requirements Table
| ID | Requirement | Priority | Acceptance Criteria | Owner |
|---|---|---|---|---|
| FR-001 | Sistem dapat melakukan discovery & onboarding host untuk monitoring (Linux/Windows) berdasarkan daftar aset (CSV/Sheet export) | P1 | Given daftar aset valid, When di-import, Then host muncul di inventory dan status onboarding tercatat (success/failed + reason) | Eng |
| FR-002 | Sistem mengumpulkan metrik resource server (CPU, memory, disk, IO, network) | P1 | Given host onboarded, When 5 menit berjalan, Then metrik tersedia di Grafana dengan resolusi 15 detik | Eng |
| FR-003 | Sistem mengumpulkan metrik database (connections, query performance/slow queries indicator, replication lag bila ada) | P1 | Given DB exporter aktif, When dashboard DB dibuka, Then metrik utama tampil dan dapat difilter per instance | Eng |
| FR-004 | Sistem menyediakan dashboard Overview (status per DC, top offenders, SLA-style scorecard) | P1 | Given data masuk, When Overview dibuka, Then menampilkan ringkasan per DC + daftar alert aktif + tren 24 jam | Product/Ops |
| FR-005 | Sistem menyediakan dashboard per-server (drill-down) | P1 | Given host dipilih, When detail dibuka, Then tampil CPU/mem/disk/io/net + event alert 24 jam terakhir | Ops |
| FR-006 | Sistem menyediakan dashboard database | P1 | Given DB dipilih, When dashboard dibuka, Then tampil connections, latency proxy metric, replication lag (jika tersedia), errors | Ops |
| FR-007 | Sistem melakukan centralized logging dan mendukung pencarian log full-text + filter (host, service, DC, time range) | P1 | Given log sources aktif, When user query, Then hasil muncul <5 detik untuk window 15 menit (p95) | Eng/Ops |
| FR-008 | Sistem mengirim alert ke Slack dengan severity (Warning/Critical) dan deduplication | P1 | Given rule terpenuhi, When alert fired, Then pesan Slack berisi host, metric, nilai, threshold, timestamp, runbook link; alert tidak spam (dedup aktif) | Ops |
| FR-009 | Sistem mendukung routing alert per DC / team / service owner | P2 | Given label routing, When alert fired, Then alert masuk channel yang sesuai (mis. #ops-jakarta) | Ops |
| FR-010 | Sistem menyimpan histori metrik dan log sesuai retention policy | P1 | Given kebijakan retention, When data melewati batas, Then data terhapus/diroll sesuai policy tanpa mengganggu ingestion | Eng |
| FR-011 | Sistem menyediakan audit log akses dashboard (login, view/export bila ada) | P2 | Given user login via LDAP, When akses terjadi, Then event audit tercatat dan dapat dicari | Security |
| FR-012 | Sistem menyediakan export laporan periodik (mingguan) untuk uptime/availability komponen infra (ringkas) | P3 | Given jadwal, When waktu terpenuhi, Then report (PDF/CSV) tersedia atau dikirim ke channel/email internal | Ops |

### 4.2 Business Rules & Alerting Logic (ringkas)
- Severity:
  - **Warning**: degradasi, perlu investigasi tapi tidak mengancam layanan saat ini.
  - **Critical**: berpotensi/terjadi outage atau risk tinggi.
- Deduplication:
  - Alert yang sama (host+metric+label) dalam window tertentu digabung.
- Routing:
  - Berdasarkan label: `dc`, `env`, `service_owner`, `severity`.

### 4.3 Default Thresholds (baseline)
| Metric | Warning | Critical | Notes |
|---|---:|---:|---|
| CPU usage | >70% | >90% | window 5 menit, exclude spikes <1m |
| Memory usage | >75% | >90% | include cache policy per OS |
| Disk usage | >80% | >90% | per mountpoint, exclude tmpfs |
| Disk IO wait | >20ms | >50ms | p95 5 menit |
| Network latency (intra-DC) | >5ms | >20ms | perlu definisi probe target |

---

## 5. Non-Functional Requirements (NFR)

| ID | Requirement | Target | Priority | Acceptance Criteria | Owner |
|---|---|---|---|---|---|
| NFR-001 | Availability platform monitoring (Grafana + Prometheus query path) | 99.5% jam kerja | P1 | p95 downtime jam kerja <= 0.5%/bulan | Eng |
| NFR-002 | Data freshness metrics | <= 60 detik (p95) | P1 | p95 delay ingest-to-dashboard <= 60s | Eng |
| NFR-003 | Log search performance | p95 < 5 detik untuk 15 menit window | P1 | Beban uji memenuhi target p95 | Eng |
| NFR-004 | Retention metrics | 90 hari | P1 | Data metrics tersimpan dan dapat diquery untuk 90 hari | Eng |
| NFR-005 | Retention logs | 30 hari | P1 | Data logs tersimpan dan dapat dicari untuk 30 hari | Eng |
| NFR-006 | Access control | LDAP + RBAC | P1 | User role membatasi akses dashboard (viewer/editor/admin) | Security |
| NFR-007 | Auditability | Audit log akses tersedia 180 hari | P2 | Event login/akses tercatat dan disimpan | Security |
| NFR-008 | Backup & restore konfigurasi | RPO 24 jam, RTO 4 jam (config only) | P2 | Restore konfigurasi Grafana/Alert rules berhasil pada uji DR | Eng |

---

## 6. Data Architecture (Logical)

### 6.1 Data Domains
1. **Asset Inventory** (minimal): host/instance, DC, environment, owner, criticality.
2. **Metrics**: time-series (CPU, memory, disk, DB stats).
3. **Logs**: semi-structured text + metadata (host, service, dc, severity).
4. **Alert Events**: fired/resolved, labels, routing, ack notes (jika ada).

### 6.2 Logical Data Model (minimum fields)
**Asset**
- `asset_id`, `hostname`, `ip`, `dc`, `env`, `os`, `owner_team`, `service_name`, `criticality`

**Metric Series (conceptual)**
- `metric_name`, `labels{...}`, `timestamp`, `value`

**Log Event (conceptual)**
- `timestamp`, `host`, `service`, `dc`, `env`, `message`, `severity`, `tags[]`

**Alert Event (conceptual)**
- `alert_name`, `labels{...}`, `status(firing/resolved)`, `startsAt`, `endsAt`, `notified_channel`

---

## 7. UI/UX Specifications (Dashboard)

### 7.1 Overview Dashboard (Ops + Mgmt)
Komponen minimum:
- Status ringkas per DC: jumlah host up/down, alert firing by severity
- Top 10 host dengan CPU/mem/disk tertinggi (rolling 15 menit)
- Trend incident: jumlah alert critical per hari (7/30 hari)
- Scorecard availability (berbasis uptime exporter dan scrape success)

### 7.2 Drill-down Dashboard
- Filter: DC, environment, owner team, service
- Per host: CPU/mem/disk/io/net + event timeline alert

### 7.3 Database Dashboard
- Connections, slow query indicator (proxy metric), replication lag (jika replika)
- Error rate (dari log query) bila tersedia

### 7.4 Log Search Dashboard
- Search bar (full-text)
- Filter facet: host, dc, service, severity
- Time picker default 15 menit

---

## 8. Security Requirements

### 8.1 Access Control & Authentication
- Grafana menggunakan **LDAP authentication**.
- RBAC minimal:
  - **Viewer** (Ops): view dashboard, query logs.
  - **Editor** (Senior Ops/Eng): edit dashboard, bukan admin.
  - **Admin** (Platform/Infra): manage datasources, plugins, user mapping.

### 8.2 Network & Hardening
- Prometheus/Grafana/Elasticsearch hanya dapat diakses dari internal network.
- Akses antar komponen menggunakan TLS bila memungkinkan (prioritas P2 jika keterbatasan existing infra).

### 8.3 Data Sensitivity
- Asumsi input: logs tidak mengandung PII karena sudah difilter di aplikasi sumber.
- Jika ditemukan PII di log, wajib:
  - update filter pipeline Fluentd
  - batasi akses query log
  - definisikan retensi lebih ketat untuk index terkait

---

## 9. Testing Strategy & Acceptance Criteria

### 9.1 Test Types
- **Functional Testing**: onboarding, scraping, dashboard rendering, alert routing.
- **Performance Testing**: log search latency, dashboard query latency.
- **Reliability Testing**: failover monitoring server (jika diterapkan), restart behavior.
- **Security Testing**: RBAC verification, LDAP integration, audit log verification.

### 9.2 UAT Entry/Exit Criteria
**Entry**
- 80% aset target onboarded.
- Dashboard Overview dan per-host tersedia.
- Minimal 10 alert rules aktif (mix warning/critical).

**Exit (Pass/Fail)**
- MTTD simulasi (inject CPU stress/disk fill) menghasilkan alert di Slack dalam **<= 15 menit**.
- Log dari minimal 80% host dapat dicari dan difilter.
- RBAC bekerja (viewer tidak bisa edit dashboard).
- Ops lead menandatangani UAT hasil.

---

## 10. Implementation Plan (Agile)

### 10.1 Sprint Plan (baseline 6 sprints @2 minggu)
- **Sprint 1**: setup environment + baseline Prometheus/Grafana
- **Sprint 2**: onboarding Linux + exporter hardening + inventory import
- **Sprint 3**: onboarding Windows + dashboard ops v1
- **Sprint 4**: logging pipeline + log search dashboard v1
- **Sprint 5**: alert rules + routing + runbook links
- **Sprint 6**: UAT + tuning + operational readiness + go-live

### 10.2 Definition of Done (DoD)
- Requirement diterjemahkan menjadi user story, memiliki acceptance criteria.
- Dashboard/rule memiliki owner dan runbook link.
- Konfigurasi tersimpan di repo (Git) dan dapat direstore.
- Monitoring server hardening minimum diterapkan.
- Dokumentasi handover + SOP incident tersedia.

---

## 11. Dependencies, Assumptions, and Constraints

### 11.1 Dependencies
- Ketersediaan dedicated monitoring server(s) dan akses network ke semua DC.
- LDAP/Directory service tersedia dan ada group mapping.
- Slack webhook dapat dibuat dan disetujui oleh admin Slack.

### 11.2 Constraints
- Semua environment on-premise (tidak include cloud monitoring).
- Deployment menggunakan Docker Compose (bukan Kubernetes) pada baseline.

---

## 12. Risk Register

| Risk | Impact | Likelihood | Mitigation |
|---|---|---:|---|
| Monitoring server down | Single point of failure → tidak ada visibility | Medium | Terapkan 2 server (primary/standby) + backup config + runbook failover |
| Network partition antar DC | Gap data metrics/logs | Medium | Buffering/queue pada log forwarder + alert untuk scrape failures |
| Storage penuh | Data metrics/logs hilang, cluster tidak stabil | Medium | Retention policy + ILM index lifecycle + disk alert pre-emptive |
| Log berisi data sensitif | Compliance risk | Low-Med | Filtering pipeline + access restriction + audit review |
| Alert fatigue | Ops mengabaikan alert | Medium | Tuning threshold, grouping, label hygiene, review mingguan |

---

## 13. Change Management Plan
- Semua perubahan scope/retention/threshold mengikuti proses change request oleh PM.
- Backlog diprioritaskan oleh Ops Lead + Tech Lead.
- Perubahan alert rules major harus melalui review mingguan (Ops + Eng + Security bila relevan).

---

## 14. Handover & Support Plan

### 14.1 Ownership After Go-Live
- **Primary Owner**: Operations/NOC (operasional harian, response)
- **Platform Owner**: Infra/DevOps (perubahan konfigurasi platform, upgrade, capacity)

### 14.2 Support Model
- On-call channel Slack untuk P1.
- Runbook minimal untuk alert critical (CPU, disk, memory, scrape failures, Elasticsearch cluster health).

### 14.3 Operational Readiness Checklist (minimum)
- Backup & restore konfigurasi diuji.
- Akses RBAC diverifikasi.
- Retention dan ILM diaktifkan.
- Dashboard mgmt + ops diset.

---

## 15. Approval Matrix (Sign-off)

| Role | Name | Status |
|---|---|---|
| VP Infrastructure | [REDACTED] | Draft |
| Tech Lead | [REDACTED] | Draft |
| CTO | [REDACTED] | Draft |
| Operations Lead | [REDACTED] | Draft |
| Security Lead | [REDACTED] | Draft |

---

## 16. Methodology Notes (Agile/Scrum alignment)

Mapping terhadap *Methodology Compliance Matrix* (Agile/Scrum):
- Sprint planning: disediakan sprint plan baseline (6 sprint) dan exit criteria per sprint.
- Backlog: requirement table FR/NFR menjadi input backlog dan prioritas (P1/P2/P3).
- Definition of Done: DoD didefinisikan eksplisit.
- Cross-functional: kebutuhan peran (Infra/DevOps/PM/Ops/Security) dideklarasikan.
- Retrospectives: aksi tuning alert dan review mingguan dimasukkan sebagai proses berkelanjutan.

---

## 17. Assumptions & Open Questions

### 17.1 Assumptions (explicit)
1. Slack adalah platform chat utama untuk alerting (Teams disebut sebagai opsi tetapi baseline Slack).
2. Logs dari aplikasi sumber sudah memfilter PII (akan diverifikasi).
3. TimescaleDB dan Elasticsearch dapat dioperasikan on-premise dengan resource yang memadai.
4. Dedicated monitoring server tersedia minimal 1; opsi HA (2 server) tergantung budget.

### 17.2 Open Questions (harus dikonfirmasi)
1. Apakah ada kebutuhan **SLA incident response** (mis. P1 acknowledge <= 10 menit) dan siapa on-call owner per DC?
2. Apakah perlu integrasi tambahan selain Slack (email fallback, Teams, ticketing seperti Jira/ServiceNow)?
3. Definisi **network latency probe target**: antar host mana yang dijadikan baseline (gateway, core switch, antar DC)?
4. Apakah dibutuhkan multi-tenancy (pemisahan dashboard per tim) di Grafana?
5. Apakah diperlukan HA untuk Elasticsearch/TimescaleDB (cluster) atau single node cukup untuk fase awal?
6. Kebijakan compliance internal untuk log retention: 30 hari sudah sesuai atau harus berbeda?

---

## 18. Completeness Check (FSD Rubric)

| Section (Rubric) | Status | Notes |
|---|---|---|
| Project Background & Objectives | COMPLETE | Ada goals + success metrics |
| Project Scope | COMPLETE | In/Out scope jelas |
| Stakeholder Identification | COMPLETE | Stakeholder + RACI high level |
| Functional Requirements | COMPLETE | FR table + acceptance criteria |
| Non-Functional Requirements | COMPLETE | NFR target + acceptance |
| Data Architecture / Data Model | COMPLETE | Logical model minimum |
| System Architecture / Integration Design | COMPLETE | High-level + data flow |
| UI/UX Specifications | COMPLETE | Dashboard sections minimum |
| Business Rules & Logic | COMPLETE | Severity/dedup/routing |
| Security Requirements | COMPLETE | LDAP/RBAC/network/data |
| Testing Strategy & Acceptance Criteria | COMPLETE | Test types + UAT criteria |
| Implementation Timeline | COMPLETE | Timeline + sprint plan |
| Dependencies & Assumptions | COMPLETE | Dependencies + assumptions + questions |
| Risk Register | COMPLETE | Risiko + mitigasi |
| Change Management Plan | COMPLETE | Proses CR + review |
| Handover & Support Plan | COMPLETE | Ownership + readiness |
| Sign-off / Approval Matrix | COMPLETE | Role + status |

---

## 19. Security & Confidentiality Notes
- Dokumen ini berisi informasi operasional internal (topologi logical, tooling). Klasifikasi: **CONFIDENTIAL**.
- Tidak ada credential/secret dicantumkan. Semua nama individu diredaq dengan **[REDACTED]**.
- Jika pada implementasi ditemukan token/webhook/credential tercatat di repo atau dashboard, lakukan rotasi credential dan audit akses.

*End of Document*

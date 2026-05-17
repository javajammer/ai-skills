# Functional Specification Design (FSD)
## Sistem Manajemen Aset dan Monitoring Infrastruktur (SAMAHI)

**Versi:** 1.0
**Tanggal:** 10 Mei 2026
**Disusun oleh:** Tim Teknologi PT Nusantara Digital
**Status:** Draft

---

## 1. Introduction

### 1.1 Background

PT Nusantara Digital mengoperasikan lebih dari 200 server fisik dan virtual
yang tersebar di 3 data center (Jakarta, Surabaya, dan Manado). Saat ini
pemantauan infrastruktur dilakukan secara manual menggunakan spreadsheet
yang diperbarui setiap minggu oleh tim operasi.

Proses manual ini menyebabkan:
- Keterlambatan deteksi anomali (rata-rata 4-6 jam sebelum diketahui)
- Tidak adanya histori performa untuk analisis tren
- Kesulitan dalam capacity planning
- Human error dalam pencatatan

### 1.2 Project Objectives

1. Membangun sistem monitoring otomatis untuk seluruh infrastruktur
2. Menyediakan dashboard real-time untuk tim operasi dan manajemen
3. Mengimplementasikan alerting otomatis via chat (Slack/Teams)
4. Mengurangi time-to-detect anomali dari 4 jam menjadi < 15 menit

### 1.3 Scope

**In Scope:**
1. Monitoring server Linux dan Windows (200+ server)
2. Monitoring database (PostgreSQL, MySQL, MongoDB)
3. Dashboard real-time (grafik, heatmap, scorecard)
4. Alerting via webhook ke Slack
5. Log aggregation dari seluruh server

**Out of Scope:**
1. Monitoring aplikasi end-user (APM)
2. Security scanning dan vulnerability assessment
3. Backup monitoring
4. Cloud resource monitoring (saat ini semua on-premise)

---

## 2. Current State

Saat ini, monitoring dilakukan dengan:

| Komponen | Status |
|---|---|
| Server Monitoring | Manual (spreadsheet) |
| Database Monitoring | pg_stat_activity (manual check) |
| Log Management | Tidak ada centralized logging |
| Alerting | Email dari Nagios (kadang tidak terbaca) |
| Dashboard | Tidak ada |

---

## 3. Proposed Solution

### 3.1 Architecture Overview

Sistem akan dibangun menggunakan stack berikut:
- **Data Collection:** Prometheus (metrics) + Fluentd (logs)
- **Storage:** TimescaleDB (time-series) + Elasticsearch (logs)
- **Visualization:** Grafana (dashboard)
- **Alerting:** Alertmanager → Slack Webhook
- **Orchestration:** Docker Compose di dedicated monitoring server

### 3.2 Monitoring Scope

Server yang akan dimonitor meliputi:
- 120 Linux servers (Ubuntu 20.04/22.04)
- 80 Windows Server 2019
- 15 PostgreSQL instances
- 8 MySQL instances
- 5 MongoDB replica sets

### 3.3 Alert Threshold

| Metric | Warning | Critical |
|---|---|---|
| CPU Usage | > 70% | > 90% |
| Memory Usage | > 75% | > 90% |
| Disk Usage | > 80% | > 90% |
| Disk I/O Wait | > 20ms | > 50ms |
| Network Latency | > 5ms | > 20ms |

### 3.4 Dashboard Design

Dashboard akan terdiri dari:
1. **Overview Dashboard** — Status seluruh infrastruktur
2. **Server Detail Dashboard** — Per-server metrics
3. **Database Dashboard** — Query performance, connections, replication lag
4. **Log Search Dashboard** — Full-text search logs

---

## 4. Data Flow

1. Prometheus scrapes metrics dari setiap server setiap 15 detik
2. Fluentd mengirim logs ke Elasticsearch
3. Grafana mengquery TimescaleDB dan Elasticsearch
4. Alertmanager memeriksa rules dan mengirim ke Slack

---

## 5. Implementation Plan

### 5.1 Timeline

| Fase | Durasi | Target |
|---|---|---|
| Setup Monitoring Server | 1 minggu | Minggu ke-1 |
| Install Prometheus + Exporters | 2 minggu | Minggu ke-2 s/d ke-3 |
| Setup Elasticsearch + Fluentd | 2 minggu | Minggu ke-4 s/d ke-5 |
| Dashboard Development | 3 minggu | Minggu ke-6 s/d ke-8 |
| Alerting Configuration | 1 minggu | Minggu ke-9 |
| UAT & Tuning | 2 minggu | Minggu ke-10 s/d ke-11 |
| Go-Live | 1 minggu | Minggu ke-12 |

Total estimasi: **12 minggu (3 bulan)**

### 5.2 Resources

| Role | Jumlah | Durasi |
|---|---|---|
| Infrastructure Engineer | 2 orang | Full-time selama project |
| DevOps Engineer | 1 orang | Full-time bulan ke-2 dan ke-3 |
| Project Manager | 1 orang | Part-time |

---

## 6. Security Considerations

- Akses ke monitoring dashboard menggunakan LDAP authentication
- Prometheus dan Grafana hanya bisa diakses dari internal network
- Logs tidak mengandung data sensitif (PII sudah difilter di aplikasi source)

---

## 7. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Monitoring server down | Tidak ada monitoring sama sekali | Deploy di 2 server (primary + standby) |
| Network partition | Metrics tidak sampai ke Prometheus | Local buffer di node exporter |
| Storage penuh | Metrics/logs hilang | Retention policy 90 hari metrics, 30 hari logs |

---

## 8. Approval

| Role | Name | Status |
|---|---|---|
| VP Infrastructure | [待定] | Pending |
| Tech Lead | [待定] | Pending |
| CTO | [待定] | Pending |

---

*End of Document*

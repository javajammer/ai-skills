# Functional Specification Design (FSD)
## Sistem Manajemen Aset dan Monitoring Infrastruktur (SAMAHI)

**Organisasi:** PT Nusantara Digital  
**Tipe Dokumen:** FSD  
**Versi:** 1.0  
**Tanggal:** 17 Mei 2026  
**Metodologi:** Agile (Scrum, 6 sprint x 2 minggu)  
**Audiens Utama:** VP Infrastructure & Security, CTO, Head of IT Operations, Product Owner Internal  
**Klasifikasi Informasi:** Confidential - Internal Use Only

---

## 1. Executive Summary

PT Nusantara Digital membutuhkan kapabilitas monitoring infrastruktur yang terpusat, real-time, dan dapat ditindaklanjuti untuk mendukung operasi lebih dari 200 server di tiga data center: Jakarta, Surabaya, dan Manado. Kondisi saat ini masih bergantung pada spreadsheet manual dan pengecekan ad hoc, yang menyebabkan keterlambatan deteksi anomali 4-6 jam, tidak tersedianya histori performa, kesulitan capacity planning, serta tingginya risiko human error.

Dokumen FSD ini mendefinisikan kebutuhan fungsional dan non-fungsional untuk membangun SAMAHI sebagai platform monitoring dan observability operasional tahap pertama. Solusi akan menyediakan dashboard real-time, alerting otomatis berbasis webhook ke Slack, pengumpulan log terpusat, dan monitoring host serta database. Arsitektur yang diusulkan menggunakan Prometheus, Fluentd, TimescaleDB, Elasticsearch, Grafana, dan Alertmanager yang dijalankan pada server monitoring dedicated dengan desain primary-warm standby.

Target bisnis utama adalah menurunkan waktu deteksi awal insiden (time-to-detect/TTD) menjadi kurang dari 15 menit, meningkatkan kualitas keputusan operasi melalui histori dan tren, serta menstandardisasi proses respons insiden lintas data center dalam horizon implementasi 12 minggu.

---

## 2. Konteks Proyek

### 2.1 Nama Proyek
Sistem Manajemen Aset dan Monitoring Infrastruktur (SAMAHI)

### 2.2 Latar Belakang
PT Nusantara Digital mengoperasikan infrastruktur on-premise skala menengah-besar yang melayani sistem internal dan layanan bisnis inti. Monitoring saat ini belum terstandardisasi, sebagian besar berbasis spreadsheet mingguan dan pengecekan manual oleh tim operasi. Mekanisme tersebut tidak memadai untuk kebutuhan operasi 24x7 karena keterlambatan deteksi, tidak adanya telemetry historis, dan keterbatasan eskalasi otomatis.

### 2.3 Tujuan Proyek
1. Menyediakan satu sumber kebenaran operasional untuk kesehatan server, database, dan log infrastruktur.
2. Menyediakan dashboard real-time untuk kebutuhan operasi harian dan ringkasan eksekutif.
3. Mengimplementasikan alerting otomatis berbasis severity ke kanal kolaborasi tim operasi.
4. Menurunkan TTD anomali dari 4-6 jam menjadi kurang dari 15 menit.
5. Menyediakan data historis untuk capacity planning, analisis tren, dan post-incident review.

### 2.4 Stakeholder

| Kelompok Stakeholder | Peran | Kepentingan Utama | Representasi Owner |
|---|---|---|---|
| VP Infrastructure & Security | Sponsor eksekutif | Keandalan operasi, risiko, ROI | Executive Sponsor |
| CTO | Sponsor teknologi | Keselarasan arsitektur dan standardisasi platform | Technology Sponsor |
| Head of IT Operations | Pemilik proses operasi | Efektivitas monitoring, respons insiden, runbook | Service Owner |
| Infrastructure Operations Team | Pengguna utama | Visibilitas real-time server, jaringan dasar, storage | Operations Lead |
| Database Administration Team | Pengguna utama | Kesehatan database, kapasitas, replication lag | DBA Lead |
| Security & Compliance Team | Reviewer | Akses, logging, retensi data, pemfilteran PII | Security Manager |
| Enterprise Architecture / Tech Lead | Reviewer teknis | Integrasi, maintainability, operability | Solution Architect |
| PMO / Scrum Master | Pengendali delivery | Timeline, dependency, risiko, ritme sprint | Scrum Master |

### 2.5 Timeline Tingkat Tinggi
Durasi proyek adalah 12 minggu, dilaksanakan dalam 6 sprint Agile berdurasi 2 minggu, dilanjutkan hypercare 2 minggu setelah go-live sebagai bagian dari dukungan transisi.

---

## 3. Scope

### 3.1 In-Scope
1. Monitoring host untuk server Linux dan Windows pada 3 data center.
2. Monitoring database PostgreSQL, MySQL, dan MongoDB.
3. Dashboard operasional real-time dan dashboard eksekutif di Grafana.
4. Alerting otomatis melalui Alertmanager ke Slack webhook internal.
5. Desain notifikasi yang siap diperluas ke Microsoft Teams melalui webhook-compatible adapter pada fase lanjutan.
6. Centralized logging menggunakan Fluentd dan Elasticsearch.
7. Penyimpanan histori metrik dan log sesuai kebijakan retensi.
8. Role-based access melalui integrasi LDAP/Active Directory.
9. Baseline aset server terhubung ke platform monitoring untuk keperluan inventaris operasional.

### 3.2 Out-of-Scope
1. End-user application performance monitoring (APM) dan distributed tracing aplikasi.
2. Vulnerability scanning, compliance scanning, dan penetration testing otomatis.
3. Monitoring backup job dan validasi restore otomatis.
4. Monitoring resource cloud publik atau hybrid cloud pada fase ini.
5. Network device monitoring lanjutan untuk switch, router, firewall, dan SD-WAN, selain reachability dasar yang mendukung server monitoring.
6. Service desk ticket automation dua arah ke ITSM tool existing.

---

## 4. Current State / Problem Statement

### 4.1 Kondisi Saat Ini

| Area | Kondisi Saat Ini | Dampak Operasional |
|---|---|---|
| Server Monitoring | Spreadsheet manual dan pengecekan host per kebutuhan | Kondisi host tidak terlihat real-time |
| Database Monitoring | Pengecekan manual melalui utilitas bawaan DB | Isu performa terlambat diketahui |
| Log Management | Tidak ada agregasi terpusat | Root cause analysis lambat |
| Alerting | Email/Nagios lama yang tidak konsisten dipantau | Alert terlewat dan tidak tereskalasi |
| Dashboard | Belum tersedia | Manajemen tidak memiliki visibilitas ringkas |
| Capacity Planning | Berdasarkan perkiraan dan data parsial | Risiko bottleneck dan overprovisioning |

### 4.2 Pernyataan Masalah
1. Organisasi tidak memiliki visibilitas terpusat atas kesehatan infrastruktur yang tersebar di tiga data center.
2. Deteksi anomali memerlukan 4-6 jam, jauh di atas kebutuhan operasi near real-time.
3. Tidak tersedia histori performa yang memadai untuk trend analysis, forecasting, dan RCA.
4. Proses manual meningkatkan human error dan ketergantungan pada personel tertentu.
5. Mekanisme alert tidak terstandar dan tidak terintegrasi ke kanal kerja tim operasi.

### 4.3 Baseline Keberhasilan yang Diharapkan
1. Seluruh aset in-scope memiliki visibilitas status dan metrik minimum di dashboard.
2. Alert severity tinggi diterima tim operasi kurang dari 1 menit setelah rule terpicu.
3. TTD untuk insiden infrastruktur prioritas tinggi turun menjadi kurang dari 15 menit.
4. Tim operasi memiliki histori metrik minimal 90 hari dan log minimal 30 hari untuk analisis.

---

## 5. Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria | Owner |
|---|---|---|---|---|
| FR-001 | Sistem harus menginventarisasi seluruh server Linux dan Windows in-scope beserta metadata minimum: hostname, IP, OS, data center, environment, owner tim, dan status monitoring. | P1 | Given daftar aset disetujui, when aset diregistrasi, then setiap aset muncul pada inventory dashboard dengan metadata wajib lengkap minimal 98%. | Infrastructure Operations Lead |
| FR-002 | Sistem harus mengumpulkan metrik host CPU, memory, disk usage, disk I/O, filesystem, uptime, dan network throughput dari seluruh server in-scope. | P1 | Given exporter aktif, when Prometheus melakukan scrape, then metrik host tersedia di Grafana maksimal 60 detik sejak pengambilan data. | DevOps Engineer |
| FR-003 | Sistem harus mengumpulkan metrik database untuk PostgreSQL, MySQL, dan MongoDB termasuk connections, query latency, cache/buffer utilization, storage growth, dan replication lag jika berlaku. | P1 | Given database exporter terpasang, when dashboard database diakses, then metrik inti tersedia per instance dan tervalidasi oleh DBA pada UAT. | DBA Lead |
| FR-004 | Sistem harus menyediakan dashboard overview eksekutif yang menampilkan status kesehatan per data center, jumlah alert aktif, kapasitas storage utama, dan tren utilisasi 7/30/90 hari. | P1 | Given user role Executive atau Operations, when membuka dashboard overview, then status ringkasan seluruh data center tampil dalam kurang dari 5 detik. | Product Owner Internal |
| FR-005 | Sistem harus menyediakan dashboard operasional per server yang menampilkan metrik near real-time, histori 24 jam, 7 hari, dan 30 hari. | P1 | Given host dipilih, when user membuka server detail dashboard, then data time-series dapat difilter berdasarkan rentang waktu tanpa error. | Operations Lead |
| FR-006 | Sistem harus menyediakan dashboard database terpisah per engine untuk memantau health dan kapasitas. | P1 | Given user DBA atau Operations, when membuka dashboard DB, then indikator health utama per instance tampil dan dapat diurutkan berdasarkan severity. | DBA Lead |
| FR-007 | Sistem harus mengumpulkan log sistem dan aplikasi infrastruktur yang relevan ke repositori terpusat untuk pencarian dan korelasi insiden. | P1 | Given Fluentd agent aktif, when log dikirim, then log dapat dicari di dashboard dalam maksimal 2 menit sejak event tercatat. | DevOps Engineer |
| FR-008 | Sistem harus menerapkan alert rule untuk CPU, memory, disk, disk I/O wait, network latency, host down, service down, dan replication lag database. | P1 | Given ambang batas terlampaui melewati durasi hold-down, when alert rule dievaluasi, then alert severity Warning atau Critical dibuat otomatis. | Operations Lead |
| FR-009 | Sistem harus mengirim alert ke Slack channel operasional melalui webhook internal dengan format standar berisi severity, objek terdampak, ringkasan masalah, timestamp, dan tautan dashboard. | P1 | Given alert aktif, when Alertmanager mengirim notifikasi, then pesan Slack diterima kanal target dalam kurang dari 60 detik dan memuat tautan panel relevan. | Service Owner |
| FR-010 | Sistem harus melakukan deduplikasi dan grouping alert agar insiden yang sama tidak membanjiri kanal notifikasi. | P1 | Given beberapa host dalam satu cluster mengalami gejala identik, when grouping rule diterapkan, then jumlah notifikasi terkonsolidasi sesuai kebijakan grouping. | DevOps Engineer |
| FR-011 | Sistem harus mendukung autentikasi melalui LDAP/Active Directory dan otorisasi berbasis role minimal: Executive View, Operations, DBA, Security Reviewer, dan Platform Admin. | P1 | Given user valid di LDAP, when login, then akses dashboard sesuai role tanpa privilege escalation. | Security Manager |
| FR-012 | Sistem harus menyediakan pencarian log berdasarkan hostname, timestamp, severity, dan kata kunci. | P2 | Given log tersedia, when user melakukan query pencarian, then hasil tampil dengan filter yang konsisten dan dapat dibuka detailnya. | Operations Lead |
| FR-013 | Sistem harus menyimpan histori metrik untuk analisis tren dan menampilkan baseline kapasitas per data center. | P1 | Given data telah terkumpul 30 hari, when user membuka dashboard trend, then grafik utilisasi dan pertumbuhan storage dapat dibandingkan antar periode. | Infrastructure Operations Lead |
| FR-014 | Sistem harus mendukung penandaan maintenance window agar alert dari aset terkait disenyapkan sementara tanpa menghapus histori. | P2 | Given maintenance window aktif, when aset masuk kondisi abnormal, then alert baru untuk aset tersebut disuppress tetapi event tetap tercatat. | Operations Lead |
| FR-015 | Sistem harus menyediakan audit trail perubahan konfigurasi dashboard, alert rule, data source, dan role mapping. | P2 | Given administrator melakukan perubahan, when perubahan disimpan, then jejak audit memuat siapa, apa, dan kapan perubahan dilakukan. | Platform Admin |
| FR-016 | Sistem harus menghasilkan laporan mingguan otomatis untuk ringkasan availability host kritikal, tren kapasitas, dan daftar insiden utama. | P3 | Given jadwal laporan mingguan, when scheduler berjalan, then laporan tersedia untuk diunduh atau dibagikan ke stakeholder internal. | Service Owner |
| FR-017 | Sistem harus memungkinkan penambahan aset baru melalui proses onboarding standar tanpa perubahan manual di banyak komponen. | P2 | Given aset baru disetujui, when template onboarding dijalankan, then host termonitor penuh dalam waktu maksimal 1 hari kerja. | DevOps Engineer |
| FR-018 | Sistem harus menampilkan status konektivitas lintas data center untuk membedakan host failure dari network partition. | P2 | Given koneksi antar data center terganggu, when alert muncul, then dashboard menandai kemungkinan gangguan jaringan area agar triase lebih cepat. | Network & Infra Operations |

---

## 6. Non-Functional Requirements

| ID | Requirement | Priority | Measurable Target | Owner |
|---|---|---|---|---|
| NFR-001 | Availability platform monitoring | P1 | Availability layanan dashboard dan alerting minimum 99.5% per bulan di luar maintenance window terjadwal | Service Owner |
| NFR-002 | Time-to-Detect (TTD) | P1 | Rata-rata deteksi awal insiden prioritas tinggi kurang dari 15 menit; target operasional alert generation kurang dari 2 menit sejak threshold dilanggar | Head of IT Operations |
| NFR-003 | Notification latency | P1 | Notifikasi Slack terkirim kurang dari 60 detik setelah alert dibuat | DevOps Engineer |
| NFR-004 | Metrics freshness | P1 | Data metrik host tersedia di dashboard dengan lag maksimum 60 detik; interval scrape default 15 detik | Platform Admin |
| NFR-005 | Log ingestion latency | P1 | Log searchable dalam maksimum 2 menit sejak diterbitkan oleh sumber | DevOps Engineer |
| NFR-006 | Dashboard performance | P2 | Dashboard overview memuat kurang dari 5 detik untuk 95% request pada jam kerja normal | Platform Admin |
| NFR-007 | Scalability | P1 | Platform mendukung minimum 300 server, 40 instance database, dan pertumbuhan log 2x dari baseline tanpa redesign mayor | Solution Architect |
| NFR-008 | Data retention | P1 | Retensi metrik minimum 90 hari online; retensi log minimum 30 hari online; arsip bulanan 12 bulan untuk agregat kapasitas | Data Owner - IT Operations |
| NFR-009 | Security access control | P1 | 100% akses dashboard menggunakan autentikasi terpusat LDAP/AD dan role-based access; akun lokal admin dibatasi untuk break-glass | Security Manager |
| NFR-010 | Auditability | P2 | Perubahan konfigurasi utama dan login administratif tercatat dan dapat ditelusuri minimal 180 hari | Security Manager |
| NFR-011 | Backup & recovery | P1 | Backup konfigurasi harian; backup metadata dan dashboard minimal 1 kali per hari; RPO maksimum 4 jam dan RTO maksimum 8 jam | Service Owner |
| NFR-012 | Operability | P1 | Tersedia runbook operasional, checklist health check, dan prosedur incident escalation sebelum go-live | Head of IT Operations |
| NFR-013 | Data quality | P1 | Cakupan monitoring minimum 95% dari aset in-scope pada saat go-live dan 98% dalam 30 hari setelah go-live | Infrastructure Operations Lead |
| NFR-014 | Maintainability | P2 | Perubahan rule alert dan dashboard mengikuti version-controlled configuration dan peer review minimal 1 reviewer | Platform Admin |
| NFR-015 | Confidentiality | P1 | Log yang mengandung PII atau secret tidak boleh disimpan; kebijakan filter dan masking wajib diterapkan pada pipeline log | Security Manager |

### 6.1 SLA dan Target Respons Operasional
1. Insiden P1: acknowledgement maksimal 10 menit, mitigasi awal maksimal 60 menit, update status setiap 30 menit.
2. Insiden P2: acknowledgement maksimal 30 menit, mitigasi awal maksimal 4 jam.
3. Insiden P3: ditangani dalam hari kerja yang sama atau dijadwalkan ke backlog sprint berikutnya jika bukan gangguan aktif.
4. Platform monitoring sendiri dikategorikan sebagai layanan internal kritikal tier-2 karena menjadi enabler operasi seluruh infrastruktur.

---

## 7. Data Architecture / Key Data Entities

### 7.1 Entitas Data Utama

| Entitas | Deskripsi | Atribut Kunci | Pemilik Data | Retensi |
|---|---|---|---|---|
| Asset | Representasi server/instance yang dimonitor | asset_id, hostname, IP, OS, data_center, environment, owner_team, criticality | IT Operations | Selama aset aktif + 12 bulan histori |
| Metric Sample | Sampel time-series hasil scrape | metric_name, timestamp, value, labels, asset_id | IT Operations | 90 hari online |
| Database Instance | Objek database yang dimonitor | db_id, engine, version, host_asset_id, environment, role, cluster_name | DBA Team | Selama instance aktif + 12 bulan histori |
| Log Event | Record log terpusat | timestamp, hostname, application, severity, message, tags | IT Operations / Security Review | 30 hari online |
| Alert Event | Event alert yang dihasilkan rule engine | alert_id, rule_id, severity, started_at, status, affected_object, routed_channel | IT Operations | 180 hari |
| Dashboard Definition | Konfigurasi visualisasi | dashboard_id, owner_role, data_source, version, last_updated | Platform Admin | Sepanjang lifecycle sistem |
| User Role Mapping | Pemetaan akses pengguna | user_id, ldap_group, role, last_sync | Security Manager | Selama akses aktif |
| Audit Event | Jejak aktivitas perubahan | event_id, actor, action, target, timestamp | Security Manager | 180 hari minimum |

### 7.2 Arsitektur Data
1. Prometheus menyimpan metrik aktif untuk query cepat dan evaluasi alert.
2. TimescaleDB digunakan sebagai repositori time-series historis untuk analisis tren dan pelaporan jangka menengah jika diperlukan oleh dashboard agregat dan query kapasitas.
3. Elasticsearch menyimpan log terpusat dengan indeks berbasis waktu untuk pencarian dan korelasi insiden.
4. Grafana berfungsi sebagai lapisan presentasi atas data source Prometheus, TimescaleDB, dan Elasticsearch.
5. Metadata aset dikelola melalui konfigurasi inventory terstandar dan sinkronisasi periodik dari daftar aset operasi.

### 7.3 Prinsip Kepemilikan Data
1. IT Operations menjadi data owner untuk inventory aset, metrik host, alert operasional, dan dashboard operasional.
2. DBA Team menjadi co-owner untuk metrik serta dashboard database.
3. Security Team memiliki hak review untuk log dan audit event terkait kepatuhan, tanpa menjadi pemilik perubahan operasional harian.
4. Data monitoring hanya dipakai untuk operasi, RCA, capacity planning, dan audit internal; tidak untuk evaluasi kinerja personal.

---

## 8. System Architecture / Integration Design

### 8.1 Komponen Solusi
1. **Prometheus** sebagai collector dan rule evaluator untuk metrik.
2. **Node Exporter / Windows Exporter** untuk metrik host.
3. **Database Exporters** untuk PostgreSQL, MySQL, dan MongoDB.
4. **Fluentd** sebagai collector dan forwarder log.
5. **Elasticsearch** sebagai repositori log.
6. **TimescaleDB** sebagai penyimpanan histori analitik time-series tambahan.
7. **Grafana** sebagai dashboard dan akses pengguna.
8. **Alertmanager** untuk routing, deduplication, silence, dan notifikasi.
9. **LDAP/AD** sebagai identity provider.
10. **Slack Webhook Gateway** sebagai kanal notifikasi utama fase ini.
11. **Dedicated Monitoring Servers** minimal 2 node: primary dan warm standby, masing-masing menjalankan stack berbasis Docker Compose sesuai kebutuhan peran layanan.

### 8.2 Alur Integrasi
1. Exporter pada server Linux/Windows dan instance database mempublikasikan metrik.
2. Prometheus melakukan scrape metrik setiap 15 detik dan mengevaluasi rules setiap 30 detik.
3. Fluentd mengirim log dari host ke Elasticsearch dengan buffer lokal sementara untuk mengurangi kehilangan data saat gangguan koneksi singkat.
4. Grafana membaca data dari Prometheus untuk real-time, dari TimescaleDB untuk histori agregat, dan dari Elasticsearch untuk log search.
5. Alertmanager menerima alert dari Prometheus, menerapkan grouping, suppression, dan routing, lalu mengirimkan notifikasi ke Slack webhook.
6. LDAP/AD melakukan autentikasi pengguna Grafana dan mapping group-to-role.

### 8.3 Desain Deployment
1. Monitoring ditempatkan pada segmen jaringan internal yang hanya dapat diakses dari subnet operasional dan jump host administratif.
2. Primary node menangani workload aktif; warm standby menerima sinkronisasi konfigurasi dan backup berkala untuk pemulihan cepat.
3. Backup konfigurasi Grafana, Alertmanager, inventory, dan file Compose disimpan ke repositori backup internal harian.
4. Storage dipisah logis antara data metrics, data log, dan backup untuk mengurangi blast radius kapasitas penuh.

### 8.4 Rencana Rollback dan Disaster Recovery
1. Rollback perubahan konfigurasi dilakukan melalui versi konfigurasi terakhir yang telah tervalidasi dan disimpan di repository konfigurasi.
2. Jika deployment sprint menyebabkan degradasi, tim mengembalikan konfigurasi dashboard/alert ke baseline stabil maksimal 2 jam sejak keputusan rollback.
3. Jika primary monitoring node gagal, warm standby diaktifkan dengan target RTO 8 jam dan RPO 4 jam.
4. Jika penyimpanan Elasticsearch atau metrics storage penuh, sistem menerapkan alert kapasitas kritikal, penghentian indeks non-kritis, dan prosedur archiving sesuai runbook.

---

## 9. UI/UX Specifications / Dashboard and User Journey

### 9.1 Prinsip UI/UX
1. Dashboard harus memprioritaskan kejelasan operasional, bukan visual dekoratif.
2. Informasi kritikal ditampilkan melalui status warna severity yang konsisten: Normal, Warning, Critical, Acknowledged, Silenced.
3. Setiap alert yang tampil harus memiliki tautan ke dashboard detail dan runbook ringkas.
4. Dashboard eksekutif menekankan tren, risiko kapasitas, dan heatmap status data center.
5. Dashboard operasional menekankan triase cepat, filter berdasarkan data center/environment, dan korelasi log.

### 9.2 Dashboard Wajib

| Dashboard | Pengguna Utama | Konten Minimum | Frekuensi Pemakaian |
|---|---|---|---|
| Executive Overview | VP, CTO, Head of IT Ops | Status 3 data center, jumlah alert aktif, top risk kapasitas, availability mingguan | Harian / mingguan |
| Infrastructure Overview | Operations Team | Health seluruh host, host down, CPU/memory/storage teratas, filter per DC | Harian intensif |
| Server Detail | Operations Team | CPU, memory, disk, network, log terkait, alert history | Saat investigasi |
| Database Health | DBA Team | Connections, latency, locks, replication lag, storage growth | Harian intensif |
| Log Search & Incident Correlation | Operations, Security Review | Pencarian log, filter severity, korelasi per host/periode | Saat insiden |
| Capacity & Trend | Operations, Management | Tren 7/30/90 hari, forecast konsumsi storage dan memory | Mingguan / bulanan |

### 9.3 User Journey Utama
1. Operator menerima notifikasi Slack severity Critical.
2. Operator membuka tautan dashboard dari notifikasi dan melihat affected asset serta panel utama.
3. Operator memvalidasi apakah insiden berasal dari host, database, atau gangguan jaringan area.
4. Operator membuka log terkait untuk korelasi waktu kejadian.
5. Operator melakukan acknowledgement dan eskalasi sesuai runbook.
6. Setelah mitigasi, operator menutup insiden dan menambahkan catatan singkat untuk post-incident review.

### 9.4 Spesifikasi Navigasi dan Filter
1. Filter global minimum: data center, environment, asset type, severity, dan time range.
2. Semua dashboard detail harus memiliki breadcrumb ke overview.
3. Pencarian log harus mendukung query berdasarkan hostname, kata kunci, severity, dan rentang waktu.
4. Tampilan mobile bukan target utama operasional, tetapi dashboard notifikasi harus tetap dapat dibaca pada perangkat mobile untuk triase awal.

---

## 10. Business Rules & Logic

1. Aset tanpa metadata owner_team dan criticality tidak boleh dinyatakan siap go-live monitoring penuh.
2. Threshold default berlaku global, tetapi host atau database kritikal dapat memiliki override yang disetujui oleh Operations Lead dan DBA Lead.
3. Alert Critical hanya dibuat jika threshold terlampaui secara konsisten selama hold-down period yang ditetapkan untuk mengurangi false positive.
4. Maintenance window harus disetujui oleh Operations Lead atau delegate dan wajib memiliki waktu mulai/selesai yang jelas.
5. Acknowledge alert tidak menutup alert; status close hanya dilakukan setelah kondisi normal kembali atau incident commander menyetujui override.
6. Jika satu kejadian berdampak ke banyak host dalam cluster yang sama, alert harus digrupkan berdasarkan cluster/service label.
7. Data log yang teridentifikasi mengandung PII atau secret harus difilter atau dimasking sebelum masuk ke Elasticsearch.
8. Hanya role Platform Admin yang dapat mengubah data source, rule global, dan konfigurasi integrasi notifikasi.
9. Dashboard eksekutif hanya menampilkan data agregat; detail teknis mendalam dibatasi ke role operasional.
10. Onboarding aset baru harus menggunakan template standar label dan exporter agar konsistensi data terjaga.
11. Jika host tidak terscrape lebih dari 2 interval berturut-turut, sistem harus menandai status Unknown dan memicu alert host_unreachable sesuai rule.
12. Perubahan threshold global harus melalui change request ringan dan validasi setidaknya satu siklus observasi sebelum dijadikan baseline baru.

### 10.1 Threshold Operasional Awal

| Metric | Warning | Critical | Hold-Down Period | Catatan |
|---|---|---|---|---|
| CPU Usage | > 70% | > 90% | 5 menit | Dikecualikan untuk batch window terjadwal |
| Memory Usage | > 75% | > 90% | 5 menit | Pertimbangkan cache behavior OS |
| Disk Usage | > 80% | > 90% | 10 menit | Berlaku per filesystem kritikal |
| Disk I/O Wait | > 20 ms | > 50 ms | 5 menit | Berlaku pada volume produksi |
| Network Latency Antar-DC | > 5 ms baseline deviasi signifikan | > 20 ms | 3 menit | Dibaca sebagai indikator gangguan area |
| Host Reachability | 1 interval gagal | 2 interval gagal | 30-60 detik | Alert langsung untuk host kritikal |
| PostgreSQL Replication Lag | > 30 detik | > 120 detik | 5 menit | Hanya untuk instance replika |
| MySQL Connection Usage | > 75% | > 90% | 5 menit | Dibandingkan max_connections |
| MongoDB Replication Health | Node warning | Primary unavailable | 2 menit | Disesuaikan dengan topology replica set |

---

## 11. Security Requirements

1. Seluruh akses pengguna harus menggunakan LDAP/AD; akun lokal hanya untuk break-glass dan disimpan sesuai prosedur keamanan internal.
2. Akses ke Grafana, Prometheus UI, dan endpoint administratif dibatasi ke jaringan internal dan/atau jump host yang disetujui.
3. Integrasi webhook Slack harus menggunakan secret yang disimpan pada secret store internal atau file konfigurasi terproteksi; secret tidak boleh ditampilkan di dashboard atau log.
4. Data in transit antar komponen monitoring harus menggunakan kanal terenkripsi apabila melintasi segmen jaringan yang tidak sepenuhnya trusted; minimal TLS diwajibkan untuk akses pengguna ke dashboard.
5. Role-based access minimal mencakup pembatasan edit dashboard, edit rule, dan akses log sensitif.
6. Logging audit harus mencatat login administratif, perubahan rule, perubahan data source, perubahan role mapping, dan aktivasi maintenance window.
7. Pipeline log harus menerapkan filtering/masking untuk PII, credential, token, dan informasi rahasia lain sebelum data diindeks.
8. Backup konfigurasi dan metadata monitoring harus disimpan pada lokasi internal yang terproteksi dan hanya dapat diakses personel berwenang.
9. Review akses pengguna dilakukan minimal triwulanan oleh Security Manager dan Service Owner.
10. Hardening OS monitoring server mengikuti baseline internal: patching berkala, non-root container runtime bila memungkinkan, pembatasan port, dan sinkronisasi waktu NTP.

---

## 12. Testing Strategy & Acceptance Criteria

### 12.1 Strategi Pengujian
1. **Unit/Component Validation:** verifikasi exporter, endpoint scrape, koneksi data source, dan format payload alert.
2. **Integration Testing:** verifikasi alur end-to-end dari metric/log ingestion sampai dashboard dan notifikasi Slack.
3. **Performance Testing:** uji beban pada dashboard overview dan ingestion baseline untuk 200+ server.
4. **Security Testing:** verifikasi autentikasi LDAP/AD, otorisasi role, pembatasan jaringan, dan masking log sensitif.
5. **User Acceptance Testing (UAT):** dilakukan oleh IT Operations, DBA Team, dan Security Review berdasarkan skenario operasional nyata.
6. **Failover/Recovery Drill:** simulasi kegagalan primary monitoring node dan restore konfigurasi utama.

### 12.2 Kriteria Penerimaan Solusi
1. Cakupan monitoring minimal 95% aset in-scope pada saat go-live.
2. Dashboard overview, server detail, dan database health dapat digunakan tanpa defect severity tinggi yang belum ditutup.
3. Alert host down, disk critical, dan database replication lag tervalidasi terkirim ke Slack sesuai target latency.
4. Log dapat dicari dan dikorelasikan untuk minimal 3 skenario insiden referensi.
5. UAT ditandatangani Head of IT Operations, DBA Lead, dan Security Manager.
6. Runbook operasional, backup procedure, dan access matrix tersedia dan disosialisasikan.

### 12.3 Definition of Done (DoD)
1. Requirement sprint memenuhi acceptance criteria yang disepakati.
2. Konfigurasi berada dalam version control dan telah direview minimal satu reviewer.
3. Monitoring rule dan dashboard memiliki bukti uji yang dapat ditelusuri.
4. Dokumentasi konfigurasi, runbook, dan known limitations diperbarui.
5. Tidak ada defect severity Critical atau High yang terbuka untuk scope go-live.

### 12.4 UAT Exit Criteria
1. Seluruh skenario UAT prioritas tinggi lulus 100%.
2. Maksimal 5 defect medium dapat dibawa ke backlog pasca go-live, tanpa memengaruhi kontrol dasar monitoring.
3. Tidak ada defect high/critical terbuka pada akses, alerting, atau integritas data monitoring.
4. KPI TTD simulasi menunjukkan hasil konsisten kurang dari 15 menit untuk skenario P1 yang diuji.

---

## 13. Implementation Timeline Aligned to Agile

### 13.1 Rencana Sprint 12 Minggu

| Sprint | Minggu | Tujuan Sprint | Deliverable Utama |
|---|---|---|---|
| Sprint 1 | 1-2 | Foundation & environment setup | Monitoring node primary siap, baseline network/security, repository konfigurasi, daftar aset tervalidasi |
| Sprint 2 | 3-4 | Host metrics onboarding | Exporter Linux/Windows terpasang untuk gelombang 1, dashboard host dasar, scrape dan label standard aktif |
| Sprint 3 | 5-6 | Database monitoring & asset coverage expansion | Exporter DB aktif, dashboard database awal, onboarding aset gelombang 2 |
| Sprint 4 | 7-8 | Centralized logging & executive dashboard | Fluentd + Elasticsearch aktif, log search dasar, executive overview tersedia |
| Sprint 5 | 9-10 | Alerting, tuning, dan role-based access | Alertmanager ke Slack, threshold tuning, LDAP/AD integration, maintenance window |
| Sprint 6 | 11-12 | UAT, failover drill, go-live readiness | UAT lulus, runbook final, failover drill, sign-off go-live |

### 13.2 Ritme Scrum
1. Sprint duration: 2 minggu.
2. Sprint planning: hari pertama sprint.
3. Daily stand-up: 15 menit pada hari kerja.
4. Sprint review: hari terakhir sprint dengan demo dashboard/fitur yang selesai.
5. Sprint retrospective: setelah review, dengan action item yang dilacak pada backlog improvement.

### 13.3 Resource Plan

| Peran | Alokasi | Tanggung Jawab Utama |
|---|---|---|
| Product Owner Internal | 30% | Prioritas backlog, keputusan scope, UAT sign-off bisnis-operasional |
| Scrum Master / PMO | 50% | Delivery cadence, dependency, impediment removal |
| Solution Architect / Tech Lead | 40% | Review arsitektur, standard label, review teknis |
| Infrastructure Engineers | 2 FTE | Setup host, exporter, network access, inventory |
| DevOps Engineer | 1 FTE | Prometheus, Alertmanager, Grafana, automation config |
| DBA Engineer | 0.5 FTE | Exporter DB, dashboard DB, validasi metrik |
| Security Reviewer | 0.2 FTE | LDAP/AD, access review, log filtering review |

---

## 14. Dependencies & Assumptions

### 14.1 Dependencies
1. Akses jaringan dari monitoring node ke seluruh server dan endpoint exporter harus tersedia lintas tiga data center.
2. Tim operasi menyediakan daftar aset yang akurat dan mutakhir sebelum Sprint 2.
3. LDAP/AD group yang dibutuhkan untuk role mapping tersedia sebelum Sprint 5.
4. Kanal Slack operasional dan webhook internal telah disetujui oleh tim keamanan.
5. Kapasitas server monitoring, storage, dan backup telah disetujui sebelum implementasi produksi.
6. Tim aplikasi/sistem sumber bekerja sama untuk memastikan log dapat difilter dari data sensitif jika diperlukan.

### 14.2 Assumptions
1. Seluruh workload target fase ini berada pada lingkungan on-premise dan dapat diakses dari jaringan internal PT Nusantara Digital.
2. Tidak ada kebutuhan compliance khusus yang mewajibkan retensi log lebih dari 30 hari untuk fase pertama.
3. Tim operasi menerima Slack sebagai kanal notifikasi utama; Microsoft Teams belum diwajibkan untuk go-live fase ini.
4. Dedicated monitoring node dapat diprovisikan sebagai VM atau server fisik internal tanpa proses procurement tambahan lebih dari 2 minggu.
5. Exporter dapat dipasang pada mayoritas host tanpa perubahan aplikasi yang signifikan.
6. Backup monitoring configuration menggunakan infrastruktur backup internal yang sudah ada.

---

## 15. Risk Register

| ID | Risk | Probability | Impact | Mitigation | Owner |
|---|---|---|---|---|---|
| R-001 | Monitoring server primary down | Medium | High | Sediakan warm standby, backup harian, failover drill, alert kesehatan platform sendiri | Service Owner |
| R-002 | Network partition antar data center menyebabkan false host-down massal | Medium | High | Terapkan alert correlation area, baseline reachability antar-DC, dashboard status konektivitas | Network & Infra Operations |
| R-003 | Storage metrics/log penuh | Medium | High | Capacity alert 70/80/90%, retensi tegas, archiving, review pertumbuhan mingguan | Platform Admin |
| R-004 | Cakupan aset tidak lengkap karena inventory awal tidak akurat | High | Medium | Validasi inventory Sprint 1, owner per data center, progress tracking coverage | Infrastructure Operations Lead |
| R-005 | False positive alert tinggi pada minggu awal | Medium | Medium | Tuning threshold pada Sprint 5 dan UAT, gunakan hold-down dan grouping | Operations Lead |
| R-006 | Log sensitif masuk ke Elasticsearch | Low | High | Filter/masking di Fluentd, review sample log, kontrol akses ketat | Security Manager |
| R-007 | Keterlambatan integrasi LDAP/AD | Medium | Medium | Gunakan jadwal dependency awal dan fallback akses terbatas untuk SIT non-produksi; go-live tetap menunggu LDAP | Security Manager |
| R-008 | Kinerja dashboard menurun saat volume naik | Medium | Medium | Optimasi query, dashboard simplification, partitioning storage, sizing buffer 2x baseline | Solution Architect |
| R-009 | Resistensi adopsi dari tim operasi yang terbiasa manual | Medium | Medium | Training, champion per tim, dashboard disesuaikan use case harian | Head of IT Operations |
| R-010 | Backup/restore konfigurasi tidak tervalidasi | Low | High | Recovery drill sebelum go-live dan triwulanan sesudahnya | Service Owner |

---

## 16. Change Management Plan

1. Semua perubahan scope, threshold global, data source baru, atau retensi harus dicatat sebagai change request internal.
2. Change request dikategorikan sebagai:
   - Minor: perubahan dashboard non-kritis, label, atau visual tanpa dampak kontrol inti.
   - Standard: penambahan aset massal, rule baru, perubahan threshold global.
   - Major: perubahan arsitektur, penambahan data center baru, perubahan teknologi inti.
3. Minor change dapat disetujui oleh Service Owner dan Platform Admin.
4. Standard change memerlukan review Product Owner Internal, Operations Lead, dan Tech Lead.
5. Major change memerlukan persetujuan VP Infrastructure & Security dan CTO.
6. Semua perubahan produksi harus diuji pada lingkungan staging/non-prod atau melalui validasi terkontrol sebelum diterapkan.
7. Setiap sprint review harus memuat daftar change item yang selesai, deferred, dan candidate untuk sprint berikutnya.
8. Backlog harus diprioritaskan menggunakan kombinasi nilai operasional, risiko, dan effort.

---

## 17. Handover & Support Plan

### 17.1 Handover Deliverables
1. Dokumen FSD final dan konfigurasi arsitektur operasional.
2. Runbook untuk health check harian, response alert, backup, restore, dan failover.
3. Access matrix dan prosedur onboarding/offboarding pengguna.
4. Dashboard catalog dan daftar alert rule aktif.
5. Known limitations serta backlog improvement pasca go-live.

### 17.2 Support Model
1. **Hypercare:** 2 minggu setelah go-live dengan dukungan harian dari tim implementasi dan war room virtual pada jam kerja utama.
2. **Business-as-Usual (BAU):** setelah hypercare, kepemilikan operasional berpindah ke Head of IT Operations sebagai service owner.
3. **L1 Support:** NOC/Operations untuk triase alert dan monitoring harian.
4. **L2 Support:** Infrastructure/DevOps/DBA sesuai domain gangguan.
5. **L3 Support:** Solution Architect atau vendor/internal specialist untuk isu platform kompleks.

### 17.3 Knowledge Transfer
1. Minimal 3 sesi knowledge transfer: operasi harian, administrasi platform, dan troubleshooting/failover.
2. Rekaman dan materi KT disimpan pada repositori dokumentasi internal.
3. Tim penerima wajib menyelesaikan simulasi incident handling sebelum transisi BAU.

---

## 18. Sign-off / Approval Matrix

| Role | Responsibility in Approval | Decision Type |
|---|---|---|
| VP Infrastructure & Security | Persetujuan sponsor dan risiko operasional | Final Business/Operational Approval |
| CTO | Persetujuan keselarasan arsitektur dan arah teknologi | Final Technical Approval |
| Head of IT Operations | Persetujuan kesiapan operasional dan service ownership | Operational Readiness Approval |
| Solution Architect / Tech Lead | Validasi desain integrasi, kapasitas, maintainability | Technical Review Approval |
| DBA Lead | Persetujuan cakupan dan kualitas monitoring database | Domain Approval |
| Security Manager | Persetujuan kontrol akses, logging, dan kerahasiaan data | Security Approval |
| Product Owner Internal | Persetujuan backlog prioritas dan hasil UAT | Product Acceptance |
| Scrum Master / PMO | Konfirmasi governance delivery dan evidensi proses | Delivery Governance Acknowledgement |

---

## 19. Methodology Notes (Agile / Scrum Alignment)

| Area | Penerapan pada SAMAHI | Status |
|---|---|---|
| Sprint Planning | 6 sprint, masing-masing 2 minggu, dengan sprint goal dan deliverable jelas | COMPLETE |
| Backlog | Requirement diprioritaskan P1/P2/P3 dan akan dipecah menjadi user story implementasi | COMPLETE |
| User Stories | Functional requirement siap diturunkan menjadi user story INVEST per sprint | COMPLETE |
| Definition of Done | DoD eksplisit pada bagian pengujian dan kualitas | COMPLETE |
| Technical Debt | Setiap sprint menyediakan slot tuning/rule refinement minimal 10-15% kapasitas | COMPLETE |
| Retrospectives | Retro dilakukan setiap akhir sprint dan action item masuk backlog improvement | COMPLETE |
| Velocity | Tracking velocity tim digunakan untuk forecasting Sprint 4-6 setelah dua sprint awal | COMPLETE |
| Cross-functional Team | Tim mencakup infra, DevOps, DBA, security review, dan governance | COMPLETE |

### 19.1 Catatan Agile Operasional
1. FSD ini berfungsi sebagai baseline requirement dan arah delivery, bukan kontrak scope yang kaku.
2. Penyesuaian minor dapat dilakukan melalui backlog grooming selama tidak mengubah objective utama proyek.
3. Demo per sprint wajib menampilkan working configuration, bukan hanya desain konseptual.
4. Metrik keberhasilan delivery meliputi coverage onboarding, latency alert, defect trend, dan kesiapan operasional.

---

## 20. Assumptions & Open Questions

### 20.1 Assumptions
1. Jumlah awal aset adalah lebih dari 200 server dengan komposisi sekitar 120 Linux dan 80 Windows, dan dapat bertambah sampai 300 dalam 12-18 bulan.
2. Data center Jakarta menjadi lokasi primary monitoring node, dengan warm standby di Surabaya untuk ketahanan regional.
3. Slack merupakan platform kolaborasi yang paling siap untuk fase pertama; Teams disiapkan secara desain namun tidak menjadi syarat go-live.
4. Monitoring database difokuskan pada telemetry kesehatan dan performa inti, bukan query-level tracing mendalam.
5. Retensi log 30 hari dinilai cukup untuk kebutuhan investigasi operasional saat ini, sedangkan trend kapasitas memanfaatkan agregasi metrik 90 hari.

### 20.2 Open Questions
1. Apakah PT Nusantara Digital menginginkan fase lanjutan yang mewajibkan notifikasi paralel ke Microsoft Teams selain Slack.
2. Apakah ada klasifikasi sistem kritikal yang memerlukan threshold, retensi, atau SLA respons lebih ketat daripada baseline pada dokumen ini.
3. Apakah inventory aset authoritative berasal dari CMDB formal, spreadsheet existing, atau kombinasi keduanya.
4. Apakah ada kebijakan compliance internal yang mensyaratkan enkripsi tambahan antar data center untuk trafik log dan metrics.
5. Apakah hypercare 2 minggu dianggap cukup, atau diperlukan 4 minggu untuk musim perubahan operasional tertentu.
6. Apakah laporan mingguan otomatis perlu didistribusikan melalui email internal selain tersedia di dashboard.

---

## 21. Completeness Check

| Section | Status | Catatan |
|---|---|---|
| Title/Header with document type, version, date, methodology, audience | COMPLETE | Memuat klasifikasi dokumen dan audiens utama |
| Executive Summary | COMPLETE | Menjelaskan masalah, solusi, target bisnis |
| Project Context | COMPLETE | Nama proyek, latar belakang, tujuan, stakeholder, timeline |
| Scope | COMPLETE | In-scope dan out-of-scope eksplisit |
| Current State / Problem Statement | COMPLETE | Baseline dan dampak operasional terdokumentasi |
| Functional Requirements | COMPLETE | FR-001 s.d. FR-018 dengan acceptance criteria dan owner |
| Non-Functional Requirements | COMPLETE | NFR terukur termasuk SLA, RTO, RPO, retensi |
| Data Architecture / Key Data Entities | COMPLETE | Entitas, owner data, retensi, prinsip kepemilikan |
| System Architecture / Integration Design | COMPLETE | Komponen, alur integrasi, deployment, rollback, DR |
| UI/UX Specifications / Dashboard and User Journey | COMPLETE | Dashboard wajib, user journey, prinsip desain |
| Business Rules & Logic | COMPLETE | Aturan operasional dan threshold awal |
| Security Requirements | COMPLETE | LDAP, RBAC, masking, audit, hardening |
| Testing Strategy & Acceptance Criteria | COMPLETE | SIT, UAT, failover, DoD, exit criteria |
| Implementation Timeline aligned to Agile | COMPLETE | 6 sprint, resource plan, ritme Scrum |
| Dependencies & Assumptions | COMPLETE | Dependency dan asumsi eksplisit |
| Risk Register | COMPLETE | Risiko, dampak, mitigasi, owner |
| Change Management Plan | COMPLETE | Proses CR berbasis klasifikasi perubahan |
| Handover & Support Plan | COMPLETE | Support model, KT, ownership BAU |
| Sign-off / Approval Matrix | COMPLETE | Role-based approval tanpa placeholder |
| Methodology Notes aligned to Agile/Scrum | COMPLETE | Selaras dengan matriks Agile/Scrum |
| Assumptions & Open Questions | COMPLETE | Dibedakan dengan jelas |
| Security & Confidentiality Notes | COMPLETE | Tercantum pada bagian berikut |

---

## 22. Security & Confidentiality Notes

1. Dokumen ini diklasifikasikan sebagai **Confidential - Internal Use Only** dan hanya boleh diedarkan kepada stakeholder yang terdaftar dalam approval matrix dan tim delivery terkait.
2. Tidak ada secret, credential, token, atau URL webhook aktual yang dituliskan pada dokumen ini. Jika artefak implementasi ditemukan mengandung secret, secret tersebut wajib di-redact, dirotasi, dan diaudit sebelum distribusi lanjut.
3. Seluruh contoh arsitektur, role, dan alur akses pada dokumen ini dimaksudkan untuk desain internal PT Nusantara Digital dan tidak untuk publikasi eksternal.
4. Log dan metrik yang diproses SAMAHI harus diperlakukan sesuai prinsip least privilege dan minimization; hanya data yang diperlukan untuk operasi dan investigasi yang boleh disimpan.
5. Review lanjutan menggunakan `/analyse-fsd` harus tetap mempertahankan klasifikasi kerahasiaan yang sama.

---

**Status Dokumen:** COMPLETE dan siap direview menggunakan `/analyse-fsd`.

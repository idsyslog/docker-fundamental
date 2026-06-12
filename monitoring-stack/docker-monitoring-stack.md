# Monitoring Stack dengan Docker Compose
## Docker + cAdvisor + Node Exporter + Prometheus + Grafana

---

# SLIDE 1 — Cover

**Judul:** Monitoring Infrastruktur Modern dengan Docker Compose  
**Subtitle:** cAdvisor · Node Exporter · Prometheus · Grafana  
**Mata Kuliah:** Sistem Terdistribusi / DevOps / Cloud Computing  
**Dosen:** *(isi nama)*  
**Semester:** *(isi semester)*

---

# SLIDE 2 — Agenda

1. Mengapa Monitoring Penting?
2. Arsitektur Stack Monitoring
3. Pengenalan Komponen
4. Alur Data (Data Flow)
5. Implementasi dengan Docker Compose
6. Konfigurasi Prometheus
7. Visualisasi dengan Grafana
8. Demo & Hands-on
9. Quiz & Kesimpulan

---

# SLIDE 3 — Mengapa Monitoring Penting?

**Masalah tanpa monitoring:**
- Server down → tidak tahu sampai user komplain
- Memory leak → performa turun bertahap tanpa terdeteksi
- Disk penuh → aplikasi crash mendadak

**Dengan monitoring kita bisa:**
- Deteksi anomali **sebelum** terjadi masalah (proactive)
- Analisis tren penggunaan resource
- Audit performa container secara real-time
- Buat alert otomatis (PagerDuty, Slack, Email)

> *"You can't improve what you don't measure."* — Peter Drucker

---

# SLIDE 4 — Arsitektur Stack Monitoring

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Host                          │
│                                                         │
│  ┌──────────────┐    ┌───────────────────────────────┐  │
│  │  Your App    │    │     Metrics Exporters         │  │
│  │  Containers  │───▶│  cAdvisor   │  Node Exporter  │  │
│  └──────────────┘    └──────┬──────┴────────┬────────┘  │
│                             │               │            │
│                             ▼               ▼            │
│                      ┌─────────────────────────┐         │
│                      │       Prometheus        │         │
│                      │   (Time-Series DB)      │         │
│                      └──────────────┬──────────┘         │
│                                     │                    │
│                                     ▼                    │
│                      ┌─────────────────────────┐         │
│                      │        Grafana          │         │
│                      │   (Visualization UI)    │         │
│                      └─────────────────────────┘         │
└─────────────────────────────────────────────────────────┘
```

**Pull Model:** Prometheus aktif **menarik (scrape)** metrics dari exporter setiap interval tertentu (default: 15 detik).

---

# SLIDE 5 — Komponen #1: Docker

**Docker** adalah platform containerisasi yang memungkinkan aplikasi berjalan dalam lingkungan terisolasi.

| Konsep | Penjelasan |
|--------|-----------|
| **Image** | Blueprint/template container |
| **Container** | Instance yang sedang berjalan |
| **Volume** | Penyimpanan persisten |
| **Network** | Jaringan antar container |

**Docker Compose** = tool untuk mendefinisikan dan menjalankan **multi-container** Docker apps dalam satu file `docker-compose.yml`.

```bash
# Perintah dasar
docker compose up -d       # jalankan semua service
docker compose down        # hentikan dan hapus container
docker compose ps          # lihat status container
docker compose logs -f     # lihat log
```

---

# SLIDE 6 — Komponen #2: cAdvisor

**cAdvisor** (Container Advisor) — dibuat oleh Google

**Fungsi:**
- Monitor resource usage **per container**
- Expose metrics dalam format Prometheus
- Berjalan sebagai container, butuh akses ke Docker socket

**Metrics yang dikumpulkan:**
```
container_cpu_usage_seconds_total       # penggunaan CPU
container_memory_usage_bytes            # penggunaan RAM
container_fs_reads_bytes_total          # I/O disk
container_network_transmit_bytes_total  # traffic jaringan
```

**Port default:** `8080`

**Penting:** cAdvisor perlu mount `/var/run/docker.sock` dan `/sys` dari host agar bisa membaca stats container.

---

# SLIDE 7 — Komponen #3: Node Exporter

**Node Exporter** — mengumpulkan metrics **level host/OS**

**Bedanya dengan cAdvisor:**

| | cAdvisor | Node Exporter |
|--|---------|---------------|
| **Scope** | Per container | Seluruh host/server |
| **Data** | CPU, RAM, net per container | CPU, RAM, disk, load avg host |
| **Pembuat** | Google | Prometheus community |

**Metrics penting:**
```
node_cpu_seconds_total          # total CPU time
node_memory_MemAvailable_bytes  # RAM tersedia
node_disk_io_time_seconds_total # waktu I/O disk
node_filesystem_avail_bytes     # ruang disk tersedia
node_network_receive_bytes_total # traffic masuk
```

**Port default:** `9100`

---

# SLIDE 8 — Komponen #4: Prometheus

**Prometheus** — Time-Series Database + Monitoring System

**Cara kerja:**
1. **Scrape** → Prometheus request ke `/metrics` endpoint setiap interval
2. **Store** → Simpan data time-series di local storage
3. **Query** → Gunakan bahasa **PromQL** untuk query data
4. **Alert** → Kirim alert ke Alertmanager jika kondisi terpenuhi

**Fitur utama:**
- Pull-based model (tidak perlu agent di setiap server)
- PromQL: bahasa query yang powerful
- Storage efisien dengan kompresi time-series
- Built-in alerting rules

**Port default:** `9090`

---

# SLIDE 9 — PromQL: Bahasa Query Prometheus

**PromQL** (Prometheus Query Language) — functional query language

**Tipe data:**
- **Instant vector** — nilai metric pada satu titik waktu
- **Range vector** — nilai metric dalam rentang waktu
- **Scalar** — angka tunggal

**Contoh query:**

```promql
# CPU usage per container (%)
rate(container_cpu_usage_seconds_total[5m]) * 100

# RAM usage dalam MB
container_memory_usage_bytes / 1024 / 1024

# Disk tersedia host
node_filesystem_avail_bytes{mountpoint="/"}

# Load average 1 menit
node_load1

# Container yang sedang running
count(container_last_seen) by (name)
```

---

# SLIDE 10 — Komponen #5: Grafana

**Grafana** — Platform visualisasi dan observability

**Fitur:**
- Dashboard interaktif dengan berbagai tipe panel (grafik, gauge, tabel, heatmap)
- Support banyak data source: Prometheus, MySQL, InfluxDB, Loki, dll
- Alerting terintegrasi
- Template dashboard siap pakai dari **Grafana Dashboard Hub**

**Konsep penting:**
| Istilah | Penjelasan |
|---------|-----------|
| **Data Source** | Koneksi ke Prometheus/DB |
| **Dashboard** | Kumpulan panel/visualisasi |
| **Panel** | Satu unit visualisasi (grafik, angka, dll) |
| **Variable** | Filter dinamis dalam dashboard |

**Port default:** `3000`  
**Login default:** `admin / admin`

---

# SLIDE 11 — Struktur Project

```
monitoring-stack/
├── docker-compose.yml          # definisi semua service
├── prometheus/
│   └── prometheus.yml          # konfigurasi scrape targets
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── prometheus.yml  # auto-config data source
        └── dashboards/
            └── dashboard.yml   # auto-load dashboard
```

**Prinsip:** Semua konfigurasi sebagai **code** (Infrastructure as Code / IaC)  
→ bisa di-version control, reproducible, mudah di-share

---

# SLIDE 12 — docker-compose.yml (Bagian 1)

```yaml
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data: {}
  grafana_data: {}

services:

  # ── cAdvisor: container metrics ──────────────────────
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring
```

**Catatan:** `:ro` = read-only mount (best practice keamanan)

---

# SLIDE 13 — docker-compose.yml (Bagian 2)

```yaml
  # ── Node Exporter: host metrics ──────────────────────
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring
```

**Penjelasan:** Node Exporter mount `/proc` dan `/sys` dari host untuk membaca statistik OS.

---

# SLIDE 14 — docker-compose.yml (Bagian 3)

```yaml
  # ── Prometheus: metrics collection ───────────────────
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    networks:
      - monitoring
```

**Flag penting:**
- `--storage.tsdb.retention.time=15d` → simpan data 15 hari
- `--web.enable-lifecycle` → reload config tanpa restart via `POST /-/reload`

---

# SLIDE 15 — docker-compose.yml (Bagian 4)

```yaml
  # ── Grafana: visualization ────────────────────────────
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - monitoring
    depends_on:
      - prometheus
```

**`depends_on`** → Grafana menunggu Prometheus siap sebelum start.

---

# SLIDE 16 — Konfigurasi Prometheus

**File:** `prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s       # seberapa sering scrape
  evaluation_interval: 15s   # seberapa sering evaluasi rules

scrape_configs:

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']  # nama service Docker

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

**Penting:** Gunakan **nama service** Docker sebagai hostname (bukan `localhost`), karena semua container dalam satu network yang sama.

---

# SLIDE 17 — Grafana: Auto-provisioning Data Source

**File:** `grafana/provisioning/datasources/prometheus.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

**Keuntungan provisioning:**
- Data source langsung tersedia saat Grafana pertama kali start
- Tidak perlu setup manual via UI
- Reproducible di environment lain (staging, production)

---

# SLIDE 18 — Alur Data Lengkap

```
[Host OS]                    [Container]
    │                            │
    ▼                            ▼
[Node Exporter]            [cAdvisor]
 port 9100                  port 8080
    │                            │
    └──────────┬─────────────────┘
               │  HTTP GET /metrics
               ▼  (setiap 15 detik)
          [Prometheus]
           port 9090
           time-series DB
               │
               │  PromQL query
               ▼
           [Grafana]
            port 3000
           dashboard UI
               │
               ▼
          [Browser User]
```

**Ingat:** Prometheus yang aktif "menarik" data — bukan exporter yang "mendorong".

---

# SLIDE 19 — Menjalankan Stack

```bash
# 1. Clone/buat struktur project
mkdir monitoring-stack && cd monitoring-stack

# 2. Buat file konfigurasi (sesuai slide sebelumnya)

# 3. Jalankan semua service
docker compose up -d

# 4. Verifikasi container berjalan
docker compose ps

# 5. Cek log jika ada masalah
docker compose logs prometheus
docker compose logs grafana

# 6. Akses UI
# Prometheus : http://localhost:9090
# Grafana    : http://localhost:3000
# cAdvisor   : http://localhost:8080
# Node Exporter metrics: http://localhost:9100/metrics
```

---

# SLIDE 20 — Verifikasi di Prometheus UI

**Langkah verifikasi:**

1. Buka `http://localhost:9090/targets`
2. Pastikan semua target berstatus **UP** (hijau)
3. Coba query di tab **Graph**:

```promql
# Cek semua instance aktif
up

# CPU host
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)

# Memory host (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Top container by RAM
topk(5, container_memory_usage_bytes{name!=""})
```

---

# SLIDE 21 — Import Dashboard Grafana

**Cara termudah:** Gunakan dashboard dari Grafana.com

**Dashboard populer:**
| ID | Nama | Untuk |
|----|------|-------|
| **1860** | Node Exporter Full | Metrics host lengkap |
| **193** | Docker Monitoring | Metrics container |
| **11600** | Docker & System Monitoring | Kombinasi host + container |

**Langkah import:**
1. Buka Grafana → **Dashboards** → **Import**
2. Masukkan ID (misal: `1860`) → **Load**
3. Pilih data source **Prometheus** → **Import**

Dashboard langsung menampilkan grafik CPU, RAM, disk, network secara real-time.

---

# SLIDE 22 — Contoh Panel Grafana

**CPU Usage per Container:**
```promql
sum(rate(container_cpu_usage_seconds_total{name!=""}[5m])) by (name) * 100
```

**Memory Usage per Container (MB):**
```promql
container_memory_usage_bytes{name!=""} / 1024 / 1024
```

**Disk I/O Host:**
```promql
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])
```

**Network Traffic:**
```promql
rate(node_network_receive_bytes_total{device="eth0"}[5m])
rate(node_network_transmit_bytes_total{device="eth0"}[5m])
```

---

# SLIDE 23 — Troubleshooting Umum

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| Target Prometheus `DOWN` | Container belum ready / nama service salah | Cek `docker compose ps`, verifikasi nama service di `prometheus.yml` |
| cAdvisor tidak bisa baca container | Permission `/var/run/docker.sock` | Tambahkan user ke group docker atau jalankan sebagai root |
| Grafana tidak bisa query Prometheus | URL data source salah | Gunakan `http://prometheus:9090` bukan `localhost` |
| Data Grafana hilang saat restart | Volume tidak di-mount | Pastikan `grafana_data` volume terdefinisi dan di-mount |
| Node Exporter metric kosong | Path `/proc` tidak benar | Verifikasi flag `--path.procfs` |

---

# SLIDE 24 — Best Practices

**Keamanan:**
- Ganti password default Grafana
- Jangan expose port monitoring ke public internet
- Gunakan reverse proxy (Nginx) + HTTPS untuk production
- Gunakan read-only mount (`:ro`) pada volume sensitif

**Performa:**
- Sesuaikan `scrape_interval` (lebih jarang = lebih hemat storage)
- Set `retention.time` sesuai kebutuhan (default 15 hari)
- Gunakan recording rules untuk query berat yang sering dipakai

**Skalabilitas:**
- Untuk multi-host: gunakan **Prometheus Federation** atau **Thanos**
- Untuk log: tambahkan **Loki** + **Promtail** ke stack
- Untuk tracing: tambahkan **Tempo** atau **Jaeger**

---

# SLIDE 25 — Ekosistem Observability

```
              ┌─────────────────────────────────┐
              │     Three Pillars of Observability│
              └─────────────────────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   METRICS    │  │    LOGS      │  │   TRACES     │
│              │  │              │  │              │
│  Prometheus  │  │    Loki      │  │    Tempo     │
│  + Grafana   │  │  + Promtail  │  │   + Jaeger   │
│              │  │              │  │              │
│ "Berapa CPU  │  │ "Error apa   │  │ "Request ini │
│  sekarang?"  │  │  yang terjadi│  │  lewat mana  │
│              │  │  jam 2 tadi?"│  │  saja?"      │
└──────────────┘  └──────────────┘  └──────────────┘
```

Stack yang kita bangun hari ini = **Metrics pillar** (fondasi observability)

---

# SLIDE 26 — Tugas / Lab

**Lab: Bangun monitoring stack sendiri**

**Minimum requirements:**
1. Jalankan stack lengkap (cAdvisor + Node Exporter + Prometheus + Grafana)
2. Screenshot halaman **Targets** Prometheus — semua harus `UP`
3. Import dashboard Node Exporter Full (ID: 1860)
4. Buat 1 dashboard custom dengan minimal 3 panel:
   - Panel CPU usage
   - Panel Memory usage
   - Panel 1 metric pilihan sendiri

**Bonus (nilai tambah):**
- Tambahkan alerting rule di Prometheus
- Tambahkan container aplikasi sendiri (web server, database) dan monitor
- Deploy ke VM (bukan laptop sendiri)

**Deadline:** *(isi deadline)*

---

# SLIDE 27 — Quiz Kilat

**Pertanyaan:**

1. Apa perbedaan utama antara **cAdvisor** dan **Node Exporter**?

2. Mengapa Prometheus menggunakan model **pull** bukan **push**?

3. Dalam `docker-compose.yml`, mengapa kita menggunakan nama **service** (misal `prometheus:9090`) bukan `localhost:9090` saat menghubungkan Grafana ke Prometheus?

4. Apa fungsi `volumes` bernama (named volumes) seperti `grafana_data`?

5. Sebutkan 3 **Three Pillars of Observability** dan tool yang digunakan untuk masing-masing!

---

# SLIDE 28 — Referensi & Resource

**Dokumentasi Resmi:**
- Prometheus: https://prometheus.io/docs/
- Grafana: https://grafana.com/docs/
- cAdvisor: https://github.com/google/cadvisor
- Node Exporter: https://github.com/prometheus/node_exporter

**Dashboard Grafana:**
- https://grafana.com/grafana/dashboards/

**PromQL Cheatsheet:**
- https://promlabs.com/promql-cheat-sheet/

**Buku & Kursus:**
- *Prometheus: Up & Running* — O'Reilly
- *Kubernetes Monitoring with Prometheus* — Judge & Bhambri

---

# SLIDE 29 — Kesimpulan

**Apa yang kita pelajari:**

✓ **cAdvisor** → metrics per container (CPU, RAM, network container)  
✓ **Node Exporter** → metrics level OS/host (CPU, disk, network host)  
✓ **Prometheus** → time-series database, scrape & store metrics, PromQL  
✓ **Grafana** → visualisasi dashboard interaktif  
✓ **Docker Compose** → orkestrasi multi-container dalam 1 file  

**Takeaway utama:**
> Monitoring bukan opsional — ini kebutuhan wajib sistem production. Stack Prometheus + Grafana adalah standar industri yang digunakan di ribuan perusahaan di seluruh dunia.

---

# SLIDE 30 — Q&A

## Pertanyaan?

*"The goal of monitoring is to enable humans to make better decisions, faster."*

**Kontak dosen:**  
*(isi email/info kontak)*

---

## Lampiran: File Lengkap docker-compose.yml

```yaml
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data: {}
  grafana_data: {}

services:

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - monitoring
    depends_on:
      - prometheus
```

# Monitoring Stack: Docker Compose + Prometheus + Grafana

Panduan lengkap membangun stack monitoring dengan cAdvisor, Node Exporter, Prometheus, dan Grafana menggunakan Docker Compose.

---

## Daftar Isi

- [Struktur Project](#struktur-project)
- [Cara Menjalankan](#cara-menjalankan)
- [Konfigurasi File](#konfigurasi-file)
- [Visualisasi & Query](#visualisasi--query)
  - [Cara Kerja PromQL](#1-cara-kerja-promql)
  - [Tipe Data PromQL](#2-tipe-data-promql)
  - [Fungsi PromQL](#3-fungsi-promql-yang-sering-dipakai)
  - [Query Siap Pakai](#4-query-siap-pakai)
  - [Membuat Panel Grafana](#5-membuat-panel-di-grafana--step-by-step)
  - [Contoh Panel Lengkap](#6-contoh-panel-lengkap)
  - [Variable Filter Dinamis](#7-menggunakan-variable-filter-dinamis)
  - [Tips Unit Formatting](#8-tips-unit-formatting)
- [Materi PPT](#materi-ppt)

---

## Struktur Project

```
monitoring-stack/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── prometheus.yml
        └── dashboards/
            └── dashboard.yml
```

---

## Cara Menjalankan

```bash
# Jalankan semua service
docker compose up -d

# Verifikasi container berjalan
docker compose ps

# Cek log
docker compose logs -f prometheus

# Hentikan
docker compose down
```

| Service | URL |
|---------|-----|
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| cAdvisor | http://localhost:8080 |
| Node Exporter metrics | http://localhost:9100/metrics |

Login Grafana default: `admin` / `admin123`

---

## Konfigurasi File

### docker-compose.yml

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

### prometheus/prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

### grafana/provisioning/datasources/prometheus.yml

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

---

## Visualisasi & Query

### 1. Cara Kerja PromQL

Prometheus menyimpan data sebagai **time-series** — setiap metric punya label sebagai identitas:

```
metric_name{label1="value1", label2="value2"} nilai timestamp
```

Contoh data mentah yang disimpan Prometheus:

```
container_memory_usage_bytes{name="nginx", instance="localhost:8080"} 52428800
container_memory_usage_bytes{name="postgres", instance="localhost:8080"} 134217728
node_cpu_seconds_total{cpu="0", mode="idle"} 12345.67
node_cpu_seconds_total{cpu="0", mode="user"} 234.56
```

---

### 2. Tipe Data PromQL

#### Instant Vector — nilai sekarang

```promql
# Semua metric memory container
container_memory_usage_bytes

# Filter by label — hanya container bernama "nginx"
container_memory_usage_bytes{name="nginx"}

# Exclude label — semua kecuali yang name kosong
container_memory_usage_bytes{name!=""}

# Regex match — container yang namanya diawali "app"
container_memory_usage_bytes{name=~"app.*"}
```

#### Range Vector — nilai dalam rentang waktu (wajib pakai fungsi)

```promql
# Data 5 menit terakhir — TIDAK bisa langsung di-plot
container_cpu_usage_seconds_total[5m]

# Harus dibungkus fungsi aggregasi:
rate(container_cpu_usage_seconds_total[5m])
```

---

### 3. Fungsi PromQL yang Sering Dipakai

#### `rate()` — laju perubahan per detik (untuk Counter)

```promql
# CPU usage rate per container
rate(container_cpu_usage_seconds_total[5m])

# Network bytes masuk per detik
rate(node_network_receive_bytes_total[5m])
```

> Gunakan `rate()` untuk metric bertipe **Counter** (nilainya hanya naik terus).

#### `irate()` — rate instan (lebih responsif, lebih "spiky")

```promql
irate(container_cpu_usage_seconds_total[5m])
```

#### `avg_over_time()` — rata-rata dalam window waktu

```promql
avg_over_time(node_load1[10m])
```

#### Aggregasi: `sum()`, `avg()`, `max()`, `min()`, `count()`

```promql
# Total memory semua container dijumlah
sum(container_memory_usage_bytes{name!=""})

# Rata-rata CPU per instance
avg by (instance) (rate(node_cpu_seconds_total[5m]))

# Memory tertinggi — top 5
topk(5, container_memory_usage_bytes{name!=""})

# Jumlah container yang running
count(container_last_seen{name!=""})
```

#### `by()` dan `without()` — grouping label

```promql
# Sum CPU, kelompokkan per nama container
sum by (name) (rate(container_cpu_usage_seconds_total[5m]))

# Sum semua, hilangkan label "cpu" dari grouping
sum without (cpu) (rate(node_cpu_seconds_total[5m]))
```

---

### 4. Query Siap Pakai

#### CPU

```promql
# CPU usage host dalam persen
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)

# CPU usage per container (persen)
sum by (name) (rate(container_cpu_usage_seconds_total{name!=""}[5m])) * 100

# CPU breakdown by mode (user, system, iowait, dll)
avg by (mode) (rate(node_cpu_seconds_total[5m])) * 100
```

#### Memory

```promql
# RAM used host (GB)
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / 1024^3

# RAM usage host dalam persen
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# RAM per container (MB)
container_memory_usage_bytes{name!=""} / 1024 / 1024

# RAM limit vs usage per container (persen)
container_memory_usage_bytes{name!=""} / container_spec_memory_limit_bytes{name!=""} * 100
```

#### Disk

```promql
# Disk tersedia (GB)
node_filesystem_avail_bytes{mountpoint="/"} / 1024^3

# Disk usage persen
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Disk read speed (MB/s)
rate(node_disk_read_bytes_total[5m]) / 1024 / 1024

# Disk write speed (MB/s)
rate(node_disk_written_bytes_total[5m]) / 1024 / 1024
```

#### Network

```promql
# Bandwidth masuk host (MB/s)
rate(node_network_receive_bytes_total{device="eth0"}[5m]) / 1024 / 1024

# Bandwidth keluar host (MB/s)
rate(node_network_transmit_bytes_total{device="eth0"}[5m]) / 1024 / 1024

# Network per container (bytes/s)
rate(container_network_receive_bytes_total{name!=""}[5m])
rate(container_network_transmit_bytes_total{name!=""}[5m])
```

---

### 5. Membuat Panel di Grafana — Step by Step

**Langkah 1: Buka Dashboard Editor**
```
Grafana → Dashboards → New Dashboard → Add visualization
```

**Langkah 2: Pilih Data Source**

Pilih **Prometheus** dari dropdown data source.

**Langkah 3: Tulis Query**

Di bagian bawah panel ada tab **Query**. Contoh membuat panel CPU:

```
Query A:
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)

Legend: CPU Usage %
```

**Langkah 4: Pilih Tipe Visualisasi**

Di panel kanan atas, pilih tipe panel:

| Tipe Panel | Cocok Untuk |
|------------|-------------|
| **Time series** | Data berubah waktu (CPU, RAM, network) |
| **Gauge** | Nilai saat ini dalam persen (disk usage) |
| **Stat** | Angka tunggal besar (total RAM, uptime) |
| **Bar chart** | Perbandingan antar container |
| **Table** | Banyak metric sekaligus dalam tabel |
| **Heatmap** | Distribusi nilai (latency distribution) |

**Langkah 5: Konfigurasi Panel Options**

```
Panel title   : "CPU Usage Host"
Description   : "Persentase CPU yang digunakan"

Unit          : Percent (0-100)
Min           : 0
Max           : 100

Thresholds:
  Green  → 0–60%
  Yellow → 60–80%
  Red    → 80–100%
```

---

### 6. Contoh Panel Lengkap

#### Panel 1: Time Series — CPU Usage

```
Query:
  100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)

Visualization : Time series
Unit          : Percent (0-100)
Thresholds    : 80 = red, 60 = yellow
Fill opacity  : 20
```

#### Panel 2: Gauge — Memory Usage

```
Query:
  (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

Visualization : Gauge
Unit          : Percent (0-100)
Min / Max     : 0 / 100
Thresholds    : 90 = red, 70 = yellow
```

#### Panel 3: Stat — Total Container Running

```
Query:
  count(container_last_seen{name!=""})

Visualization : Stat
Color mode    : Background
Unit          : short
```

#### Panel 4: Bar Chart — RAM per Container

```
Query:
  topk(10, container_memory_usage_bytes{name!=""}) / 1024 / 1024

Legend        : {{name}}
Visualization : Bar chart
Unit          : megabytes (MB)
Orientation   : Horizontal
```

#### Panel 5: Time Series — Network In/Out

```
Query A:
  rate(node_network_receive_bytes_total{device="eth0"}[5m]) / 1024 / 1024
  Legend: ↓ Inbound

Query B:
  rate(node_network_transmit_bytes_total{device="eth0"}[5m]) / 1024 / 1024
  Legend: ↑ Outbound

Unit          : megabytes/sec (MBs)
```

---

### 7. Menggunakan Variable (Filter Dinamis)

Variable membuat dashboard bisa difilter tanpa harus edit query.

**Buat variable:**

```
Dashboard Settings → Variables → Add variable

Name  : container
Type  : Query
Query : label_values(container_memory_usage_bytes{name!=""}, name)
```

**Pakai di query:**

```promql
container_memory_usage_bytes{name="$container"}
```

Sekarang ada dropdown di atas dashboard untuk memilih container mana yang ingin dilihat.

---

### 8. Tips Unit Formatting

Selalu set **Unit** di panel agar angka terbaca manusia:

| Data | Unit di Grafana |
|------|----------------|
| Bytes | `bytes (IEC)` |
| Bytes/sec | `bytes/sec (IEC)` |
| Persen | `Percent (0-100)` |
| Detik | `seconds (s)` |
| Milidetik | `milliseconds (ms)` |
| Jumlah biasa | `short` |

> Tanpa unit, grafik akan menampilkan angka mentah seperti `52428800` alih-alih `50 MiB`.

---

### Dashboard Siap Pakai (Import dari Grafana.com)

| ID | Nama | Untuk |
|----|------|-------|
| **1860** | Node Exporter Full | Metrics host lengkap |
| **193** | Docker Monitoring | Metrics container |
| **11600** | Docker & System Monitoring | Kombinasi host + container |

**Cara import:**
1. Grafana → **Dashboards** → **Import**
2. Masukkan ID → **Load**
3. Pilih data source **Prometheus** → **Import**

---

## Materi PPT

Lihat file [docker-monitoring-stack.md](docker-monitoring-stack.md) untuk 30 slide materi kuliah lengkap.

**Konversi ke PowerPoint:**

```bash
# Opsi 1 — Marp
npm install -g @marp-team/marp-cli
marp docker-monitoring-stack.md -o slide.pptx

# Opsi 2 — Pandoc
pandoc docker-monitoring-stack.md -o slide.pptx
```

---

## Referensi

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [cAdvisor GitHub](https://github.com/google/cadvisor)
- [Node Exporter GitHub](https://github.com/prometheus/node_exporter)
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)
- [Grafana Dashboard Hub](https://grafana.com/grafana/dashboards/)

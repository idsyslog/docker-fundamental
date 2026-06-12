# Tugas Praktikum
## Monitoring Infrastruktur dengan Docker Compose + Prometheus + Grafana

| | |
|---|---|
| **Mata Kuliah** | Sistem Terdistribusi / DevOps / Cloud Computing |
| **Topik** | Container Monitoring Stack |
| **Sifat** | Individu |
| **Waktu** | 120 menit |

---

## Tujuan Pembelajaran

Setelah menyelesaikan praktikum ini, mahasiswa mampu:

1. Membangun stack monitoring multi-container menggunakan Docker Compose
2. Mengonfigurasi Prometheus untuk scrape metrics dari exporter
3. Membuat dashboard visualisasi di Grafana menggunakan PromQL
4. Memahami alur data dari exporter → Prometheus → Grafana

---

## Prasyarat

Pastikan sudah terinstall di sistem kamu:

- Docker Engine
- Docker Compose v2+
- Text editor (VS Code, Nano, dll)
- Browser

Cek dengan menjalankan:

```bash
docker --version
docker compose version
```

---

## Deskripsi Tugas

Kamu diminta membangun sebuah **monitoring stack** dari nol menggunakan Docker Compose yang terdiri dari 4 service:

| Service | Fungsi |
|---------|--------|
| **cAdvisor** | Monitor resource usage per container |
| **Node Exporter** | Monitor metrics level OS/host |
| **Prometheus** | Scrape & simpan metrics (time-series DB) |
| **Grafana** | Visualisasi dashboard |

---

## Bagian A — Setup Project (30 poin)

### A1. Buat Struktur Folder (5 poin)

Buat struktur folder berikut di komputer kamu:

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

**Perintah yang perlu kamu jalankan:**

```bash
mkdir -p monitoring-stack/prometheus
mkdir -p monitoring-stack/grafana/provisioning/datasources
mkdir -p monitoring-stack/grafana/provisioning/dashboards
cd monitoring-stack
```

---

### A2. Buat `docker-compose.yml` (15 poin)

Buat file `docker-compose.yml` yang mendefinisikan 4 service berikut.

**Ketentuan:**

- Semua service harus berada dalam **satu network** bernama `monitoring`
- Gunakan **named volume** untuk menyimpan data Prometheus dan Grafana agar data tidak hilang saat container restart
- cAdvisor memerlukan akses ke Docker daemon host — tambahkan opsi yang diperlukan agar metrics container terbaca
- Node Exporter harus mount `/proc`, `/sys`, dan `/` dari host dan menggunakan flag path yang sesuai
- Prometheus harus mount file `prometheus.yml` dan menyimpan data di volume, dengan retention 15 hari dan lifecycle API aktif
- Grafana harus mount folder `grafana/provisioning` dan dikonfigurasi dengan environment variable untuk username/password admin

**Port yang harus di-expose:**

| Service | Port |
|---------|------|
| cAdvisor | 8080 |
| Node Exporter | 9100 |
| Prometheus | 9090 |
| Grafana | 3000 |

**Image yang digunakan:**

| Service | Image |
|---------|-------|
| cAdvisor | `gcr.io/cadvisor/cadvisor:latest` |
| Node Exporter | `prom/node-exporter:latest` |
| Prometheus | `prom/prometheus:latest` |
| Grafana | `grafana/grafana:latest` |

> 💡 **Hint:** Lihat dokumentasi resmi masing-masing image di Docker Hub untuk melihat volume dan flag yang diperlukan.

---

### A3. Buat `prometheus/prometheus.yml` (5 poin)

Konfigurasi Prometheus agar melakukan scrape ke 3 target:

1. Prometheus sendiri (self-monitoring)
2. Node Exporter
3. cAdvisor

**Ketentuan:**

- `scrape_interval` dan `evaluation_interval` : **15 detik**
- Gunakan **nama service Docker** sebagai hostname target (bukan `localhost`)

> ❓ **Pertanyaan:** Mengapa kita menggunakan nama service Docker dan bukan `localhost` untuk target scrape?  
> Tulis jawabanmu di file `jawaban.md`.

---

### A4. Buat Grafana Provisioning Files (5 poin)

**File 1:** `grafana/provisioning/datasources/prometheus.yml`

Konfigurasi agar Grafana otomatis terhubung ke Prometheus saat pertama kali start.

Ketentuan:
- `type`: prometheus
- `url`: gunakan nama service Docker (bukan localhost)
- Jadikan sebagai data source default

**File 2:** `grafana/provisioning/dashboards/dashboard.yml`

Konfigurasi dashboard provider agar Grafana bisa auto-load dashboard dari folder provisioning.

---

## Bagian B — Menjalankan Stack (20 poin)

### B1. Start Stack (5 poin)

Jalankan semua container:

```bash
docker compose up -d
```

### B2. Verifikasi Container (5 poin)

Pastikan semua container berstatus **Up**:

```bash
docker compose ps
```

**Screenshot** output perintah di atas dan sertakan dalam laporan.

### B3. Verifikasi Prometheus Targets (10 poin)

Buka `http://localhost:9090/targets` di browser.

Pastikan **semua 3 target berstatus UP** (hijau).

**Screenshot** halaman targets dan sertakan dalam laporan.

> ❓ **Pertanyaan:** Apa yang terjadi jika salah satu target berstatus DOWN? Bagaimana cara mendiagnosis masalahnya?  
> Tulis jawabanmu di file `jawaban.md`.

---

## Bagian C — Visualisasi Grafana (50 poin)

Buka Grafana di `http://localhost:3000` dan login dengan kredensial yang kamu set di `docker-compose.yml`.

Buat **satu dashboard baru** dengan nama: `[NIM] - Monitoring Dashboard`

Dashboard harus memiliki **minimal 5 panel** berikut:

---

### C1. Panel: CPU Usage Host (10 poin)

**Tipe panel:** Time series  
**Ketentuan:**
- Tampilkan persentase CPU yang digunakan oleh host
- Unit: `Percent (0-100)`
- Range: 0–100%
- Threshold: kuning di 60%, merah di 80%
- Refresh otomatis setiap 30 detik

**Metric yang digunakan:** `node_cpu_seconds_total`

> 💡 **Hint:** Gunakan fungsi `rate()` dan filter mode `idle` untuk menghitung CPU yang sedang dipakai.

---

### C2. Panel: Memory Usage Host (10 poin)

**Tipe panel:** Gauge  
**Ketentuan:**
- Tampilkan persentase RAM yang digunakan
- Unit: `Percent (0-100)`
- Threshold: kuning di 70%, merah di 90%

**Metric yang digunakan:** `node_memory_MemAvailable_bytes` dan `node_memory_MemTotal_bytes`

> 💡 **Hint:** RAM yang dipakai = Total - Available. Ubah ke persen dengan membagi total.

---

### C3. Panel: Running Containers (10 poin)

**Tipe panel:** Stat  
**Ketentuan:**
- Tampilkan jumlah container yang sedang berjalan
- Unit: `short` (angka bulat)
- Warna background: biru

**Metric yang digunakan:** `container_cpu_usage_seconds_total`

> 💡 **Hint:** Gunakan fungsi `count()` dengan filter `name!=""` untuk exclude container system.

---

### C4. Panel: Memory per Container (10 poin)

**Tipe panel:** Bar gauge (horizontal)  
**Ketentuan:**
- Tampilkan penggunaan RAM setiap container
- Tampilkan Top 10 container
- Unit: `bytes (IEC)`
- Legend menampilkan nama container

**Metric yang digunakan:** `container_memory_usage_bytes`

> 💡 **Hint:** Gunakan `topk()` untuk mengambil Top N, dan filter `name!=""`.

---

### C5. Panel Bebas — Pilih Sendiri (10 poin)

Buat **1 panel tambahan** sesuai kreativitasmu. Pilih salah satu:

**Opsi A — Network Traffic:**
- Tampilkan bandwidth masuk dan keluar host
- Gunakan 2 query dalam 1 panel (In dan Out)
- Unit: `bytes/sec (IEC)`

**Opsi B — Disk Usage:**
- Tampilkan persentase penggunaan disk root partition `/`
- Tipe panel: Stat atau Gauge

**Opsi C — Disk I/O Speed:**
- Tampilkan kecepatan baca/tulis disk
- Gunakan 2 query (Read dan Write)
- Unit: `bytes/sec (IEC)`

**Opsi D — Load Average:**
- Tampilkan load average 1 menit, 5 menit, dan 15 menit dalam satu panel
- Tipe panel: Time series atau Stat

> ❓ **Pertanyaan:** Jelaskan panel yang kamu buat — metric apa yang digunakan, mengapa kamu memilih tipe panel tersebut, dan apa insight yang bisa didapat dari panel ini?  
> Tulis jawabanmu di file `jawaban.md`.

---

## Bagian D — Bonus (20 poin)

### D1. CPU Usage per Container (5 poin)

Buat panel **Time series** yang menampilkan CPU usage per container secara individual dalam satu grafik — setiap container ditampilkan sebagai satu garis berbeda warna.

---

### D2. Variable Filter Dinamis (5 poin)

Tambahkan **variable dashboard** agar user bisa memfilter berdasarkan nama container dari dropdown menu.

Langkah:
1. Dashboard Settings → Variables → Add variable
2. Name: `container`
3. Type: Query
4. Query: `label_values(container_memory_usage_bytes{name!=""}, name)`

Setelah variable dibuat, ubah query salah satu panel agar menggunakan `{name="$container"}`.

---

### D3. Deploy ke VM / Server (5 poin)

Deploy monitoring stack ini ke VM atau server (bukan localhost). Sertakan:
- Screenshot akses dari browser menggunakan IP VM/server
- IP address VM

---

### D4. Tambahkan Container Aplikasi (5 poin)

Tambahkan container aplikasi (web server Nginx, atau aplikasi lain) ke `docker-compose.yml` dan pastikan container tersebut:
- Termonitor di cAdvisor (muncul di panel container)
- Terlihat CPU dan RAM-nya di Grafana

---

## Format Pengumpulan

Kumpulkan dalam satu folder ZIP dengan nama: `NIM_Nama_Monitoring.zip`

Isi folder:

```
NIM_Nama_Monitoring/
├── monitoring-stack/           ← seluruh file project
│   ├── docker-compose.yml
│   ├── prometheus/
│   │   └── prometheus.yml
│   └── grafana/
│       └── provisioning/
│           ├── datasources/prometheus.yml
│           └── dashboards/dashboard.yml
├── jawaban.md                  ← jawaban pertanyaan teori
└── screenshot/                 ← semua screenshot bukti
    ├── 01-docker-compose-ps.png
    ├── 02-prometheus-targets.png
    ├── 03-grafana-dashboard.png
    ├── 04-panel-cpu.png
    ├── 05-panel-memory.png
    ├── 06-panel-containers.png
    ├── 07-panel-memory-container.png
    └── 08-panel-bebas.png
```

---

## Rubrik Penilaian

| Bagian | Komponen | Poin |
|--------|----------|------|
| **A** | Struktur folder benar | 5 |
| | docker-compose.yml lengkap & jalan | 15 |
| | prometheus.yml — 3 target scrape | 5 |
| | Grafana provisioning files | 5 |
| **B** | Semua container Up | 5 |
| | Screenshot docker compose ps | 5 |
| | Semua target Prometheus UP + screenshot | 10 |
| **C** | Panel CPU Host | 10 |
| | Panel Memory Host | 10 |
| | Panel Running Containers | 10 |
| | Panel Memory per Container | 10 |
| | Panel Bebas + jawaban pertanyaan | 10 |
| **D** | CPU per Container | 5 |
| | Variable Filter Dinamis | 5 |
| | Deploy ke VM/Server | 5 |
| | Tambah container aplikasi | 5 |
| **Total** | | **120 poin** |

> Nilai akhir = (Total poin / 100) × 100. Maksimum 100 (bonus bisa menutup kekurangan nilai wajib).

---

## Pertanyaan Teori (Bagian Jawaban.md)

Jawab pertanyaan-pertanyaan berikut dalam file `jawaban.md`:

1. Mengapa Prometheus menggunakan nama **service Docker** dan bukan `localhost` sebagai target scrape?

2. Apa perbedaan utama antara **cAdvisor** dan **Node Exporter**? Berikan contoh metric yang dihasilkan masing-masing.

3. Mengapa Prometheus menggunakan model **pull** (menarik data) bukan **push** (mengirim data)?

4. Apa fungsi **named volume** (`prometheus_data`, `grafana_data`) dalam `docker-compose.yml`? Apa yang terjadi jika volume ini tidak didefinisikan?

5. Jelaskan panel yang kamu buat di bagian C5 — metric apa yang digunakan, mengapa memilih tipe panel tersebut, dan apa insight yang bisa diambil.

---

## Tips & Referensi

| Sumber | Link |
|--------|------|
| Prometheus Docs | https://prometheus.io/docs/ |
| Grafana Docs | https://grafana.com/docs/ |
| cAdvisor GitHub | https://github.com/google/cadvisor |
| Node Exporter GitHub | https://github.com/prometheus/node_exporter |
| PromQL Cheat Sheet | https://promlabs.com/promql-cheat-sheet/ |
| Grafana Dashboards Hub | https://grafana.com/grafana/dashboards/ |

> **Dilarang** mengumpulkan file konfigurasi yang identik dengan teman. Setiap mahasiswa wajib membuat panel Grafana sendiri (query boleh sama, layout dan konfigurasi visual harus berbeda).

---

*Selamat mengerjakan! Jika ada pertanyaan, tanyakan kepada asisten praktikum atau dosen.*

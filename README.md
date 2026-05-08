# docker-fundamental
# Belajar Docker — TazkiaTech Sample App

Project sederhana untuk latihan build dan deploy Docker dengan berbeda versi.

---

## Struktur Project

```
belajar-docker/
├── Dockerfile          # instruksi build image
├── docker-compose.yml  # jalankan semua versi sekaligus
├── .dockerignore       # file yang tidak ikut di-build
├── index.html          # halaman web (versi ada di footer)
└── README.md
```

---

## Prasyarat

Pastikan Docker sudah terinstall:

```bash
docker --version
```

---

## Latihan 1 — Build & Run Pertama

```bash
# 1. Build image
docker build -t tazkiatech:1.0 .

# 2. Jalankan container
docker run -d -p 8080:80 --name tazkia-v1 tazkiatech:1.0

# 3. Buka di browser
# http://localhost:8080
```

Cek footer halaman — seharusnya tampil `v1.0.0` dan `staging`.

---

## Latihan 2 — Deploy Versi Baru

1. Edit `index.html`, ubah bagian footer:

```html
<!-- sebelum -->
<span class="ver">v1.0.0</span>
<span class="env-tag">staging</span>

<!-- sesudah -->
<span class="ver">v2.0.0</span>
<span class="env-tag">production</span>
```

2. Build image baru dan jalankan di port berbeda:

```bash
docker build -t tazkiatech:2.0 .
docker run -d -p 8081:80 --name tazkia-v2 tazkiatech:2.0
```

3. Bandingkan keduanya:
   - `http://localhost:8080` → v1.0.0 staging
   - `http://localhost:8081` → v2.0.0 production

---

## Perintah Berguna

```bash
# Lihat semua container yang berjalan
docker ps

# Lihat semua image lokal
docker images

# Stop container
docker stop tazkia-v1

# Hapus container
docker rm tazkia-v1

# Stop dan hapus sekaligus
docker rm -f tazkia-v1

# Lihat log container
docker logs tazkia-v1

# Masuk ke dalam container
docker exec -it tazkia-v1 sh
```

---

## Latihan 3 — Docker Compose

Jalankan v1 dan v2 sekaligus tanpa perlu ketik banyak perintah.

> **Sebelum mulai:** pastikan `index.html` sudah ada dua kondisi berbeda untuk v1 dan v2.
> Compose menggunakan file `index.html` yang sama saat ini — ubah dulu sebelum build masing-masing,
> atau gunakan dua folder terpisah untuk simulasi yang lebih nyata.

```bash
# Jalankan semua service sekaligus
docker compose up -d

# Cek status
docker compose ps

# Lihat log semua service
docker compose logs -f

# Lihat log satu service saja
docker compose logs -f tazkia-v1

# Stop semua
docker compose down
```

Setelah `up`, buka di browser:

| URL | Container |
|-----|-----------|
| http://localhost:8081 | tazkia-v1 |
| http://localhost:8082 | tazkia-v2 |

---

## Latihan 4 — Push ke Docker Hub (Container Registry)

Setelah image jadi di lokal, kita upload ke Docker Hub supaya bisa di-pull dari mana saja.

### 4.1 Buat Akun & Login

Daftar di [hub.docker.com](https://hub.docker.com) kalau belum punya akun, lalu login:

```bash
docker login
# masukkan username dan password Docker Hub kamu
```

### 4.2 Tag Image

Image lokal perlu diberi nama sesuai format Docker Hub sebelum bisa di-push:

```
<username>/<nama-image>:<tag>
```

```bash
# Tag v1
docker tag tazkiatech:1.0 <username>/tazkiatech:1.0

# Tag v2
docker tag tazkiatech:2.0 <username>/tazkiatech:2.0

# Cek hasilnya
docker images
```

### 4.3 Push ke Docker Hub

```bash
docker push <username>/tazkiatech:1.0
docker push <username>/tazkiatech:2.0
```

Setelah selesai, image bisa dilihat di:
`https://hub.docker.com/r/<username>/tazkiatech`

---

## Latihan 5 — Pull dari Docker Hub (Simulasi Server Lain)

Ini mensimulasikan kondisi di mana kita deploy ke server yang belum punya image-nya sama sekali.

### 5.1 Hapus image lokal dulu

```bash
# Hapus semua container yang pakai image ini
docker rm -f tazkia-v1 tazkia-v2

# Hapus image dari lokal
docker rmi tazkiatech:1.0 tazkiatech:2.0
docker rmi <username>/tazkiatech:1.0 <username>/tazkiatech:2.0

# Pastikan sudah bersih
docker images
```

### 5.2 Pull dari Docker Hub

```bash
docker pull <username>/tazkiatech:1.0
docker pull <username>/tazkiatech:2.0
```

### 5.3 Jalankan dari image hasil pull

```bash
docker run -d -p 8080:80 --name tazkia-v1 <username>/tazkiatech:1.0
docker run -d -p 8081:80 --name tazkia-v2 <username>/tazkiatech:2.0
```

Buka browser dan bandingkan:
- `http://localhost:8080` → v1.0.0 staging
- `http://localhost:8081` → v2.0.0 production

---

## Perbandingan: Lokal vs Docker Hub

| | **Build Lokal** | **Pull dari Docker Hub** |
|---|---|---|
| **Sumber image** | Dockerfile di mesin sendiri | Registry di internet |
| **Perlu source code?** | ✅ Ya | ❌ Tidak |
| **Cocok untuk** | Development & testing | Deployment ke server |
| **Perintah** | `docker build` | `docker pull` |
| **Bisa dipakai orang lain?** | ❌ Hanya di mesin sendiri | ✅ Siapa saja bisa pull |
| **Kecepatan pertama kali** | Tergantung koneksi (base image) | Tergantung ukuran image |

> **Alur yang benar di dunia nyata:**
> Developer build di lokal → push ke registry → server pull dari registry → jalankan container

---

## Catatan

- Versi ada di **footer** halaman — ubah langsung di `index.html` sebelum build
- Setiap `docker build` membuat image baru — image lama tetap ada
- Gunakan port berbeda (`-p`) kalau mau jalankan beberapa versi sekaligus
- Ganti `<username>` dengan username Docker Hub kamu yang sebenarnya

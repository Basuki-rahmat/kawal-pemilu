# KAWAL PEMILU - PROVINSI LAMPUNG

Sistem Monitoring dan Manajemen Pemenangan Pemilu terintegrasi untuk wilayah Provinsi Lampung.

---

## Arsitektur Sistem

```
Browser → Nginx → [API Node.js] → Redis Queue → [Worker] → MySQL
          ↓
     [S3/MinIO] ← Upload foto langsung (Presigned URL)
          ↓
     [PDF Service] ← Microservice PHP (mPDF)
```

**Prinsip Utama:**
- Gambar **tidak pernah** lewat Node.js → upload langsung ke S3 via Presigned URL
- Data teks **tidak pernah** insert langsung ke MySQL dari API → semua lewat Redis Queue (BullMQ)
- Dashboard dibaca dari **Redis cache**, bukan query langsung ke MySQL per request

---

## Tech Stack

| Komponen | Teknologi | Keterangan |
| --- | --- | --- |
| Backend API | Node.js + Fastify | REST API, port 3000 |
| Database | MySQL 8.x | Nama DB: `kawal_pemilu` |
| Queue / Cache | Redis + BullMQ | AOF + snapshot untuk persistensi |
| Object Storage | MinIO (lokal) / S3 (produksi) | Bucket: `c1-uploads` |
| Frontend | HTML + Tailwind CSS + Vanilla JS | Mobile-first, CDN |
| PDF Generator | PHP 8 + mPDF | Microservice terpisah, port 8000 |
| Auth | scrypt + HMAC stateless | Password/PIN scrypt, token HMAC SHA256 |

---

## Data Master Tersedia

Aplikasi sudah terintegrasi dengan data master MySQL untuk Provinsi Lampung:

| Level Wilayah | Jumlah |
| --- | --- |
| Kabupaten/Kota | 15 |
| Kecamatan | 229 |
| Desa/Kelurahan | 2.642 |
| TPS | 27.440 |

---

## Hierarki Pengurus & Wilayah

Sistem mendukung manajemen pengurus berjenjang sesuai struktur organisasi:

| Tingkat | Level | Cakupan |
| --- | --- | --- |
| **DPW** | Provinsi | Seluruh Provinsi Lampung |
| **DPC** | Kabupaten/Kota | Satu kabupaten/kota |
| **PAC** | Kecamatan | Satu kecamatan |
| **Ranting** | Desa/Kelurahan | Satu desa |
| **Anak Ranting** | TPS/Dusun | Satu TPS |

---

## Fitur Utama

### Dashboard
- Desain responsif dengan Header & Toggle Menu yang mobile-friendly
- Real Count: rekap perhitungan internal dari Form C1 saksi
- Sirekap: hasil *scraping* Sirekap KPU sesuai Dapil
- Multi-User: dashboard khusus untuk Admin dan berbagai level User

### Manajemen Data & Pendaftaran
- **Pengurus:** Input data sesuai tingkat hierarki (DPW hingga Anak Ranting)
- **Kandidat:** Manajemen calon Pilgub, Pilbup, Caleg, dan DPD
- **Saksi:** Penempatan saksi di tiap TPS (Master TPS)
- **Dapil & TPS:** Pengaturan wilayah pemilihan

### Manajemen Biaya & Logistik
- **Logistik:** Pencatatan distribusi alat peraga kampanye
- **Dana Saksi:** Manajemen pembayaran dan operasional saksi
- **Export PDF:** Laporan otomatis menggunakan mPDF

### Autentikasi
- **Admin:** Login dengan username + password (session-based)
- **Saksi:** Login dengan nomor HP + PIN (HMAC stateless token)
- **Hak Akses:** Hierarki pengurus menentukan visibilitas data

---

## Struktur Folder

```
kawal-pemilu/
├── db/
│   ├── db.sql                          # Master data (sudah ada)
│   └── migrations/
│       └── 001_operational.sql         # Tabel operasional + view
│
├── api/                                # Backend Node.js
│   ├── src/
│   │   ├── server.js                   # Entry point API
│   │   ├── worker.js                   # Entry point Worker
│   │   ├── config.js                   # Konfigurasi environment
│   │   ├── lib/                        # Utilitas (crypto, env parser)
│   │   ├── middleware/                 # Auth, reject-multipart
│   │   ├── routes/                     # API endpoints
│   │   └── services/                   # MySQL, Redis, S3, Queue, dll
│   ├── scripts/                        # Migrate, seed, dlq-replay
│   └── test/                           # Unit test
│
├── pdf-service/                        # Microservice PHP
│   ├── generate.php                    # POST JSON → PDF
│   └── composer.json
│
├── frontend/                           # Frontend statis
│   ├── index.html                      # SPA shell
│   ├── admin.html                      # Dashboard admin
│   ├── saksi.html                      # Dashboard saksi
│   └── assets/                         # CSS, JS, gambar
│
└── deploy/                             # Konfigurasi produksi
    ├── nginx/                          # Reverse proxy config
    ├── mysql/                          # MySQL tuning
    ├── redis/                          # Redis config
    ├── pm2/                            # Process manager
    └── scripts/                        # Deployment scripts
```

---

## Setup Localhost

### Prasyarat

- Node.js 20 LTS
- MySQL 8.x (Laragon / XAMPP / Docker)
- Redis 8.x
- MinIO (Docker)
- PHP 8.x + Composer (untuk PDF service)

### Langkah Setup

```bash
# 1) Clone repository
git clone https://github.com/username/kawal-pemilu.git
cd kawal-pemilu

# 2) Setup MinIO (S3 lokal)
docker run -d --name minio -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin -e MINIO_ROOT_PASSWORD=minioadmin \
  -v minio-data:/data \
  minio/minio server /data --console-address ":9001"
# Buka http://localhost:9001 → buat bucket "c1-uploads" → set PRIVATE

# 3) Setup Redis
docker run -d --name redis -p 6379:6379 \
  -v redis-data:/data --appendonly yes redis:8-alpine

# 4) Setup Database
cd api
npm install
cp .env.example .env
# Edit .env sesuai kebutuhan

npm run migrate    # Import db.sql + 001_operational.sql
npm run seed       # Seed admin & saksi dev

# 5) Jalankan Backend (2 terminal)
npm run start:api      # Terminal 1: API di port 3000
npm run start:worker   # Terminal 2: Worker queue

# 6) Jalankan PDF Service
cd ../pdf-service
composer install
PDF_SERVICE_TOKEN=change-me php -S 0.0.0.0:8000 -t .

# 7) Buka Frontend
# Buka frontend/index.html di browser
# Login: admin / dev123 atau 081234567890 / dev123
```

---

## Environment Variables

Buat file `api/.env` berdasarkan `api/.env.example`:

```bash
# === SERVER ===
NODE_ENV=development
PORT=3000

# === DATABASE ===
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=root
DB_PASS=
DB_NAME=kawal_pemilu
DB_POOL=10

# === REDIS ===
REDIS_HOST=127.0.0.1
REDIS_PORT=6379

# === OBJECT STORAGE ===
S3_ENDPOINT=http://127.0.0.1:9000
S3_REGION=us-east-1
S3_BUCKET=c1-uploads
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
PRESIGNED_URL_EXPIRY=600
MAX_UPLOAD_SIZE_MB=10

# === PDF SERVICE ===
PDF_SERVICE_URL=http://127.0.0.1:8000
PDF_SERVICE_TOKEN=change-me

# === AUTH ===
APP_SECRET=change-me
SESSION_TTL_HOURS=24
LOGIN_MAX_ATTEMPTS=5

# === SCRAPER ===
SIRREKAP_INTERVAL_MIN=30
```

---

## API Endpoints

| Method | Endpoint | Auth | Keterangan |
| --- | --- | --- | --- |
| POST | `/api/v1/auth/admin/login` | - | Login admin |
| POST | `/api/v1/auth/saksi/login` | - | Login saksi |
| GET | `/api/v1/upload/presigned-url` | Saksi | Dapatkan URL upload ke S3 |
| POST | `/api/v1/saksi/c1` | Saksi | Submit Form C1 |
| GET | `/api/v1/saksi/c1/:tps_id/:kategori` | Saksi | Cek status submit |
| GET | `/api/v1/dashboard/rekap` | - | Rekap suara per dapil |
| GET | `/api/v1/dashboard/sirekap/:dapil_id` | - | Data Sirekap KPU |
| GET | `/api/v1/healthz` | - | Health check |
| GET | `/api/v1/admin/c1-pending` | Admin | Daftar C1 pending verifikasi |
| PUT | `/api/v1/admin/c1-verifikasi/:bukti_id` | Admin | Verifikasi C1 |
| GET | `/api/v1/admin/export/pdf/:jenis` | Admin | Export laporan PDF |

---

## Persiapan Hari H (High Traffic)

Sistem dirancang untuk dapat dipecah menjadi server terpisah guna menangani beban upload Form C1 yang sangat tinggi.

### Arsitektur Multi-Server

```
┌─────────────────────────────────────────────────────────┐
│                    HARI H: 5+ SERVER                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [LB: Nginx] ──┬── [API-1: Node.js]  ← Presigned URL  │
│                │── [API-2: Node.js]  ← Submit JSON     │
│                │                                        │
│                │── [Worker: Node.js + Redis]            │
│                │    └── BullMQ drain → MySQL            │
│                │                                        │
│                │── [MySQL: dedicated]                   │
│                │    └── 8GB RAM buffer pool             │
│                │                                        │
│  [S3: Object Storage] ← Upload foto langsung           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Skenario Beban

```
Hari H Pemilu:
- 27.440 TPS di Provinsi Lampung
- ±15.000 saksi aktif mengirim Form C1
- Peak: 5.000 request/menit (submit + upload foto)
- Setiap submit: 1 foto 2-5MB + JSON angka suara
```

### Strategi Scaling

| Fase | Server | Estimasi Biaya/bulan |
| --- | --- | --- |
| Pra-pemilu | 1 VPS (API + Worker + MySQL + Redis) | ±Rp389.000 |
| Hari H | 5 VPS (LB + 2 API + Worker/Redis + MySQL) | ±Rp4.168.050 |

---

## Estimasi Biaya Tahunan

| Komponen | Pra-pemilu (11 bulan) | Hari H (1 bulan) |
| --- | --- | --- |
| VPS | Rp4.279.000 | Rp3.096.000 |
| Object Storage | Rp110.000 | Rp100.000 |
| Domain + SSL | Rp90.750 | - |
| Monitoring | Rp110.000 | Rp10.000 |
| Lain-lain | - | Rp530.260 |
| **Subtotal** | **Rp4.589.750** | **Rp3.736.260** |

```
Total Tahunan: Rp8.326.010 (sudah PPN)
```

---

## Checklist Implementasi

### Tahap 1: Fondasi (Minggu 1-2)
- [ ] Setup MySQL + import `db.sql`
- [ ] Jalankan `001_operational.sql`
- [ ] Setup Redis + MinIO (Docker)
- [ ] Buat `api/.env` dari `.env.example`
- [ ] Implementasi services (mysql, redis, s3, queue)
- [ ] Implementasi middleware (auth, reject-multipart)
- [ ] Test: `npm run test` → semua hijau

### Tahap 2: API Auth & Upload (Minggu 3)
- [ ] Route auth: login admin + login saksi
- [ ] Route upload: presigned URL
- [ ] Test login berhasil
- [ ] Test presigned URL → upload ke MinIO

### Tahap 3: Queue & Worker (Minggu 4)
- [ ] Implementasi BullMQ queue
- [ ] Implementasi C1 processor
- [ ] Route saksi: submit C1 → queue → worker
- [ ] Test: submit → queue → worker insert ke MySQL
- [ ] Test: submit ulang idempotent (tidak ganda)

### Tahap 4: Dashboard & Rekap (Minggu 5)
- [ ] Route dashboard: rekap + sirekap + healthz
- [ ] Implementasi Redis cache untuk rekap
- [ ] Implementasi Sirekap scraper
- [ ] View `v_rekap_dapil` di MySQL

### Tahap 5: Admin Panel (Minggu 6)
- [ ] Route admin: list pending, verifikasi, export PDF
- [ ] Microservice PDF (mPDF)
- [ ] Test verifikasi → audit log tercatat

### Tahap 6: Frontend (Minggu 7-8)
- [ ] Login page (admin + saksi)
- [ ] Dashboard utama + stats cards
- [ ] Tabel rekap dengan filter
- [ ] Manajemen pengurus, kandidat, saksi
- [ ] Manajemen Dapil & TPS
- [ ] Manajemen keuangan/logistik
- [ ] Responsive testing (mobile + desktop)

### Tahap 7: Testing & Hardening (Minggu 9)
- [ ] Load test: k6 (target 500 req/s)
- [ ] Security audit
- [ ] Backup & restore test
- [ ] Dokumentasi API

---

## Perintah Berguna

```bash
# Development
npm run start:api        # Jalankan API server
npm run start:worker     # Jalankan BullMQ worker
npm run test             # Jalankan test
npm run lint             # Cek syntax

# Database
npm run migrate          # Import schema
npm run seed             # Seed data dev

# Production (PM2)
pm2 start deploy/pm2/api.ecosystem.config.js
pm2 start deploy/pm2/worker.ecosystem.config.js
pm2 logs                 # Lihat logs
```

---

## License

Proprietary - Untuk penggunaan internal Kawal Pemilu Provinsi Lampung.

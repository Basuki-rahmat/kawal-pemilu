# KAWAL PEMILU - PROVINSI LAMPUNG

## Panduan Implementasi Lengkap (Low-Level)

Dokumen ini merencanakan dan mengarahkan implementasi aplikasi web **Kawal Pemilu** tingkat Provinsi Lampung. Setiap bagian ditulis **sangat detail** agar bisa diimplementasikan oleh junior programmer atau model AI yang lebih murah.

> **Prinsip Utama:**
> - Gambar **TIDAK PERNAH** lewat Node.js → upload langsung ke S3 via Presigned URL.
> - Data teks **TIDAK PERNAH** insert langsung ke MySQL dari API → semua lewat Redis Queue (BullMQ).
> - Dashboard dibaca dari **Redis cache**, bukan query langsung ke MySQL per request.

---

# ============================================================
# BAGIAN PERTAMA: APLIKASI INTI (PRA-PEMILU)
# ============================================================

---

## BAB 1: TECH STACK & DEPENDENSI

### 1.1 Stack Teknologi

| Komponen         | Teknologi                                    | Versi        | Keterangan                              |
| ---------------- | -------------------------------------------- | ------------ | --------------------------------------- |
| Backend API      | Node.js + Fastify                            | 20 LTS / 4.x | REST API, port 3000                   |
| Database         | MySQL                                        | 8.x          | Nama DB: `kawal_pemilu`               |
| Queue / Cache    | Redis + BullMQ                               | 8.x / 5.x   | AOF + snapshot untuk persistensi       |
| Object Storage   | MinIO (lokal) / S3-compatible (produksi)     | -            | Bucket PRIVATE: `c1-uploads`           |
| Frontend         | HTML + CSS + Vanilla JS                      | -            | Mobile-first, Tailwind CSS via CDN     |
| PDF Generator    | PHP 8 + mPDF                                 | 8.x          | Microservice terpisah, port 8000       |
| Auth             | scrypt + HMAC stateless                      | built-in     | Password/PIN scrypt, token HMAC SHA256 |
| Process Manager  | PM2                                          | -            | Cluster mode di produksi               |

### 1.2 Package.json (Backend Node.js)

```json
{
  "name": "kawal-pemilu-api",
  "version": "0.1.0",
  "private": true,
  "type": "commonjs",
  "scripts": {
    "start:api": "node src/server.js",
    "start:worker": "node src/worker.js",
    "migrate": "node scripts/migrate.js",
    "seed": "node scripts/seed-dev.js",
    "test": "node --test \"test/**/*.test.js\"",
    "lint": "node --check src/server.js && node --check src/worker.js"
  },
  "dependencies": {
    "fastify": "^4.28.0",
    "mysql2": "^3.11.0",
    "ioredis": "^5.4.0",
    "bullmq": "^5.12.0",
    "@aws-sdk/client-s3": "^3.637.0",
    "@aws-sdk/s3-request-presigner": "^3.637.0",
    "undici": "^6.19.0",
    "zod": "^3.23.0"
  }
}
```

### 1.3 Package.json (PDF Service - PHP)

```json
{
  "require": {
    "mpdf/mpdf": "^8.2.0"
  }
}
```

---

## BAB 2: ATURAN ARSITEKTUR (WAJIB DIPATUHI)

### 2.1 ATURAN #1: JANGAN PERNAH KIRIM GAMBAR KE NODE.JS

```
YANG DILARANG:
  Browser → POST /api/v1/upload (multipart/form-data) → Node.js → MySQL ❌

YANG BENAR:
  Step 1: Browser → GET /api/v1/upload/presigned-url → Node.js → return { uploadUrl, objectKey }
  Step 2: Browser → HTTP PUT gambar langsung ke S3/MinIO (via uploadUrl)
  Step 3: Browser → POST /api/v1/saksi/c1 (JSON, berisi objectKey) → Node.js → Redis Queue
```

**Mengapa:** Node.js single-threaded. Jika menerima file gambar 5MB dari 1000 saksi serentak, RAM dan CPU habis untuk buffer → server crash.

**Implementasi:**
- Middleware global `reject-multipart.js` menolak semua request `content-type: multipart/form-data` → response `415 Unsupported Media Type`.
- Presigned URL: expiry 10 menit, key: `c1/{YYYY-MM-DD}/{uuid}.jpg`
- Setelah upload, saksi kirim JSON berisi `{ submission_id, tps_id, kategori_pemilihan_id, foto_object_key, data_suara }`.

### 2.2 ATURAN #2: JANGAN PERNAH INSERT KE MYSQL DARI API

```
YANG DILARANG:
  API Route → mysql.query('INSERT INTO ...') ❌

YANG BENAR:
  API Route → zod validate → BullMQ queue.add() → response 202 Accepted
  Worker → BullMQ queue.process() → mysql.transaction() → COMMIT
```

**Mengapa:** Jika 5000 saksi submit bersamaan, MySQL connection pool habis → lock table → timeout → crash.

**Implementasi:**
- Route hanya menerima JSON, validasi dengan Zod, lalu masukkan ke BullMQ queue.
- Worker berjalan terpisah, mengambil job dari queue, lalu insert ke MySQL dalam transaksi.
- Idempotensi: `jobId = submission_id (UUID)`. Submit ulang → `INSERT ... ON DUPLICATE KEY UPDATE`.

### 2.3 ATURAN #3: DASHBOARD BACA DARI REDIS CACHE

```
YANG DILARANG:
  GET /dashboard/rekap → SELECT SUM(...). GROUP BY ... dari MySQL → response ❌

YANG BENAR:
  GET /dashboard/rekap → redis.get('rekap:...') → jika ada, return JSON
                       → jika tidak ada, query MySQL → simpan ke Redis (TTL 60s) → return
```

---

## BAB 3: DATABASE SCHEMA LENGKAP

### 3.1 Master Data (SUDAH ADA di `db/db.sql`)

Berikut tabel master data yang sudah tersedia dan **TIDAK BOLEH dibuat ulang**:

#### Tabel `provinsi`
```sql
CREATE TABLE provinsi (
  id       INT NOT NULL AUTO_INCREMENT,
  nama     VARCHAR(64) NOT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
-- Data: id=18, nama='Lampung'
```

#### Tabel `kabupaten`
```sql
CREATE TABLE kabupaten (
  id          INT NOT NULL AUTO_INCREMENT,
  provinsi_id INT NOT NULL,
  nama        VARCHAR(64) NOT NULL,
  kode        VARCHAR(8) NOT NULL COMMENT 'kode Kemendagri',
  PRIMARY KEY (id),
  KEY idx_kab_prov (provinsi_id),
  FOREIGN KEY (provinsi_id) REFERENCES provinsi(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
-- 15 kabupaten/kota di Lampung
```

#### Tabel `kecamatan`
```sql
CREATE TABLE kecamatan (
  id            INT NOT NULL AUTO_INCREMENT,
  kabupaten_id  INT NOT NULL,
  nama          VARCHAR(64) NOT NULL,
  kode          VARCHAR(8) NOT NULL,
  PRIMARY KEY (id),
  KEY idx_kec_kab (kabupaten_id),
  FOREIGN KEY (kabupaten_id) REFERENCES kabupaten(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
-- 229 kecamatan
```

#### Tabel `desa`
```sql
CREATE TABLE desa (
  id            INT NOT NULL AUTO_INCREMENT,
  kecamatan_id  INT DEFAULT NULL,
  nama          VARCHAR(100) NOT NULL,
  kode          VARCHAR(10) DEFAULT NULL,
  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_desa_kec (kecamatan_id),
  FOREIGN KEY (kecamatan_id) REFERENCES kecamatan(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
-- 2.642 desa/kelurahan
```

#### Tabel `master_tps`
```sql
CREATE TABLE master_tps (
  id               VARCHAR(24) NOT NULL COMMENT 'Kode ternormalisasi, contoh: 180100439001',
  desa_id          INT NOT NULL,
  nama_wilayah     VARCHAR(255) NOT NULL COMMENT 'Denormalisasi: nama desa, kecamatan, kab',
  no_tps           INT UNSIGNED NOT NULL,
  jumlah_pemilih   INT UNSIGNED NOT NULL DEFAULT 0 CHECK (jumlah_pemilih <= 500),
  lat              DECIMAL(10,7) DEFAULT NULL,
  lng              DECIMAL(10,7) DEFAULT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_tps_desa_no (desa_id, no_tps),
  KEY idx_tps_desa (desa_id),
  FOREIGN KEY (desa_id) REFERENCES desa(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
-- ±27.440 TPS untuk seluruh Provinsi Lampung
```

#### Tabel `kategori_pemilihan`
```sql
CREATE TABLE kategori_pemilihan (
  id                 VARCHAR(32) NOT NULL,
  nama_pemilihan     VARCHAR(128) NOT NULL,
  tipe_pemilihan     ENUM('PILGUB','PILBUP','PILWAKOTA','DPR_RI','DPD','DPRD_PROV','DPRD_KABKOTA') NOT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
-- Contoh data:
-- '01' = PILEG DPRD Provinsi
-- '02' = PILEG DPR RI
-- '03' = PILEG DPRD Kabupaten
-- '04' = DPD
-- '05' = PILGUB
-- '06' = PILBUP/PILWAKOTA
```

#### Tabel `dapil`
```sql
CREATE TABLE dapil (
  id              VARCHAR(24) NOT NULL COMMENT 'Contoh: DPRRI-LPG-1, DPRDPROV-LPG-5, DPRDKAB-1871-1',
  tingkat         ENUM('DPR_RI','DPD','DPRD_PROV','DPRD_KABKOTA') NOT NULL,
  kode            VARCHAR(16) NOT NULL,
  nama            VARCHAR(128) NOT NULL COMMENT 'Contoh: Dapil Lampung I (DPR RI)',
  provinsi_id     INT NOT NULL DEFAULT 18,
  kabupaten_id    INT DEFAULT NULL COMMENT 'Diisi khusus DPRD_KABKOTA',
  jml_kursi       SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (id),
  KEY idx_dapil_tingkat (tingkat),
  KEY idx_dapil_kab (kabupaten_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
-- 95 dapil: DPR RI (2), DPD (1), DPRD Prov (8), DPRD Kab/Kota (84)
```

#### Tabel `dapil_kabupaten` (relasi)
```sql
CREATE TABLE dapil_kabupaten (
  dapil_id      VARCHAR(24) NOT NULL,
  kabupaten_id  INT NOT NULL,
  PRIMARY KEY (dapil_id, kabupaten_id),
  KEY idx_dk_kab (kabupaten_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Tabel `dapil_kecamatan` (relasi)
```sql
CREATE TABLE dapil_kecamatan (
  dapil_id      VARCHAR(24) NOT NULL,
  kecamatan_id  INT NOT NULL,
  PRIMARY KEY (dapil_id, kecamatan_id),
  KEY idx_dc_kec (kecamatan_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Tabel `master_kandidat`
```sql
CREATE TABLE master_kandidat (
  id                      VARCHAR(64) NOT NULL,
  kategori_pemilihan_id   VARCHAR(32) NOT NULL,
  dapil_id                VARCHAR(24) DEFAULT NULL,
  tipe_kandidat           ENUM('PASLON','PARTAI','CALEG','DPD') NOT NULL,
  nama                    VARCHAR(255) NOT NULL,
  foto                    VARCHAR(512) DEFAULT NULL COMMENT 'URL foto kandidat',
  partai                  VARCHAR(128) DEFAULT NULL COMMENT 'Nama partai (untuk caleg)',
  no_urut                 INT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (id),
  KEY idx_kad_kategori (kategori_pemilihan_id),
  KEY idx_kad_dapil (dapil_id),
  FOREIGN KEY (kategori_pemilihan_id) REFERENCES kategori_pemilihan(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Tabel `admin_users`
```sql
CREATE TABLE admin_users (
  id              INT UNSIGNED AUTO_INCREMENT,
  username        VARCHAR(64) NOT NULL,
  password_hash   VARCHAR(256) NOT NULL COMMENT 'format: scrypt$salt_hex$hash_hex',
  created_at      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uq_admin_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Tabel `admin_sessions`
```sql
CREATE TABLE admin_sessions (
  token       VARCHAR(64) NOT NULL COMMENT 'hex 32 byte (sha256 dari token asli)',
  user_id     INT UNSIGNED NOT NULL,
  expires_at  DATETIME NOT NULL,
  created_at  DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (token),
  KEY idx_session_user (user_id),
  KEY idx_session_exp (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.2 Tabel Operasional (DIBUAT dari `001_operational.sql`)

#### Tabel `hierarki_pengurus`
```sql
CREATE TABLE hierarki_pengurus (
  id            INT UNSIGNED AUTO_INCREMENT,
  nama          VARCHAR(128) NOT NULL,
  no_hp         VARCHAR(16) NOT NULL,
  password_hash VARCHAR(512) NOT NULL COMMENT 'scrypt$salt_hex$hash_hex',
  tingkat       ENUM('DPW','DPC','PAC','RANTING','ANAK_RANTING') NOT NULL,
  provinsi_id   INT UNSIGNED NULL,
  kabupaten_id  INT UNSIGNED NULL,
  kecamatan_id  INT UNSIGNED NULL,
  desa_id       INT UNSIGNED NULL,
  tps_id        VARCHAR(24) NULL,
  diadmin_oleh  INT UNSIGNED NULL,
  created_at    DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uq_pengurus_nohp (no_hp)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Aturan Hierarki (siapa bisa melihat data siapa):**
```
DPW     → melihat SEMUA data provinsi Lampung
DPC     → melihat data KABUPATEN/KOTA miliknya saja
PAC     → melihat data KECAMATAN miliknya saja
RANTING → melihat data DESA miliknya saja
ANAK_RANTING → melihat data TPS miliknya saja
```

**Query filter visibilitas:**
```sql
-- Contoh untuk DPC (kabupaten_id = 1871):
SELECT * FROM saksi WHERE tps_id IN (
  SELECT id FROM master_tps WHERE desa_id IN (
    SELECT id FROM desa WHERE kecamatan_id IN (
      SELECT id FROM kecamatan WHERE kabupaten_id = 1871
    )
  )
);
```

#### Tabel `saksi`
```sql
CREATE TABLE saksi (
  id                 INT UNSIGNED AUTO_INCREMENT,
  nama               VARCHAR(128) NOT NULL,
  no_hp              VARCHAR(16) NOT NULL,
  pin_hash           VARCHAR(512) NOT NULL COMMENT 'scrypt$salt_hex$hash_hex',
  tps_id             VARCHAR(24) NOT NULL,
  status_hadir       ENUM('BELUM_HADIR','HADIR') NOT NULL DEFAULT 'BELUM_HADIR',
  didaftarkan_oleh   INT UNSIGNED NULL COMMENT 'ref hierarki_pengurus.id',
  created_at         DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uq_saksi_nohp (no_hp),
  KEY idx_saksi_tps (tps_id),
  CONSTRAINT fk_saksi_tps FOREIGN KEY (tps_id) REFERENCES master_tps(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Tabel `bukti_c1`
```sql
CREATE TABLE bukti_c1 (
  id                    INT UNSIGNED AUTO_INCREMENT,
  submission_id         CHAR(36) NOT NULL COMMENT 'UUID v4 dari klien (idempotensi)',
  saksi_id              INT UNSIGNED NOT NULL,
  tps_id                VARCHAR(24) NOT NULL,
  kategori_pemilihan_id VARCHAR(32) NOT NULL,
  s3_object_key         VARCHAR(512) NOT NULL,
  s3_image_url          VARCHAR(512) NOT NULL,
  status                ENUM('PENDING','VALID','INVALID','DOUBLE') NOT NULL DEFAULT 'PENDING',
  catatan_verifikasi    VARCHAR(512) NULL,
  diverifikasi_oleh     INT UNSIGNED NULL,
  diverifikasi_at       DATETIME(3) NULL,
  created_at            DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uq_bukti_submission (submission_id),
  KEY idx_bukti_tps (tps_id, kategori_pemilihan_id),
  KEY idx_bukti_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Tabel `suara_c1`
```sql
CREATE TABLE suara_c1 (
  id                    INT UNSIGNED AUTO_INCREMENT,
  bukti_id              INT UNSIGNED NOT NULL COMMENT 'Dokumen terakhir yang jadi sumber',
  tps_id                VARCHAR(24) NOT NULL,
  kategori_pemilihan_id VARCHAR(32) NOT NULL,
  kandidat_id           VARCHAR(64) NOT NULL,
  jumlah_suara          INT UNSIGNED NOT NULL DEFAULT 0,
  created_at            DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uq_suara (tps_id, kategori_pemilihan_id, kandidat_id),
  KEY idx_suara_kandidat (kandidat_id),
  CONSTRAINT fk_suara_kandidat FOREIGN KEY (kandidat_id) REFERENCES master_kandidat(id),
  CONSTRAINT fk_suara_bukti FOREIGN KEY (bukti_id) REFERENCES bukti_c1(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**UNIQUE KEY `(tps_id, kategori_pemilihan_id, kandidat_id)`** menjamin satu angka suara per kandidat per TPS. Submit ulang/koreksi → `ON DUPLICATE KEY UPDATE`.

#### Tabel `keuangan_logistik`
```sql
CREATE TABLE keuangan_logistik (
  id               INT UNSIGNED AUTO_INCREMENT,
  kategori         ENUM('LOGISTIK','DANA_SAKSI') NOT NULL,
  deskripsi        VARCHAR(255) NOT NULL,
  wilayah_sasaran  VARCHAR(255) NULL COMMENT 'Teks/id wilayah target',
  nominal          DECIMAL(15,2) NOT NULL DEFAULT 0,
  tanggal_kegiatan DATE NOT NULL,
  lampiran_s3_key  VARCHAR(512) NULL,
  dicatat_oleh     INT UNSIGNED NULL,
  created_at       DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  KEY idx_keuangan (kategori, tanggal_kegiatan)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Tabel `audit_log`
```sql
CREATE TABLE audit_log (
  id         BIGINT UNSIGNED AUTO_INCREMENT,
  actor_type ENUM('ADMIN','PENGURUS','SAKSI','SYSTEM') NOT NULL,
  actor_id   INT UNSIGNED NULL,
  aksi       VARCHAR(64) NOT NULL COMMENT 'LOGIN, SUBMIT_C1, VERIFIKASI, CETAK_PDF, ...',
  ref_id     VARCHAR(64) NULL,
  detail     JSON NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  KEY idx_audit_actor (actor_type, actor_id),
  KEY idx_audit_aksi (aksi, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.3 View Agregat (Dashboard)

```sql
CREATE OR REPLACE VIEW v_rekap_dapil AS
SELECT
  s.tps_id,
  s.kandidat_id,
  s.jumlah_suara,
  d.id AS dapil_id,
  d.tingkat
FROM suara_c1 s
JOIN master_tps t      ON t.id = s.tps_id
JOIN master_kandidat k ON k.id = s.kandidat_id
LEFT JOIN dapil d      ON d.id = k.dapil_id;
```

Dashboard membaca dari Redis cache, bukan langsung dari view ini per request.

---

## BAB 4: STRUKTUR FOLDER & FILE

```
kawal-pemilu/
├── db/
│   ├── db.sql                          # Master data (sudah ada, JANGAN edit)
│   └── migrations/
│       └── 001_operational.sql         # Tabel operasional (B.1-B.6 + view)
│
├── api/                                # Backend Node.js
│   ├── .env                            # Environment variables (TIDAK di git)
│   ├── .env.example                    # Template env
│   ├── package.json                    # Dependencies
│   ├── src/
│   │   ├── server.js                   # Entry point API (npm run start:api)
│   │   ├── worker.js                   # Entry point Worker (npm run start:worker)
│   │   ├── config.js                   # Baca .env, export config object
│   │   │
│   │   ├── lib/
│   │   │   ├── load-env.js             # Parser .env manual (tanpa dotenv)
│   │   │   └── crypto.js               # scryptHash, scryptVerify, newToken, sha256Hex
│   │   │
│   │   ├── middleware/
│   │   │   ├── reject-multipart.js     # Tolak multipart/form-data (415)
│   │   │   └── auth.js                 # requireAdmin, requireSaksi, signSaksi, verifySaksi
│   │   │
│   │   ├── routes/
│   │   │   ├── auth.js                 # POST /auth/admin/login, /auth/saksi/login
│   │   │   ├── upload.js              # GET /upload/presigned-url
│   │   │   ├── saksi.js               # POST /saksi/c1, GET /saksi/c1/:tps_id/:kategori
│   │   │   ├── dashboard.js           # GET /dashboard/rekap, /dashboard/sirekap/:dapil_id, /healthz
│   │   │   └── admin.js               # GET/PUT /admin/c1-pending, /admin/c1-verifikasi, /admin/export/pdf
│   │   │
│   │   └── services/
│   │       ├── mysql.js                # Pool + query() + transaction()
│   │       ├── redis.js                # ioredis shared + ping()
│   │       ├── s3.js                   # AWS SDK v3 S3Client, presigned URL
│   │       ├── queue.js                # BullMQ Queue init
│   │       ├── rekap.js                # Agregasi rekap + Redis cache
│   │       ├── scraper.js              # Sirekap KPU scraper + scheduler
│   │       └── pdf.js                  # Klien microservice PDF
│   │
│   ├── scripts/
│   │   ├── migrate.js                  # Import db.sql + 001_operational.sql
│   │   ├── seed-dev.js                 # Seed admin + saksi dev
│   │   └── dlq-replay.js              # Replay job DLQ
│   │
│   └── test/
│       ├── crypto.test.js
│       ├── rekap.test.js
│       ├── reject-multipart.test.js
│       ├── worker.mapping.test.js
│       └── teardown.js
│
├── pdf-service/                        # Microservice PHP
│   ├── generate.php                    # POST JSON → PDF
│   ├── composer.json
│   └── vendor/
│
├── frontend/                           # Frontend statis
│   ├── index.html                      # SPA shell (login + dashboard)
│   ├── admin.html                      # Dashboard admin
│   ├── saksi.html                      # Dashboard saksi
│   ├── assets/
│   │   ├── css/
│   │   │   └── app.css                 # Custom CSS (Tailwind via CDN)
│   │   ├── js/
│   │   │   ├── app.js                  # Shared: API client, auth, router
│   │   │   ├── admin.js                # Admin dashboard logic
│   │   │   ├── saksi.js                # Saksi dashboard logic
│   │   │   └── charts.js              # Chart.js wrapper untuk rekap
│   │   └── img/
│   │       └── logo.png
│   └── favicon.ico
│
└── deploy/                             # Konfigurasi produksi
    ├── nginx/
    │   ├── kawal-pemilu.conf.j2
    │   └── security-headers.conf
    ├── mysql/
    │   └── mysqld.cnf
    ├── redis/
    │   └── redis.conf
    ├── pm2/
    │   ├── api.ecosystem.config.js
    │   └── worker.ecosystem.config.js
    └── scripts/
        ├── bootstrap-server.sh
        ├── setup-lb.sh
        ├── setup-api-node.sh
        ├── setup-worker-node.sh
        ├── setup-db-node.sh
        └── backup-mysql.sh
```

---

## BAB 5: VARIABEL LINGKUNGAN (.env)

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

# === OBJECT STORAGE (MinIO lokal / NEO Object Storage / AWS S3) ===
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

## BAB 6: BACKEND NODE.JS - SETIAP FILE

### 6.1 `src/config.js` - Konfigurasi

```javascript
'use strict';
require('./lib/load-env');

const cleanEnv = (v, fallback) => (v === undefined || v === '' ? fallback : v);
const intEnv = (v, fallback) => {
  const n = Number.parseInt(v, 10);
  return Number.isFinite(n) ? n : fallback;
};

module.exports = {
  NODE_ENV: cleanEnv(process.env.NODE_ENV, 'development'),
  PORT: intEnv(process.env.PORT, 3000),
  DB_HOST: cleanEnv(process.env.DB_HOST, '127.0.0.1'),
  DB_PORT: intEnv(process.env.DB_PORT, 3306),
  DB_USER: cleanEnv(process.env.DB_USER, 'root'),
  DB_PASS: cleanEnv(process.env.DB_PASS, ''),
  DB_NAME: cleanEnv(process.env.DB_NAME, 'kawal_pemilu'),
  DB_POOL: intEnv(process.env.DB_POOL, 10),
  REDIS_HOST: cleanEnv(process.env.REDIS_HOST, '127.0.0.1'),
  REDIS_PORT: intEnv(process.env.REDIS_PORT, 6379),
  S3_ENDPOINT: cleanEnv(process.env.S3_ENDPOINT, 'http://127.0.0.1:9000'),
  S3_REGION: cleanEnv(process.env.S3_REGION, 'us-east-1'),
  S3_BUCKET: cleanEnv(process.env.S3_BUCKET, 'c1-uploads'),
  S3_ACCESS_KEY: cleanEnv(process.env.S3_ACCESS_KEY, 'minioadmin'),
  S3_SECRET_KEY: cleanEnv(process.env.S3_SECRET_KEY, 'minioadmin'),
  PRESIGNED_URL_EXPIRY: intEnv(process.env.PRESIGNED_URL_EXPIRY, 600),
  MAX_UPLOAD_SIZE_MB: intEnv(process.env.MAX_UPLOAD_SIZE_MB, 10),
  PDF_SERVICE_URL: cleanEnv(process.env.PDF_SERVICE_URL, 'http://127.0.0.1:8000'),
  PDF_SERVICE_TOKEN: cleanEnv(process.env.PDF_SERVICE_TOKEN, 'change-me'),
  APP_SECRET: cleanEnv(process.env.APP_SECRET, 'dev-only-secret'),
  SESSION_TTL_HOURS: intEnv(process.env.SESSION_TTL_HOURS, 24),
  LOGIN_MAX_ATTEMPTS: intEnv(process.env.LOGIN_MAX_ATTEMPTS, 5),
  REKAP_TTL_SECONDS: intEnv(process.env.REKAP_TTL_SECONDS, 60),
  SIRREKAP_INTERVAL_MIN: intEnv(process.env.SIRREKAP_INTERVAL_MIN, 30),
};
```

### 6.2 `src/lib/load-env.js` - Parser .env

```javascript
'use strict';
const fs = require('fs');
const path = require('path');

function loadEnvFile(file) {
  if (!fs.existsSync(file)) return;
  const content = fs.readFileSync(file, 'utf8');
  for (const rawLine of content.split(/\r?\n/)) {
    const line = rawLine.trim();
    if (!line || line.startsWith('#')) continue;
    const eq = line.indexOf('=');
    if (eq === -1) continue;
    const key = line.slice(0, eq).trim();
    let value = line.slice(eq + 1).trim();
    if ((value.startsWith('"') && value.endsWith('"')) ||
        (value.startsWith("'") && value.endsWith("'"))) {
      value = value.slice(1, -1);
    }
    if (!(key in process.env)) process.env[key] = value;
  }
}

loadEnvFile(path.resolve(__dirname, '../../.env'));
```

### 6.3 `src/lib/crypto.js` - Password & Token

```javascript
'use strict';
const crypto = require('crypto');

const KEYLEN = 64;

// Hash password/PIN: scrypt$salt_hex$hash_hex
function scryptHash(plain) {
  const salt = crypto.randomBytes(16);
  const hash = crypto.scryptSync(String(plain), salt, KEYLEN);
  return `scrypt$${salt.toString('hex')}$${hash.toString('hex')}`;
}

// Verifikasi password/PIN
function scryptVerify(plain, stored) {
  if (typeof stored !== 'string') return false;
  const parts = stored.split('$');
  if (parts.length !== 3 || parts[0] !== 'scrypt') return false;
  const [, saltHex, hashHex] = parts;
  try {
    const salt = Buffer.from(saltHex, 'hex');
    const expected = Buffer.from(hashHex, 'hex');
    const actual = crypto.scryptSync(String(plain), salt, expected.length);
    return actual.length === expected.length && crypto.timingSafeEqual(actual, expected);
  } catch { return false; }
}

// Generate token acak 64 hex
function newToken() {
  return crypto.randomBytes(32).toString('hex');
}

// SHA256 hex
function sha256Hex(value) {
  return crypto.createHash('sha256').update(String(value)).digest('hex');
}

module.exports = { scryptHash, scryptVerify, newToken, sha256Hex };
```

### 6.4 `src/services/mysql.js` - Koneksi Database

```javascript
'use strict';
const mysql = require('mysql2/promise');
const config = require('../config');

const pool = mysql.createPool({
  host: config.DB_HOST,
  port: config.DB_PORT,
  user: config.DB_USER,
  password: config.DB_PASS,
  database: config.DB_NAME,
  waitForConnections: true,
  connectionLimit: config.DB_POOL,
  queueLimit: 0,
  namedPlaceholders: true,
  charset: 'utf8mb4_unicode_ci',
  dateStrings: false,
});

async function query(sql, params = []) {
  const [rows] = await pool.execute(sql, params);
  return rows;
}

async function ping() {
  const rows = await query('SELECT 1 AS ok');
  return rows.length === 1;
}

async function transaction(fn) {
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();
    const result = await fn(conn);
    await conn.commit();
    return result;
  } catch (err) {
    try { await conn.rollback(); } catch {}
    throw err;
  } finally {
    conn.release();
  }
}

module.exports = { pool, query, ping, transaction };
```

### 6.5 `src/services/redis.js` - Koneksi Redis

```javascript
'use strict';
const Redis = require('ioredis');
const config = require('../config');

const redis = new Redis({
  host: config.REDIS_HOST,
  port: config.REDIS_PORT,
  maxRetriesPerRequest: 2,
  lazyConnect: false,
});

redis.on('error', (err) => {
  console.error('[redis] error:', err.message);
});

async function ping() {
  return (await redis.ping()) === 'PONG';
}

module.exports = { redis, ping };
```

### 6.6 `src/services/s3.js` - Presigned URL

```javascript
'use strict';
const crypto = require('crypto');
const { S3Client, PutObjectCommand, GetObjectCommand, HeadObjectCommand } = require('@aws-sdk/client-s3');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');
const config = require('../config');

const client = new S3Client({
  endpoint: config.S3_ENDPOINT,
  region: config.S3_REGION,
  forcePathStyle: true,
  credentials: {
    accessKeyId: config.S3_ACCESS_KEY,
    secretAccessKey: config.S3_SECRET_KEY,
  },
});

// Key: c1/{YYYY-MM-DD}/{uuid}.jpg
function buildObjectKey(ext = 'jpg') {
  const d = new Date();
  const date = `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
  return `c1/${date}/${crypto.randomUUID()}.${ext}`;
}

function extFromType(fileType) {
  const t = String(fileType || '').toLowerCase();
  if (t.includes('png')) return 'png';
  return 'jpg';
}

async function createPresignedUpload({ contentType }) {
  const key = buildObjectKey(extFromType(contentType));
  const maxBytes = config.MAX_UPLOAD_SIZE_MB * 1024 * 1024;
  const cmd = new PutObjectCommand({
    Bucket: config.S3_BUCKET,
    Key: key,
    ContentType: contentType,
  });
  const uploadUrl = await getSignedUrl(client, cmd, { expiresIn: config.PRESIGNED_URL_EXPIRY });
  return {
    uploadUrl,
    objectKey: key,
    expiresIn: config.PRESIGNED_URL_EXPIRY,
    maxBytes,
    headers: { 'Content-Type': contentType },
  };
}

async function createPresignedDownload(objectKey, expiresIn = 900) {
  const cmd = new GetObjectCommand({ Bucket: config.S3_BUCKET, Key: objectKey });
  return getSignedUrl(client, cmd, { expiresIn });
}

async function verifyUploadedObject(objectKey) {
  const maxBytes = config.MAX_UPLOAD_SIZE_MB * 1024 * 1024;
  const cmd = new HeadObjectCommand({ Bucket: config.S3_BUCKET, Key: objectKey });
  let out;
  try {
    out = await client.send(cmd);
  } catch (err) {
    const status = err.$metadata && err.$metadata.httpStatusCode;
    throw new Error(`object tidak ditemukan di S3: key=${objectKey} (${status || err.message})`);
  }
  const size = Number(out.ContentLength || 0);
  if (size <= 0 || size > maxBytes) {
    throw new Error(`ukuran file ${size} byte di luar batas (maks ${maxBytes})`);
  }
  return { contentLength: size, contentType: out.ContentType };
}

module.exports = { client, buildObjectKey, createPresignedUpload, createPresignedDownload, verifyUploadedObject };
```

### 6.7 `src/services/queue.js` - BullMQ Queue

```javascript
'use strict';
const { Queue } = require('bullmq');
const config = require('../config');

const QUEUE_NAME = 'c1-processing-queue';
const JOB_NAME = 'process-c1';
const QUEUE_OPTS = {
  attempts: 5,
  backoff: { type: 'exponential', delay: 60_000 },
  removeOnComplete: { count: 1000 },
  removeOnFail: false,
};

let queue = null;

function connection() {
  return new (require('ioredis'))({
    host: config.REDIS_HOST,
    port: config.REDIS_PORT,
    maxRetriesPerRequest: null,
    enableReadyCheck: true,
  });
}

function getQueue() {
  if (!queue) {
    queue = new Queue(QUEUE_NAME, {
      connection: connection(),
      defaultJobOptions: QUEUE_OPTS,
    });
  }
  return queue;
}

async function enqueueC1(payload) {
  const q = getQueue();
  await q.add(JOB_NAME, payload, { ...QUEUE_OPTS, jobId: payload.submission_id });
}

async function close() {
  if (queue) await queue.close();
}

module.exports = { QUEUE_NAME, JOB_NAME, QUEUE_OPTS, connection, getQueue, enqueueC1, close };
```

### 6.8 `src/services/rekap.js` - Cache Agregasi

```javascript
'use strict';
const { query } = require('./mysql');
const { redis } = require('./redis');
const config = require('../config');

function cacheKey({ tingkat, dapilId }) {
  return `rekap:${tingkat || 'all'}:${dapilId || 'all'}`;
}

async function getRekap({ tingkat, dapilId } = {}) {
  const key = cacheKey({ tingkat, dapilId });
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const rows = await query(
    `SELECT s.tingkat, s.dapil_id, s.kandidat_id,
            k.nama AS nama_kandidat, k.partai, k.no_urut,
            SUM(s.jumlah_suara) AS total_suara,
            COUNT(DISTINCT s.tps_id) AS tps_masuk
     FROM v_rekap_dapil s
     JOIN master_kandidat k ON k.id = s.kandidat_id
     WHERE (? IS NULL OR s.tingkat = ?)
       AND (? IS NULL OR s.dapil_id = ?)
     GROUP BY s.tingkat, s.dapil_id, s.kandidat_id, k.nama, k.partai, k.no_urut
     ORDER BY s.tingkat, s.dapil_id, total_suara DESC`,
    [tingkat, tingkat, dapilId, dapilId]
  );

  const payload = { generated_at: new Date().toISOString(), tingkat, dapil_id: dapilId, rows };
  await redis.set(key, JSON.stringify(payload), 'EX', config.REKAP_TTL_SECONDS);
  return payload;
}

module.exports = { getRekap, cacheKey };
```

### 6.9 `src/services/scraper.js` - Sirekap KPU

```javascript
'use strict';
const { request } = require('undici');
const config = require('../config');
const { redis } = require('./redis');

const SIREKAP_KEY = (dapilId) => `sirekap:${dapilId}`;
const TTL_SIREKAP = 600;
const MAX_REQ_PER_MIN = 10;

let timer = null;

async function fetchDapil(dapilId) {
  const url = `https://sirekap.kpu.go.id/api/rekap/${encodeURIComponent(dapilId)}`;
  const res = await request(url, {
    method: 'GET',
    headers: { 'user-agent': 'kawal-pemilu-lampung/0.1 (kontak: admin@example.com)' },
  });
  return res.body.json();
}

async function scrapeOnce() {
  let dapils = [];
  try {
    dapils = await require('./mysql').query('SELECT id FROM dapil ORDER BY id');
  } catch (err) {
    console.error('[sirekap] tidak bisa membaca daftar dapil:', err.message);
    return;
  }
  const gapMs = Math.max(60_000, Math.ceil(60_000 / MAX_REQ_PER_MIN));
  for (const row of dapils) {
    try {
      const json = await fetchDapil(row.id);
      await redis.set(SIREKAP_KEY(row.id), JSON.stringify(json), 'EX', TTL_SIREKAP);
    } catch (err) {
      console.error(`[sirekap] gagal dapil ${row.id}:`, err.message);
    }
    await new Promise((r) => setTimeout(r, gapMs));
  }
}

function start() {
  const intervalMs = config.SIRREKAP_INTERVAL_MIN * 60_000;
  timer = setInterval(() => {
    scrapeOnce().catch((e) => console.error('[sirekap]', e.message));
  }, intervalMs);
}

function stop() { if (timer) clearInterval(timer); }

module.exports = { start, stop, scrapeOnce, SIREKAP_KEY };
```

### 6.10 `src/services/pdf.js` - Klien PDF Service

```javascript
'use strict';
const { request } = require('undici');
const config = require('../config');

async function generatePdf(jenis, data) {
  const res = await request(config.PDF_SERVICE_URL, {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      'x-service-token': config.PDF_SERVICE_TOKEN,
    },
    body: JSON.stringify({ jenis, data }),
  });
  if (res.statusCode >= 400) {
    const text = await res.body.text();
    throw new Error(`pdf-service ${res.statusCode}: ${text.slice(0, 200)}`);
  }
  return Buffer.from(await res.body.arrayBuffer());
}

module.exports = { generatePdf };
```

### 6.11 `src/middleware/reject-multipart.js`

```javascript
'use strict';
async function rejectMultipartHook(request, reply) {
  const ct = String(request.headers['content-type'] || '').toLowerCase();
  if (ct.startsWith('multipart/form-data')) {
    return reply.code(415).send({
      error: 'unsupported_media_type',
      message: 'Upload gambar hanya via presigned URL langsung ke S3. Kirim JSON.',
    });
  }
}
module.exports = rejectMultipartHook;
```

### 6.12 `src/middleware/auth.js` - Autentikasi

```javascript
'use strict';
const crypto = require('crypto');
const { query } = require('../services/mysql');
const { sha256Hex } = require('../lib/crypto');
const config = require('../config');

// === ADMIN: Bearer token di admin_sessions ===
async function requireAdmin(request, reply) {
  const auth = String(request.headers.authorization || '');
  const token = auth.startsWith('Bearer ') ? auth.slice(7).trim() : '';
  if (!token) return reply.code(401).send({ error: 'unauthorized' });

  const rows = await query(
    `SELECT s.id, s.user_id, s.expires_at, u.username
     FROM admin_sessions s
     JOIN admin_users u ON u.id = s.user_id
     WHERE s.token = ? LIMIT 1`,
    [sha256Hex(token)]
  );
  const session = rows[0];
  if (!session) return reply.code(401).send({ error: 'unauthorized' });
  if (new Date(session.expires_at).getTime() < Date.now()) {
    await query('DELETE FROM admin_sessions WHERE id = ?', [session.id]);
    return reply.code(401).send({ error: 'session_expired' });
  }
  request.admin = { id: session.user_id, username: session.username };
}

// === SAKSI: Bearer token HMAC stateless ===
function signSaksi(payload) {
  const body = Buffer.from(JSON.stringify(payload)).toString('base64url');
  const sig = crypto.createHmac('sha256', config.APP_SECRET).update(body).digest('base64url');
  return `${body}.${sig}`;
}

function verifySaksi(token) {
  if (typeof token !== 'string') return null;
  const idx = token.lastIndexOf('.');
  if (idx < 0) return null;
  const body = token.slice(0, idx);
  const sig = token.slice(idx + 1);
  const expected = crypto.createHmac('sha256', config.APP_SECRET).update(body).digest('base64url');
  const a = Buffer.from(sig);
  const b = Buffer.from(expected);
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) return null;
  try {
    const payload = JSON.parse(Buffer.from(body, 'base64url').toString());
    if (typeof payload.exp !== 'number' || payload.exp < Date.now()) return null;
    return payload;
  } catch { return null; }
}

async function requireSaksi(request, reply) {
  const auth = String(request.headers.authorization || '');
  const token = auth.startsWith('Bearer ') ? auth.slice(7).trim() : '';
  const payload = verifySaksi(token);
  if (!payload || payload.role !== 'saksi') {
    return reply.code(401).send({ error: 'unauthorized' });
  }
  request.saksi = payload;
}

module.exports = { requireAdmin, requireSaksi, signSaksi, verifySaksi };
```

### 6.13 `src/server.js` - Entry Point API

```javascript
'use strict';
const Fastify = require('fastify');
const config = require('./config');
const rejectMultipart = require('./middleware/reject-multipart');
const { requireAdmin, requireSaksi } = require('./middleware/auth');
const { redis } = require('./services/redis');

async function buildServer(opts = {}) {
  const app = Fastify({
    logger: { level: config.NODE_ENV === 'development' ? 'debug' : 'info' },
    trustProxy: true,
    ...opts,
  });

  // CORS
  const allowedOrigins = (process.env.CORS_ORIGINS || '').split(',').map(s => s.trim()).filter(Boolean);
  app.addHook('onRequest', async (request, reply) => {
    const origin = request.headers.origin;
    if (!origin) return;
    const allowed = config.NODE_ENV === 'development' ||
      (allowedOrigins.length > 0 && allowedOrigins.includes(origin));
    if (allowed) {
      reply.header('access-control-allow-origin', origin);
      reply.header('access-control-allow-headers', 'authorization,content-type');
      reply.header('access-control-allow-methods', 'GET,POST,PUT,DELETE,OPTIONS');
      reply.header('vary', 'Origin');
    }
    if (request.method === 'OPTIONS') return reply.code(204).send();
  });

  // Tolak multipart global
  app.addHook('onRequest', rejectMultipart);

  // Decorator auth
  app.decorate('authAdmin', requireAdmin);
  app.decorate('authSaksi', requireSaksi);

  // Register routes
  await app.register(async function apiV1(fastify) {
    for (const route of [
      require('./routes/auth'),
      require('./routes/upload'),
      require('./routes/saksi'),
      require('./routes/dashboard'),
      require('./routes/admin'),
    ]) {
      await fastify.register(route);
    }
  }, { prefix: '/api/v1' });

  return app;
}

async function main() {
  const app = await buildServer();
  try {
    await app.listen({ port: config.PORT, host: '0.0.0.0' });
    console.log(`API siap di http://localhost:${config.PORT} (prefix /api/v1)`);
  } catch (err) {
    app.log.error(err);
    process.exitCode = 1;
  }
  const shutdown = async () => {
    await app.close();
    await redis.quit().catch(() => {});
    process.exit(0);
  };
  process.on('SIGINT', shutdown);
  process.on('SIGTERM', shutdown);
}

if (require.main === module) main();
module.exports = { buildServer };
```

### 6.14 `src/worker.js` - BullMQ Worker

```javascript
'use strict';
const { Worker, Queue } = require('bullmq');
const { connection, QUEUE_NAME, JOB_NAME } = require('./services/queue');
const { processJob } = require('./services/c1-processor');
const { pool } = require('./services/mysql');

const CONCURRENCY = Number(process.env.WORKER_CONCURRENCY || 10);
const DLQ_NAME = 'c1-processing-dead';

const worker = new Worker(QUEUE_NAME, processJob, {
  connection: connection(),
  concurrency: CONCURRENCY,
});

const dlq = new Queue(DLQ_NAME, { connection: connection() });

worker.on('failed', async (job, err) => {
  console.error(`[worker] job ${job.id} gagal (attempt ${job.attemptsMade}/${job.opts.attempts}):`, err.message);
  if (job.attemptsMade >= job.opts.attempts && !job.data.__dead) {
    try {
      await dlq.add(JOB_NAME,
        { __dead: true, original: job.data, error: String(err.message) },
        { jobId: `dead:${job.id}`, removeOnComplete: 200, removeOnFail: true }
      );
      console.warn(`[worker] job ${job.id} → DLQ`);
    } catch (e) {
      console.error('[worker] gagal memindah ke DLQ:', e.message);
    }
  }
});

worker.on('completed', (job) => {
  console.log(`[worker] job ${job.id} sukses (submission ${job.data.submission_id})`);
});

async function shutdown() {
  console.log('[worker] shutdown...');
  await worker.close();
  await dlq.close();
  await pool.end();
  process.exit(0);
}
process.on('SIGINT', shutdown);
process.on('SIGTERM', shutdown);
```

### 6.15 `src/services/c1-processor.js` - Job Processor

```javascript
'use strict';
const { transaction } = require('./mysql');
const { verifyUploadedObject } = require('./s3');

async function processJob(job) {
  const { submission_id, saksi_id, tps_id, kategori_pemilihan_id, foto_object_key, data_suara } = job.data;

  await transaction(async (conn) => {
    // 1) Verifikasi foto ada di S3
    await verifyUploadedObject(foto_object_key);

    // 2) Insert/upsert bukti_c1 (idempotent via submission_id)
    const [res] = await conn.execute(
      `INSERT INTO bukti_c1 (submission_id, saksi_id, tps_id, kategori_pemilihan_id, s3_object_key, s3_image_url)
       VALUES (?, ?, ?, ?, ?, ?)
       ON DUPLICATE KEY UPDATE
         saksi_id = VALUES(saksi_id),
         tps_id = VALUES(tps_id),
         kategori_pemilihan_id = VALUES(kategori_pemilihan_id),
         s3_object_key = VALUES(s3_object_key),
         s3_image_url = VALUES(s3_image_url),
         status = 'PENDING'`,
      [submission_id, saksi_id, tps_id, kategori_pemilihan_id, foto_object_key, `s3://${foto_object_key}`]
    );

    // Ambil bukti id
    let buktiId = res.insertId;
    if (!buktiId) {
      const [rows] = await conn.execute('SELECT id FROM bukti_c1 WHERE submission_id = ? LIMIT 1', [submission_id]);
      buktiId = rows[0].id;
    }

    // 3) Upsert suara_c1 per kandidat
    for (const suara of data_suara) {
      await conn.execute(
        `INSERT INTO suara_c1 (bukti_id, tps_id, kategori_pemilihan_id, kandidat_id, jumlah_suara)
         VALUES (?, ?, ?, ?, ?)
         ON DUPLICATE KEY UPDATE
           jumlah_suara = VALUES(jumlah_suara),
           bukti_id = VALUES(bukti_id)`,
        [buktiId, tps_id, kategori_pemilihan_id, suara.kandidat_id, suara.jumlah_suara]
      );
    }

    // 4) Audit log
    await conn.execute(
      `INSERT INTO audit_log (actor_type, actor_id, aksi, ref_id, detail)
       VALUES ('SAKSI', ?, 'SUBMIT_C1', ?, ?)`,
      [saksi_id, submission_id, JSON.stringify({ tps_id, kategori_pemilihan_id })]
    );
  });

  return { ok: true, submission_id };
}

module.exports = { processJob };
```

---

## BAB 7: ROUTE API - SETIAP ENDPOINT

### 7.1 `routes/auth.js` - Login Admin & Saksi

```javascript
'use strict';
const { z } = require('zod');
const config = require('../config');
const { query } = require('../services/mysql');
const { scryptVerify, newToken, sha256Hex } = require('../lib/crypto');
const { signSaksi } = require('../middleware/auth');

// === POST /api/v1/auth/admin/login ===
const adminLoginSchema = z.object({
  username: z.string().min(1).max(64),
  password: z.string().min(1).max(128),
});

async function adminLogin(request, reply) {
  const parsed = adminLoginSchema.safeParse(request.body);
  if (!parsed.success) return reply.code(400).send({ error: 'invalid_payload' });

  const { username, password } = parsed.data;
  const users = await query(
    'SELECT id, username, password_hash FROM admin_users WHERE username = ? LIMIT 1',
    [username]
  );
  const user = users[0];
  if (!user || !scryptVerify(password, user.password_hash)) {
    return reply.code(401).send({ error: 'invalid_credentials' });
  }

  const token = newToken();
  await query(
    'INSERT INTO admin_sessions (token, user_id, expires_at) VALUES (?, ?, DATE_ADD(NOW(3), INTERVAL ? HOUR))',
    [sha256Hex(token), user.id, config.SESSION_TTL_HOURS]
  );

  return { token, username: user.username };
}

// === POST /api/v1/auth/saksi/login ===
const saksiLoginSchema = z.object({
  no_hp: z.string().min(8).max(16),
  pin: z.string().min(4).max(8),
});

async function saksiLogin(request, reply) {
  const parsed = saksiLoginSchema.safeParse(request.body);
  if (!parsed.success) return reply.code(400).send({ error: 'invalid_payload' });

  const { no_hp, pin } = parsed.data;
  const rows = await query(
    'SELECT id, nama, tps_id, pin_hash, status_hadir FROM saksi WHERE no_hp = ? LIMIT 1',
    [no_hp]
  );
  const saksi = rows[0];
  if (!saksi || !scryptVerify(pin, saksi.pin_hash)) {
    return reply.code(401).send({ error: 'invalid_credentials' });
  }

  const token = signSaksi({
    role: 'saksi',
    sub: saksi.id,
    tps_id: saksi.tps_id,
    exp: Date.now() + config.SESSION_TTL_HOURS * 3600_000,
  });

  return {
    token,
    saksi: { id: saksi.id, nama: saksi.nama, tps_id: saksi.tps_id, status_hadir: saksi.status_hadir },
  };
}

module.exports = async function authRoutes(fastify) {
  fastify.post('/auth/admin/login', adminLogin);
  fastify.post('/auth/saksi/login', saksiLogin);
};
```

### 7.2 `routes/upload.js` - Presigned URL

```javascript
'use strict';
const { z } = require('zod');
const { createPresignedUpload } = require('../services/s3');
const { redis } = require('../services/redis');

const querySchema = z.object({
  fileName: z.string().min(1).max(255).optional(),
  fileType: z.string().regex(/^image\/(jpeg|jpg|png)$/i, 'Hanya JPEG/PNG'),
});

async function getPresignedUrl(request, reply) {
  const parsed = querySchema.safeParse(request.query);
  if (!parsed.success) {
    return reply.code(400).send({ error: 'invalid_query', detail: parsed.error.flatten().fieldErrors });
  }

  if ((await redis.get('kawal:upload_disabled')) === '1') {
    return reply.code(503).send({ error: 'upload_disabled', message: 'Coba lagi sebentar.' });
  }

  const { uploadUrl, objectKey, expiresIn, maxBytes, headers } =
    await createPresignedUpload({ contentType: parsed.data.fileType });

  return { uploadUrl, objectKey, expiresIn, maxBytes, headers };
}

module.exports = async function uploadRoutes(fastify) {
  fastify.get('/upload/presigned-url', { preHandler: fastify.authSaksi }, getPresignedUrl);
};
```

### 7.3 `routes/saksi.js` - Submit C1 & Status

```javascript
'use strict';
const { z } = require('zod');
const { enqueueC1 } = require('../services/queue');
const { createPresignedDownload } = require('../services/s3');
const { query } = require('../services/mysql');

const submitSchema = z.object({
  submission_id: z.string().uuid(),
  tps_id: z.string().min(1).max(24),
  kategori_pemilihan_id: z.string().min(1).max(32),
  foto_object_key: z.string().min(1).max(512),
  data_suara: z.array(z.object({
    kandidat_id: z.string().min(1).max(64),
    jumlah_suara: z.number().int().min(0).max(1000),
  })).min(1),
});

async function submitC1(request, reply) {
  const parsed = submitSchema.safeParse(request.body);
  if (!parsed.success) {
    return reply.code(400).send({ error: 'invalid_payload', detail: parsed.error.flatten().fieldErrors });
  }

  const payload = { ...parsed.data, saksi_id: request.saksi.sub };

  if (payload.tps_id !== request.saksi.tps_id) {
    return reply.code(403).send({ error: 'forbidden', message: 'TPS tidak sesuai login saksi' });
  }

  await enqueueC1(payload);
  return reply.code(202).send({ status: 'queued', submission_id: payload.submission_id });
}

async function getStatus(request, reply) {
  const { tps_id, kategori } = request.params;
  if (tps_id !== request.saksi.tps_id) {
    return reply.code(403).send({ error: 'forbidden' });
  }

  const rows = await query(
    `SELECT submission_id, status, catatan_verifikasi, s3_object_key, created_at
     FROM bukti_c1
     WHERE tps_id = ? AND kategori_pemilihan_id = ? AND saksi_id = ?
     ORDER BY id DESC LIMIT 1`,
    [tps_id, kategori, request.saksi.sub]
  );
  const bukti = rows[0];
  if (!bukti) return reply.code(404).send({ error: 'not_found' });

  return {
    submission_id: bukti.submission_id,
    status: bukti.status,
    catatan_verifikasi: bukti.catatan_verifikasi,
    created_at: bukti.created_at,
    foto_url: await createPresignedDownload(bukti.s3_object_key),
  };
}

module.exports = async function saksiRoutes(fastify) {
  fastify.post('/saksi/c1', { preHandler: fastify.authSaksi }, submitC1);
  fastify.get('/saksi/c1/:tps_id/:kategori', { preHandler: fastify.authSaksi }, getStatus);
};
```

### 7.4 `routes/dashboard.js` - Dashboard & Health

```javascript
'use strict';
const { getRekap } = require('../services/rekap');
const { redis, ping: redisPing } = require('../services/redis');
const { ping: dbPing } = require('../services/mysql');
const { SIREKAP_KEY } = require('../services/scraper');

async function rekap(request, reply) {
  const tingkat = request.query.tingkat || null;
  const dapilId = request.query.dapil_id || null;
  return getRekap({ tingkat, dapilId });
}

async function sirekap(request, reply) {
  const raw = await redis.get(SIREKAP_KEY(request.params.dapil_id));
  if (!raw) return reply.code(404).send({ error: 'not_cached_yet' });
  return reply.header('content-type', 'application/json; charset=utf-8').send(raw);
}

async function healthz(request, reply) {
  const [db, redisOk] = await Promise.all([
    dbPing().then(() => 'ok').catch(e => `error: ${e.message}`),
    redisPing().then(() => 'ok').catch(e => `error: ${e.message}`),
  ]);
  const code = db === 'ok' && redisOk === 'ok' ? 200 : 503;
  return reply.code(code).send({ status: code === 200 ? 'ok' : 'degraded', db, redis: redisOk });
}

module.exports = async function dashboardRoutes(fastify) {
  fastify.get('/dashboard/rekap', rekap);
  fastify.get('/dashboard/sirekap/:dapil_id', sirekap);
  fastify.get('/healthz', healthz);
};
```

### 7.5 `routes/admin.js` - Admin Panel

```javascript
'use strict';
const { z } = require('zod');
const { query } = require('../services/mysql');
const { generatePdf } = require('../services/pdf');

async function listPending(request, reply) {
  const rows = await query(
    `SELECT b.id, b.submission_id, b.tps_id, b.kategori_pemilihan_id, b.s3_object_key,
            b.status, b.created_at, s.nama AS nama_saksi
     FROM bukti_c1 b
     JOIN saksi s ON s.id = b.saksi_id
     WHERE b.status = 'PENDING'
     ORDER BY b.created_at ASC
     LIMIT 200`
  );
  return { items: rows };
}

const verifySchema = z.object({
  status: z.enum(['VALID', 'INVALID', 'DOUBLE']),
  catatan: z.string().max(512).optional(),
});

async function verifyBukti(request, reply) {
  const parsed = verifySchema.safeParse(request.body);
  if (!parsed.success) return reply.code(400).send({ error: 'invalid_payload' });

  const { status, catatan } = parsed.data;
  const buktiId = Number(request.params.bukti_id);
  const result = await query(
    `UPDATE bukti_c1
     SET status = ?, catatan_verifikasi = ?, diverifikasi_oleh = ?, diverifikasi_at = NOW(3)
     WHERE id = ? AND status = 'PENDING'`,
    [status, catatan ?? null, request.admin.id, buktiId]
  );
  if (result.affectedRows === 0) return reply.code(404).send({ error: 'not_found_or_not_pending' });

  await query(
    `INSERT INTO audit_log (actor_type, actor_id, aksi, ref_id, detail)
     VALUES ('ADMIN', ?, 'VERIFIKASI', ?, ?)`,
    [request.admin.id, String(buktiId), JSON.stringify({ status, catatan: catatan ?? null })]
  );

  return { ok: true, bukti_id: buktiId, status };
}

async function exportPdf(request, reply) {
  const jenis = String(request.params.jenis || '');
  if (!/^[a-z0-9_-]+$/i.test(jenis)) return reply.code(400).send({ error: 'invalid_jenis' });
  const buffer = await generatePdf(jenis, { admin: request.admin.username, ts: Date.now() });
  return reply
    .header('content-type', 'application/pdf')
    .header('content-disposition', `attachment; filename="${jenis}.pdf"`)
    .send(buffer);
}

module.exports = async function adminRoutes(fastify) {
  fastify.get('/admin/c1-pending', { preHandler: fastify.authAdmin }, listPending);
  fastify.put('/admin/c1-verifikasi/:bukti_id', { preHandler: fastify.authAdmin }, verifyBukti);
  fastify.get('/admin/export/pdf/:jenis', { preHandler: fastify.authAdmin }, exportPdf);
};
```

---

## BAB 8: FRONTEND RESPONSIF

### 8.1 Filosofi Desain

- **Mobile-first**: dirancang untuk hp, lalu diperbesar ke desktop.
- **CSS Framework**: Tailwind CSS via CDN (tidak perlu build).
- **JavaScript**: Vanilla JS (tidak perlu framework berat). Bisa pakai Alpine.js untuk interaktivitas ringan.
- **Responsive breakpoint**: `sm: 640px`, `md: 768px`, `lg: 1024px`.

### 8.2 Struktur HTML Utama (`index.html`)

```html
<!DOCTYPE html>
<html lang="id" class="scroll-smooth">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Kawal Pemilu - Lampung</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link rel="stylesheet" href="assets/css/app.css">
</head>
<body class="bg-gray-50 text-gray-900 min-h-screen flex flex-col">

  <!-- ==================== HEADER ==================== -->
  <header id="main-header" class="bg-white shadow-md sticky top-0 z-50">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
      <div class="flex items-center justify-between h-16">

        <!-- Logo + Nama -->
        <div class="flex items-center gap-3">
          <img src="assets/img/logo.png" alt="Logo" class="h-8 w-8 sm:h-10 sm:w-10">
          <div>
            <h1 class="text-lg sm:text-xl font-bold text-blue-700">Kawal Pemilu</h1>
            <p class="text-xs text-gray-500 hidden sm:block">Provinsi Lampung</p>
          </div>
        </div>

        <!-- Menu Desktop (md ke atas) -->
        <nav class="hidden md:flex items-center gap-1">
          <a href="#dashboard" class="nav-link px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50 hover:text-blue-700 transition">Dashboard</a>
          <a href="#rekap" class="nav-link px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50 hover:text-blue-700 transition">Rekap</a>
          <a href="#sirekap" class="nav-link px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50 hover:text-blue-700 transition">Sirekap KPU</a>
          <a href="#pengurus" class="nav-link px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50 hover:text-blue-700 transition">Pengurus</a>
          <a href="#kandidat" class="nav-link px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50 hover:text-blue-700 transition">Kandidat</a>
          <a href="#saksi" class="nav-link px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50 hover:text-blue-700 transition">Saksi</a>
          <a href="#dapil" class="nav-link px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50 hover:text-blue-700 transition">Dapil</a>
          <a href="#tps" class="nav-link px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50 hover:text-blue-700 transition">TPS</a>
          <a href="#keuangan" class="nav-link px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50 hover:text-blue-700 transition">Keuangan</a>
        </nav>

        <!-- User Menu + Toggle Mobile -->
        <div class="flex items-center gap-2">
          <span id="user-info" class="text-xs text-gray-500 hidden sm:block"></span>
          <button id="btn-logout" class="hidden px-3 py-1 text-xs bg-red-500 text-white rounded-lg hover:bg-red-600 transition">Logout</button>

          <!-- Hamburger Button (mobile) -->
          <button id="btn-menu-toggle" class="md:hidden p-2 rounded-lg hover:bg-gray-100 transition" aria-label="Toggle menu">
            <svg id="icon-menu" class="h-6 w-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"/>
            </svg>
            <svg id="icon-close" class="h-6 w-6 hidden" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
            </svg>
          </button>
        </div>
      </div>
    </div>

    <!-- Menu Mobile (md ke bawah) -->
    <div id="mobile-menu" class="md:hidden hidden border-t bg-white">
      <nav class="px-4 py-3 space-y-1">
        <a href="#dashboard" class="mobile-nav-link block px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50">Dashboard</a>
        <a href="#rekap" class="mobile-nav-link block px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50">Rekap</a>
        <a href="#sirekap" class="mobile-nav-link block px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50">Sirekap KPU</a>
        <a href="#pengurus" class="mobile-nav-link block px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50">Pengurus</a>
        <a href="#kandidat" class="mobile-nav-link block px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50">Kandidat</a>
        <a href="#saksi" class="mobile-nav-link block px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50">Saksi</a>
        <a href="#dapil" class="mobile-nav-link block px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50">Dapil</a>
        <a href="#tps" class="mobile-nav-link block px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50">TPS</a>
        <a href="#keuangan" class="mobile-nav-link block px-3 py-2 rounded-lg text-sm font-medium hover:bg-blue-50">Keuangan</a>
      </nav>
    </div>
  </header>

  <!-- ==================== LOGIN FORM ==================== -->
  <div id="login-section" class="flex-1 flex items-center justify-center p-4">
    <div class="w-full max-w-md bg-white rounded-2xl shadow-lg p-8">
      <h2 class="text-2xl font-bold text-center mb-6">Masuk</h2>
      <div id="login-tabs" class="flex mb-6 bg-gray-100 rounded-lg p-1">
        <button data-tab="admin" class="login-tab flex-1 py-2 text-sm font-medium rounded-md bg-white shadow">Admin</button>
        <button data-tab="saksi" class="login-tab flex-1 py-2 text-sm font-medium rounded-md text-gray-500">Saksi</button>
      </div>

      <!-- Form Admin -->
      <form id="form-admin-login" class="space-y-4">
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-1">Username</label>
          <input type="text" id="admin-username" class="w-full px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500" placeholder="admin">
        </div>
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-1">Password</label>
          <input type="password" id="admin-password" class="w-full px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500" placeholder="password">
        </div>
        <button type="submit" class="w-full py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition font-medium">Login Admin</button>
      </form>

      <!-- Form Saksi -->
      <form id="form-saksi-login" class="space-y-4 hidden">
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-1">Nomor HP</label>
          <input type="tel" id="saksi-nohp" class="w-full px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500" placeholder="081234567890">
        </div>
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-1">PIN</label>
          <input type="password" id="saksi-pin" class="w-full px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500" placeholder="1234">
        </div>
        <button type="submit" class="w-full py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 transition font-medium">Login Saksi</button>
      </form>

      <p id="login-error" class="text-red-500 text-sm text-center mt-4 hidden"></p>
    </div>
  </div>

  <!-- ==================== MAIN CONTENT ==================== -->
  <main id="app-content" class="flex-1 hidden">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-6">

      <!-- === Section: Dashboard === -->
      <section id="section-dashboard" class="page-section">
        <h2 class="text-2xl font-bold mb-6">Dashboard</h2>

        <!-- Stats Cards -->
        <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4 mb-8">
          <div class="bg-white rounded-xl shadow p-6">
            <p class="text-sm text-gray-500">Total TPS</p>
            <p id="stat-tps" class="text-3xl font-bold text-blue-600">27.440</p>
          </div>
          <div class="bg-white rounded-xl shadow p-6">
            <p class="text-sm text-gray-500">TPS Masuk</p>
            <p id="stat-tps-masuk" class="text-3xl font-bold text-green-600">0</p>
          </div>
          <div class="bg-white rounded-xl shadow p-6">
            <p class="text-sm text-gray-500">Total Suara</p>
            <p id="stat-total-suara" class="text-3xl font-bold text-purple-600">0</p>
          </div>
          <div class="bg-white rounded-xl shadow p-6">
            <p class="text-sm text-gray-500">Persentase Masuk</p>
            <p id="stat-persen" class="text-3xl font-bold text-orange-600">0%</p>
          </div>
        </div>

        <!-- Filter Dapil -->
        <div class="bg-white rounded-xl shadow p-6 mb-6">
          <h3 class="font-bold mb-3">Filter Per Dapil</h3>
          <div class="flex flex-wrap gap-3">
            <select id="filter-tingkat" class="px-4 py-2 border rounded-lg text-sm">
              <option value="">Semua Tingkat</option>
              <option value="DPR_RI">DPR RI</option>
              <option value="DPD">DPD</option>
              <option value="DPRD_PROV">DPRD Provinsi</option>
              <option value="DPRD_KABKOTA">DPRD Kab/Kota</option>
            </select>
            <select id="filter-dapil" class="px-4 py-2 border rounded-lg text-sm">
              <option value="">Semua Dapil</option>
            </select>
          </div>
        </div>

        <!-- Tabel Rekap -->
        <div class="bg-white rounded-xl shadow overflow-hidden">
          <div class="px-6 py-4 border-b">
            <h3 class="font-bold">Rekapitulasi Suara per Dapil</h3>
          </div>
          <div class="overflow-x-auto">
            <table class="w-full text-sm">
              <thead class="bg-gray-50">
                <tr>
                  <th class="px-4 py-3 text-left">Dapil</th>
                  <th class="px-4 py-3 text-left">Kandidat</th>
                  <th class="px-4 py-3 text-left">Partai</th>
                  <th class="px-4 py-3 text-right">No. Urut</th>
                  <th class="px-4 py-3 text-right">Total Suara</th>
                  <th class="px-4 py-3 text-right">TPS Masuk</th>
                </tr>
              </thead>
              <tbody id="rekap-table-body">
                <tr><td colspan="6" class="px-4 py-8 text-center text-gray-400">Memuat data...</td></tr>
              </tbody>
            </table>
          </div>
        </div>
      </section>

      <!-- === Section: Rekap (Halaman Terpisah) === -->
      <section id="section-rekap" class="page-section hidden">
        <h2 class="text-2xl font-bold mb-6">Rekapitulasi</h2>
        <div id="rekap-chart-container" class="bg-white rounded-xl shadow p-6 mb-6" style="height: 400px;">
          <canvas id="rekap-chart"></canvas>
        </div>
      </section>

      <!-- === Section: Sirekap KPU === -->
      <section id="section-sirekap" class="page-section hidden">
        <h2 class="text-2xl font-bold mb-6">Sirekap KPU</h2>
        <div class="bg-white rounded-xl shadow p-6">
          <p class="text-sm text-gray-500 mb-4">Data diambil dari server KPU setiap 30 menit. Ditampilkan dari cache Redis.</p>
          <div id="sirekap-content" class="text-gray-400">Pilih dapil untuk melihat data.</div>
        </div>
      </section>

      <!-- === Section: Pengurus === -->
      <section id="section-pengurus" class="page-section hidden">
        <div class="flex items-center justify-between mb-6">
          <h2 class="text-2xl font-bold">Pengurus</h2>
          <button id="btn-tambah-pengurus" class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 text-sm font-medium">+ Tambah Pengurus</button>
        </div>
        <div class="bg-white rounded-xl shadow overflow-hidden">
          <table class="w-full text-sm">
            <thead class="bg-gray-50">
              <tr>
                <th class="px-4 py-3 text-left">Nama</th>
                <th class="px-4 py-3 text-left">No. HP</th>
                <th class="px-4 py-3 text-left">Tingkat</th>
                <th class="px-4 py-3 text-left">Wilayah</th>
                <th class="px-4 py-3 text-center">Aksi</th>
              </tr>
            </thead>
            <tbody id="pengurus-table-body">
              <tr><td colspan="5" class="px-4 py-8 text-center text-gray-400">Memuat data...</td></tr>
            </tbody>
          </table>
        </div>

        <!-- Modal Tambah Pengurus -->
        <div id="modal-pengurus" class="fixed inset-0 bg-black/50 hidden items-center justify-center z-50">
          <div class="bg-white rounded-2xl shadow-xl w-full max-w-lg mx-4 p-6">
            <h3 class="text-xl font-bold mb-4">Tambah Pengurus</h3>
            <form id="form-pengurus" class="space-y-4">
              <div>
                <label class="block text-sm font-medium mb-1">Nama</label>
                <input type="text" name="nama" class="w-full px-4 py-2 border rounded-lg" required>
              </div>
              <div>
                <label class="block text-sm font-medium mb-1">No. HP</label>
                <input type="tel" name="no_hp" class="w-full px-4 py-2 border rounded-lg" required>
              </div>
              <div>
                <label class="block text-sm font-medium mb-1">Password</label>
                <input type="password" name="password" class="w-full px-4 py-2 border rounded-lg" required>
              </div>
              <div>
                <label class="block text-sm font-medium mb-1">Tingkat</label>
                <select name="tingkat" class="w-full px-4 py-2 border rounded-lg" required>
                  <option value="DPW">DPW (Provinsi)</option>
                  <option value="DPC">DPC (Kabupaten/Kota)</option>
                  <option value="PAC">PAC (Kecamatan)</option>
                  <option value="RANTING">RANTING (Desa)</option>
                  <option value="ANAK_RANTING">ANAK RANTING (TPS)</option>
                </select>
              </div>
              <!-- Dropdown dinamis sesuai tingkat -->
              <div id="pengurus-wilayah-fields"></div>
              <div class="flex gap-3 mt-4">
                <button type="submit" class="flex-1 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700">Simpan</button>
                <button type="button" id="btn-batal-pengurus" class="flex-1 py-2 bg-gray-200 text-gray-700 rounded-lg hover:bg-gray-300">Batal</button>
              </div>
            </form>
          </div>
        </div>
      </section>

      <!-- === Section: Kandidat === -->
      <section id="section-kandidat" class="page-section hidden">
        <div class="flex items-center justify-between mb-6">
          <h2 class="text-2xl font-bold">Kandidat</h2>
          <button id="btn-tambah-kandidat" class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 text-sm font-medium">+ Tambah Kandidat</button>
        </div>

        <!-- Filter -->
        <div class="bg-white rounded-xl shadow p-4 mb-6">
          <div class="flex flex-wrap gap-3">
            <select id="kandidat-filter-kategori" class="px-4 py-2 border rounded-lg text-sm">
              <option value="">Semua Kategori</option>
              <option value="01">DPRD Provinsi</option>
              <option value="02">DPR RI</option>
              <option value="03">DPRD Kabupaten</option>
              <option value="04">DPD</option>
              <option value="05">Pilgub</option>
              <option value="06">Pilbup/Pilwakota</option>
            </select>
            <select id="kandidat-filter-dapil" class="px-4 py-2 border rounded-lg text-sm">
              <option value="">Semua Dapil</option>
            </select>
          </div>
        </div>

        <div class="bg-white rounded-xl shadow overflow-hidden">
          <table class="w-full text-sm">
            <thead class="bg-gray-50">
              <tr>
                <th class="px-4 py-3 text-left">Foto</th>
                <th class="px-4 py-3 text-left">Nama</th>
                <th class="px-4 py-3 text-left">Tipe</th>
                <th class="px-4 py-3 text-left">Partai</th>
                <th class="px-4 py-3 text-left">Dapil</th>
                <th class="px-4 py-3 text-right">No. Urut</th>
                <th class="px-4 py-3 text-center">Aksi</th>
              </tr>
            </thead>
            <tbody id="kandidat-table-body">
              <tr><td colspan="7" class="px-4 py-8 text-center text-gray-400">Memuat data...</td></tr>
            </tbody>
          </table>
        </div>
      </section>

      <!-- === Section: Saksi === -->
      <section id="section-saksi" class="page-section hidden">
        <div class="flex items-center justify-between mb-6">
          <h2 class="text-2xl font-bold">Saksi</h2>
          <button id="btn-tambah-saksi" class="px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 text-sm font-medium">+ Daftarkan Saksi</button>
        </div>

        <div class="bg-white rounded-xl shadow overflow-hidden">
          <table class="w-full text-sm">
            <thead class="bg-gray-50">
              <tr>
                <th class="px-4 py-3 text-left">Nama</th>
                <th class="px-4 py-3 text-left">No. HP</th>
                <th class="px-4 py-3 text-left">TPS</th>
                <th class="px-4 py-3 text-left">Status</th>
                <th class="px-4 py-3 text-center">Aksi</th>
              </tr>
            </thead>
            <tbody id="saksi-table-body">
              <tr><td colspan="5" class="px-4 py-8 text-center text-gray-400">Memuat data...</td></tr>
            </tbody>
          </table>
        </div>
      </section>

      <!-- === Section: Dapil === -->
      <section id="section-dapil" class="page-section hidden">
        <div class="flex items-center justify-between mb-6">
          <h2 class="text-2xl font-bold">Manajemen Dapil</h2>
        </div>
        <div class="bg-white rounded-xl shadow overflow-hidden">
          <table class="w-full text-sm">
            <thead class="bg-gray-50">
              <tr>
                <th class="px-4 py-3 text-left">ID</th>
                <th class="px-4 py-3 text-left">Nama</th>
                <th class="px-4 py-3 text-left">Tingkat</th>
                <th class="px-4 py-3 text-left">Kode</th>
                <th class="px-4 py-3 text-right">Jml Kursi</th>
              </tr>
            </thead>
            <tbody id="dapil-table-body">
              <tr><td colspan="5" class="px-4 py-8 text-center text-gray-400">Memuat data...</td></tr>
            </tbody>
          </table>
        </div>
      </section>

      <!-- === Section: TPS === -->
      <section id="section-tps" class="page-section hidden">
        <div class="flex items-center justify-between mb-6">
          <h2 class="text-2xl font-bold">Manajemen TPS</h2>
        </div>

        <div class="bg-white rounded-xl shadow p-4 mb-6">
          <div class="flex flex-wrap gap-3">
            <select id="tps-filter-kab" class="px-4 py-2 border rounded-lg text-sm">
              <option value="">Semua Kabupaten</option>
            </select>
            <select id="tps-filter-kec" class="px-4 py-2 border rounded-lg text-sm">
              <option value="">Semua Kecamatan</option>
            </select>
            <select id="tps-filter-desa" class="px-4 py-2 border rounded-lg text-sm">
              <option value="">Semua Desa</option>
            </select>
          </div>
        </div>

        <div class="bg-white rounded-xl shadow overflow-hidden">
          <table class="w-full text-sm">
            <thead class="bg-gray-50">
              <tr>
                <th class="px-4 py-3 text-left">Kode TPS</th>
                <th class="px-4 py-3 text-left">Desa</th>
                <th class="px-4 py-3 text-left">No. TPS</th>
                <th class="px-4 py-3 text-right">Jml Pemilih</th>
              </tr>
            </thead>
            <tbody id="tps-table-body">
              <tr><td colspan="4" class="px-4 py-8 text-center text-gray-400">Memuat data...</td></tr>
            </tbody>
          </table>
        </div>
      </section>

      <!-- === Section: Keuangan === -->
      <section id="section-keuangan" class="page-section hidden">
        <div class="flex items-center justify-between mb-6">
          <h2 class="text-2xl font-bold">Manajemen Biaya</h2>
          <button id="btn-tambah-keuangan" class="px-4 py-2 bg-orange-600 text-white rounded-lg hover:bg-orange-700 text-sm font-medium">+ Catat Biaya</button>
        </div>

        <!-- Ringkasan -->
        <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 mb-6">
          <div class="bg-white rounded-xl shadow p-6">
            <p class="text-sm text-gray-500">Total Logistik</p>
            <p id="total-logistik" class="text-2xl font-bold text-orange-600">Rp 0</p>
          </div>
          <div class="bg-white rounded-xl shadow p-6">
            <p class="text-sm text-gray-500">Total Dana Saksi</p>
            <p id="total-saksi" class="text-2xl font-bold text-green-600">Rp 0</p>
          </div>
        </div>

        <div class="bg-white rounded-xl shadow overflow-hidden">
          <table class="w-full text-sm">
            <thead class="bg-gray-50">
              <tr>
                <th class="px-4 py-3 text-left">Tanggal</th>
                <th class="px-4 py-3 text-left">Kategori</th>
                <th class="px-4 py-3 text-left">Deskripsi</th>
                <th class="px-4 py-3 text-left">Wilayah</th>
                <th class="px-4 py-3 text-right">Nominal</th>
                <th class="px-4 py-3 text-center">Aksi</th>
              </tr>
            </thead>
            <tbody id="keuangan-table-body">
              <tr><td colspan="6" class="px-4 py-8 text-center text-gray-400">Memuat data...</td></tr>
            </tbody>
          </table>
        </div>
      </section>

    </div>
  </main>

  <!-- ==================== FOOTER ==================== -->
  <footer class="bg-white border-t mt-auto">
    <div class="max-w-7xl mx-auto px-4 py-4 text-center text-sm text-gray-500">
      Kawal Pemilu &copy; 2026 &mdash; Provinsi Lampung
    </div>
  </footer>

  <script src="assets/js/app.js"></script>
</body>
</html>
```

### 8.3 JavaScript Utama (`assets/js/app.js`)

```javascript
'use strict';

// ==================== API CLIENT ====================
const API_BASE = window.location.origin + '/api/v1';

async function apiFetch(path, options = {}) {
  const token = localStorage.getItem('token');
  const headers = { 'Content-Type': 'application/json', ...options.headers };
  if (token) headers['Authorization'] = `Bearer ${token}`;

  const res = await fetch(API_BASE + path, { ...options, headers });

  if (res.status === 401) {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
    showLogin();
    throw new Error('Sesi berakhir');
  }

  const data = await res.json();
  if (!res.ok) throw new Error(data.error || 'Request gagal');
  return data;
}

// ==================== STATE ====================
let currentUser = null;
let currentPage = 'dashboard';

// ==================== LOGIN ====================
const loginSection = document.getElementById('login-section');
const appContent = document.getElementById('app-content');
const btnLogout = document.getElementById('btn-logout');
const userInfo = document.getElementById('user-info');
const loginError = document.getElementById('login-error');
const loginTabs = document.querySelectorAll('.login-tab');
const formAdminLogin = document.getElementById('form-admin-login');
const formSaksiLogin = document.getElementById('form-saksi-login');

loginTabs.forEach(tab => {
  tab.addEventListener('click', () => {
    loginTabs.forEach(t => {
      t.classList.remove('bg-white', 'shadow');
      t.classList.add('text-gray-500');
    });
    tab.classList.add('bg-white', 'shadow');
    tab.classList.remove('text-gray-500');

    if (tab.dataset.tab === 'admin') {
      formAdminLogin.classList.remove('hidden');
      formSaksiLogin.classList.add('hidden');
    } else {
      formAdminLogin.classList.add('hidden');
      formSaksiLogin.classList.remove('hidden');
    }
  });
});

formAdminLogin.addEventListener('submit', async (e) => {
  e.preventDefault();
  loginError.classList.add('hidden');
  try {
    const data = await apiFetch('/auth/admin/login', {
      method: 'POST',
      body: JSON.stringify({
        username: document.getElementById('admin-username').value,
        password: document.getElementById('admin-password').value,
      }),
    });
    localStorage.setItem('token', data.token);
    localStorage.setItem('user', JSON.stringify({ role: 'admin', username: data.username }));
    showApp();
  } catch (err) {
    loginError.textContent = err.message;
    loginError.classList.remove('hidden');
  }
});

formSaksiLogin.addEventListener('submit', async (e) => {
  e.preventDefault();
  loginError.classList.add('hidden');
  try {
    const data = await apiFetch('/auth/saksi/login', {
      method: 'POST',
      body: JSON.stringify({
        no_hp: document.getElementById('saksi-nohp').value,
        pin: document.getElementById('saksi-pin').value,
      }),
    });
    localStorage.setItem('token', data.token);
    localStorage.setItem('user', JSON.stringify({ role: 'saksi', ...data.saksi }));
    showApp();
  } catch (err) {
    loginError.textContent = err.message;
    loginError.classList.remove('hidden');
  }
});

btnLogout.addEventListener('click', () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
  showLogin();
});

// ==================== NAVIGATION ====================
function showLogin() {
  loginSection.classList.remove('hidden');
  appContent.classList.add('hidden');
  btnLogout.classList.add('hidden');
  userInfo.classList.add('hidden');
  currentUser = null;
}

function showApp() {
  currentUser = JSON.parse(localStorage.getItem('user') || 'null');
  if (!currentUser) return showLogin();

  loginSection.classList.add('hidden');
  appContent.classList.remove('hidden');
  btnLogout.classList.remove('hidden');
  userInfo.classList.remove('hidden');
  userInfo.textContent = `${currentUser.role === 'admin' ? 'Admin' : 'Saksi'}: ${currentUser.username || currentUser.nama || currentUser.no_hp}`;

  navigateTo('dashboard');
}

// ==================== MOBILE MENU ====================
const btnMenuToggle = document.getElementById('btn-menu-toggle');
const mobileMenu = document.getElementById('mobile-menu');
const iconMenu = document.getElementById('icon-menu');
const iconClose = document.getElementById('icon-close');

btnMenuToggle.addEventListener('click', () => {
  mobileMenu.classList.toggle('hidden');
  iconMenu.classList.toggle('hidden');
  iconClose.classList.toggle('hidden');
});

// Close mobile menu on link click
document.querySelectorAll('.mobile-nav-link').forEach(link => {
  link.addEventListener('click', () => {
    mobileMenu.classList.add('hidden');
    iconMenu.classList.remove('hidden');
    iconClose.classList.add('hidden');
  });
});

// ==================== PAGE NAVIGATION ====================
const navLinks = document.querySelectorAll('.nav-link');
const mobileNavLinks = document.querySelectorAll('.mobile-nav-link');

function navigateTo(page) {
  currentPage = page;

  // Hide all sections
  document.querySelectorAll('.page-section').forEach(s => s.classList.add('hidden'));

  // Show target section
  const section = document.getElementById(`section-${page}`);
  if (section) section.classList.remove('hidden');

  // Update active nav
  [...navLinks, ...mobileNavLinks].forEach(l => {
    l.classList.remove('bg-blue-50', 'text-blue-700');
  });
  document.querySelectorAll(`[href="#${page}"]`).forEach(l => {
    l.classList.add('bg-blue-50', 'text-blue-700');
  });

  // Load data for page
  loadPageData(page);
}

navLinks.forEach(link => link.addEventListener('click', (e) => { e.preventDefault(); navigateTo(link.getAttribute('href').slice(1)); }));
mobileNavLinks.forEach(link => link.addEventListener('click', (e) => { e.preventDefault(); navigateTo(link.getAttribute('href').slice(1)); }));

// ==================== LOAD DATA ====================
async function loadPageData(page) {
  try {
    switch (page) {
      case 'dashboard':
      case 'rekap':
        await loadRekap();
        break;
      case 'sirekap':
        await loadSirekap();
        break;
      case 'pengurus':
        await loadPengurus();
        break;
      case 'kandidat':
        await loadKandidat();
        break;
      case 'saksi':
        await loadSaksi();
        break;
      case 'dapil':
        await loadDapil();
        break;
      case 'tps':
        await loadTps();
        break;
      case 'keuangan':
        await loadKeuangan();
        break;
    }
  } catch (err) {
    console.error(`Gagal memuat ${page}:`, err);
  }
}

// ==================== LOAD REKAP ====================
async function loadRekap() {
  const tingkat = document.getElementById('filter-tingkat')?.value || '';
  const dapilId = document.getElementById('filter-dapil')?.value || '';

  const params = new URLSearchParams();
  if (tingkat) params.set('tingkat', tingkat);
  if (dapilId) params.set('dapil_id', dapilId);

  const data = await apiFetch(`/dashboard/rekap?${params}`);
  const rows = data.rows || [];

  // Update stats
  if (rows.length > 0) {
    const totalSuara = rows.reduce((sum, r) => sum + (r.total_suara || 0), 0);
    const tpsMasuk = new Set(rows.map(r => r.tps_id)).size;
    document.getElementById('stat-tps-masuk').textContent = tpsMasuk.toLocaleString('id-ID');
    document.getElementById('stat-total-suara').textContent = totalSuara.toLocaleString('id-ID');
    document.getElementById('stat-persen').textContent = ((tpsMasuk / 27440) * 100).toFixed(1) + '%';
  }

  // Fill table
  const tbody = document.getElementById('rekap-table-body');
  if (rows.length === 0) {
    tbody.innerHTML = '<tr><td colspan="6" class="px-4 py-8 text-center text-gray-400">Belum ada data rekap</td></tr>';
    return;
  }

  tbody.innerHTML = rows.map(r => `
    <tr class="border-t hover:bg-gray-50">
      <td class="px-4 py-3">${r.dapil_id || '-'}</td>
      <td class="px-4 py-3">${r.nama_kandidat}</td>
      <td class="px-4 py-3">${r.partai || '-'}</td>
      <td class="px-4 py-3 text-right">${r.no_urut}</td>
      <td class="px-4 py-3 text-right font-bold">${(r.total_suara || 0).toLocaleString('id-ID')}</td>
      <td class="px-4 py-3 text-right">${r.tps_masuk || 0}</td>
    </tr>
  `).join('');
}

// ==================== LOAD DAPIL ====================
async function loadDapil() {
  // Dapil data dari master - perlu endpoint atau load dari rekap
  const tbody = document.getElementById('dapil-table-body');
  tbody.innerHTML = '<tr><td colspan="5" class="px-4 py-8 text-center text-gray-400">Memuat data dapil...</td></tr>';
}

// ==================== INIT ====================
(function init() {
  const saved = localStorage.getItem('user');
  if (saved && localStorage.getItem('token')) {
    showApp();
  } else {
    showLogin();
  }
})();
```

### 8.4 CSS Custom (`assets/css/app.css`)

```css
/* Override Tailwind defaults */
.nav-link.active,
.mobile-nav-link.active {
  @apply bg-blue-50 text-blue-700 font-semibold;
}

/* Smooth transitions */
.page-section {
  animation: fadeIn 0.2s ease-in;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(8px); }
  to { opacity: 1; transform: translateY(0); }
}

/* Scrollable table on mobile */
.table-responsive {
  @apply overflow-x-auto;
}

/* Loading spinner */
.spinner {
  border: 3px solid #f3f3f3;
  border-top: 3px solid #3498db;
  border-radius: 50%;
  width: 24px;
  height: 24px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
```

---

## BAB 9: PDF SERVICE (PHP mPDF)

### 9.1 `pdf-service/generate.php`

```php
<?php
require_once __DIR__ . '/vendor/autoload.php';

// Token autentikasi antar service
$expectedToken = getenv('PDF_SERVICE_TOKEN') ?: 'change-me';

$headers = getallheaders();
$token = $headers['X-Service-Token'] ?? '';
if (!hash_equals($expectedToken, $token)) {
    http_response_code(401);
    echo json_encode(['error' => 'unauthorized']);
    exit;
}

$input = json_decode(file_get_contents('php://input'), true);
$jenis = $input['jenis'] ?? '';
$data = $input['data'] ?? [];

// Generate PDF berdasarkan jenis
$mpdf = new \Mpdf\Mpdf([
    'mode' => 'utf-8',
    'format' => 'A4',
    'margin_left' => 15,
    'margin_right' => 15,
    'margin_top' => 15,
    'margin_bottom' => 15,
]);

$html = generateHtml($jenis, $data);
$mpdf->WriteHTML($html);

// Output binary PDF
header('Content-Type: application/pdf');
echo $mpdf->Output('', 'S'); // 'S' = return as string

function generateHtml($jenis, $data) {
    $ts = date('Y-m-d H:i:s', ($data['ts'] ?? time()) / 1000);

    return <<<HTML
    <!DOCTYPE html>
    <html>
    <head>
        <style>
            body { font-family: sans-serif; font-size: 12pt; }
            h1 { text-align: center; color: #1a56db; }
            table { width: 100%; border-collapse: collapse; margin-top: 20px; }
            th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
            th { background-color: #1a56db; color: white; }
            .footer { text-align: center; font-size: 10pt; color: #666; margin-top: 30px; }
        </style>
    </head>
    <body>
        <h1>Kawal Pemilu - Lampung</h1>
        <p><strong>Jenis Laporan:</strong> {$jenis}</p>
        <p><strong>Admin:</strong> {$data['admin']}</p>
        <p><strong>Waktu Cetak:</strong> {$ts}</p>
        <div class="footer">
            Dokumen ini digenerate otomatis oleh sistem Kawal Pemilu.
        </div>
    </body>
    </html>
    HTML;
}
```

### 9.2 `pdf-service/composer.json`

```json
{
  "require": {
    "mpdf/mpdf": "^8.2.0"
  }
}
```

### 9.3 Menjalankan PDF Service

```bash
cd pdf-service
composer install
PDF_SERVICE_TOKEN=change-me php -S 0.0.0.0:8000 -t .
```

---

## BAB 10: SETUP LOCALHOST

### 10.1 Prasyarat

- Node.js 20 LTS
- MySQL 8.x (Laragon / XAMPP / Docker)
- Redis 8.x (Docker atau install manual)
- MinIO (Docker)
- PHP 8.x + Composer (untuk PDF service)

### 10.2 Langkah Setup

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

# Import database
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

# ============================================================
# BAGIAN KEDUA: SERVER KHUSUS HARI H (PECAH SERVER)
# ============================================================

![Bagian 2](bagian%202.png)

---

## BAB 11: MASALAH: 30.000 TPS SERENTAK

### 11.1 Skenario Beban

```
Hari H Pemilu:
- 27.440 TPS di Provinsi Lampung
- ±15.000 saksi aktif mengirim Form C1
- Peak: 5.000 request/menit (submit + upload foto)
- Setiap submit: 1 foto 2-5MB + JSON angka suara
```

### 11.2 Mengapa Server Biasa Pecah

```
Jika Node.js menerima gambar langsung:
  5.000 request × 3MB = 15GB bandwidth masuk per menit
  Node.js single-thread → buffer semua di RAM → OOM → crash

Jika MySQL di-insert langsung dari API:
  5.000 INSERT/menit × 3 tabel (bukti + suara + audit) = 15.000 query/menit
  MySQL connection pool habis → lock → timeout → crash
```

### 11.3 Solusi: Arsitektur Terpisah

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
│  [S3: NEO Object Storage] ← Upload foto langsung      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 11.4 Strategi Scaling

| Fase        | Server                                              | Biaya/bulan        |
| ----------- | --------------------------------------------------- | ------------------ |
| Pra-pemilu  | 1 VPS (API + Worker + MySQL + Redis)                | ±Rp389.000         |
| Hari H      | 5 VPS (LB + 2 API + Worker/Redis + MySQL dedicated)| ±Rp4.168.050       |

---

## BAB 12: DEPLOY SCRIPTS

### 12.1 `deploy/pm2/api.ecosystem.config.js`

```javascript
module.exports = {
  apps: [
    {
      name: 'kawal-api',
      script: 'src/server.js',
      instances: 2,        // cluster mode
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3000,
      },
      error_file: 'logs/api-error.log',
      out_file: 'logs/api-out.log',
      max_memory_restart: '512M',
    },
  ],
};
```

### 12.2 `deploy/pm2/worker.ecosystem.config.js`

```javascript
module.exports = {
  apps: [
    {
      name: 'kawal-worker',
      script: 'src/worker.js',
      instances: 1,
      exec_mode: 'fork',
      env: {
        NODE_ENV: 'production',
        WORKER_CONCURRENCY: 10,
      },
      error_file: 'logs/worker-error.log',
      out_file: 'logs/worker-out.log',
      max_memory_restart: '256M',
    },
  ],
};
```

### 12.3 `deploy/nginx/kawal-pemilu.conf.j2`

```nginx
upstream api_backend {
    least_conn;
    server api-1:3000 max_fails=3 fail_timeout=10s;
    server api-2:3000 max_fails=3 fail_timeout=10s;
    keepalive 64;
}

server {
    listen 80;
    server_name {{ DOMAIN }};
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name {{ DOMAIN }};

    ssl_certificate /etc/letsencrypt/live/{{ DOMAIN }}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{{ DOMAIN }}/privkey.pem;

    # Frontend statis
    location / {
        root /var/www/kawal-pemilu/frontend;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # API proxy
    location /api/ {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 30s;

        # Buffer
        proxy_buffering on;
        proxy_buffer_size 8k;
        proxy_buffers 16 8k;
    }

    # Health check
    location /healthz {
        proxy_pass http://api_backend/api/v1/healthz;
        access_log off;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 256;
}
```

### 12.4 `deploy/mysql/mysqld.cnf` (Hari H, 8GB RAM)

```ini
[mysqld]
# === Basic ===
port = 3306
bind-address = 0.0.0.0
datadir = /var/lib/mysql
socket = /var/run/mysqld/mysqld.sock

# === InnoDB (buffer pool 4GB dari 8GB RAM) ===
innodb_buffer_pool_size = 4G
innodb_buffer_pool_instances = 4
innodb_log_file_size = 512M
innodb_log_buffer_size = 64M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT

# === Connections ===
max_connections = 500
wait_timeout = 300
interactive_timeout = 300

# === Query Cache (disabled di MySQL 8.x) ===

# === Binary Log (untuk replikasi & DR) ===
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
expire_logs_days = 7
sync_binlog = 1

# === Character Set ===
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

### 12.5 `deploy/redis/redis-hari-h.conf`

```ini
# Persistensi: AOF + RDB snapshot
appendonly yes
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

save 900 1
save 300 10
save 60 10000

# Memory
maxmemory 1gb
maxmemory-policy allkeys-lru

# Networking
bind 0.0.0.0
protected-mode no
```

---

## BAB 13: RUNBOOK HARI H

### 13.1 Persiapan (H-7 s.d. H-1)

```
H-7:  Provisioning semua server dari template
      npm run migrate && npm run seed
      Load test: k6 run scripts/load-test.js (target 500 req/s)

H-3:  Backup penuh: mysqldump + binlog streaming
      Uji restore di server cadangan

H-1:  Pastikan semua server hijau (healthz = 200)
      Matikan fitur non-esensial
      Briefing tim: siapa yang standby jam berapa
```

### 13.2 Hari H (T-0 s.d. T+24 jam)

```
T-2:  Aktifkan server CADANGAN (warm standby)
      Monitor queue depth (Grafana)

T-0:  PEMILU DIMULAI
      ┌─────────────────────────────────────┐
      │ MONITOR SETIAP 30 MENIT:            │
      │ 1. GET /healthz → semua 200?        │
      │ 2. Queue depth di Redis             │
      │ 3. MySQL connection count           │
      │ 4. Error rate API                   │
      │ 5. Storage S3                       │
      └─────────────────────────────────────┘

T+4:  Puncak submit C1 saksi
      → Scale up Worker jika queue > 10.000

T+12: Semua TPS selesai submit
      → Mulai verifikasi (admin)

T+24: Update rekap akhir
```

### 13.3 Troubleshooting

| Masalah                     | Solusi                                               |
| --------------------------- | ---------------------------------------------------- |
| API timeout                 | Cek worker; scale up jika queue panjang              |
| MySQL crash                 | Promote replika; restore dari binlog                 |
| Redis hilang                | Restart dari AOF; job di-retry otomatis              |
| S3 tidak bisa diakses       | Aktifkan kill-switch (flag Redis)                    |
| Total gagal 1 server        | Provision dari template; restore DB dari backup      |

---

## BAB 14: ESTIMASI BIAYA TAHUNAN

### 14.1 Biaya Pra-Pemilu (11 bulan)

| Komponen              | Harga/bulan   | 11 bulan        |
| --------------------- | ------------- | --------------- |
| 1 VPS Lite Pro MS.2.2 | Rp389.000     | Rp4.279.000     |
| Object Storage 10GB   | Rp10.000      | Rp110.000       |
| Domain + SSL          | Rp8.250       | Rp90.750        |
| Monitoring            | Rp10.000      | Rp110.000       |
| **Subtotal**          |               | **Rp4.589.750** |

### 14.2 Biaya Hari H (1 bulan peak)

| Komponen                  | Harga/bulan   |
| ------------------------- | ------------- |
| LB + 2 API (MM.4.4 × 2)  | Rp1.398.000   |
| Worker + Redis (MM.4.4)   | Rp699.000     |
| MySQL dedicated (MM.8.4)  | Rp999.000     |
| Object Storage 100GB      | Rp100.000     |
| Snapshot 80GB             | Rp160.000     |
| Monitoring                | Rp10.000      |
| **Subtotal**              | **Rp3.366.000** |
| + PPN 11%                 | Rp370.260     |
| **Total**                 | **Rp3.736.260** |

### 14.3 Total Tahunan

```
Pra-pemilu (11 bulan):  Rp4.589.750
Hari H (1 bulan):       Rp3.736.260
────────────────────────────────────
TOTAL TAHUNAN:          Rp8.326.010 (sudah PPN)
```

---

## BAB 15: CHECKLIST IMPLEMENTASI

### Tahap 1: Fondasi (Minggu 1-2)

- [ ] Setup MySQL + import `db.sql`
- [ ] Jalankan `001_operational.sql`
- [ ] Setup Redis + MinIO (Docker)
- [ ] Buat `api/.env` dari `.env.example`
- [ ] Implementasi `config.js`, `load-env.js`, `crypto.js`
- [ ] Implementasi `services/mysql.js`, `redis.js`, `s3.js`
- [ ] Implementasi `middleware/reject-multipart.js`
- [ ] Implementasi `middleware/auth.js`
- [ ] Test: `npm run test` → semua hijau

### Tahap 2: API Auth & Upload (Minggu 3)

- [ ] Route `auth.js`: login admin + login saksi
- [ ] Route `upload.js`: presigned URL
- [ ] Test login berhasil
- [ ] Test presigned URL → upload ke MinIO

### Tahap 3: Queue & Worker (Minggu 4)

- [ ] Implementasi `services/queue.js` (BullMQ)
- [ ] Implementasi `services/c1-processor.js`
- [ ] Route `saksi.js`: submit C1 → queue → worker
- [ ] Test: submit → queue → worker insert ke MySQL
- [ ] Test: submit ulang idempotent (tidak ganda)

### Tahap 4: Dashboard & Rekap (Minggu 5)

- [ ] Route `dashboard.js`: rekap + sirekap + healthz
- [ ] Implementasi `services/rekap.js` (Redis cache)
- [ ] Implementasi `services/scraper.js` (Sirekap)
- [ ] View `v_rekap_dapil` di MySQL

### Tahap 5: Admin Panel (Minggu 6)

- [ ] Route `admin.js`: list pending, verifikasi, export PDF
- [ ] Microservice PDF (`pdf-service/generate.php`)
- [ ] Test verifikasi → audit log tercatat

### Tahap 6: Frontend (Minggu 7-8)

- [ ]

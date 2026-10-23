---
title: First-Run Setup — Analisa & Plan
description: ""
status: complete
created: "2026-10-23"
updated: "2026-10-23"
tags:
    - setup
    - first-run
    - donisgo
---
# SPEC: First-Run Setup

## Ringkasan
Menyiapkan **BBH Data Board** (repo saat ini adalah aplikasi berbasis framework donisgo) agar bisa dijalankan pertama kali secara lokal dengan alur setup yang reproducible, terdokumentasi, dan dapat diverifikasi.

## Goal
1. Backend Go berhasil dibangun, terhubung ke database lokal, dan melayani HTTP di port 8080.
2. Frontend React/Vite berhasil dijalankan dan dapat mengakses backend melalui proxy dev atau SPA hasil build.

### Kriteria Sukses (Acceptance Criteria)
- [AC-1] Instruksi first-run menyebutkan prerequisites, konfigurasi lokal, dan urutan start yang konsisten.
- [AC-2] `.env.example` mempunyai satu konfigurasi lokal yang konsisten untuk pilihan database yang didukung; tidak ada nilai container hostname yang keliru untuk penggunaan host lokal.
- [AC-3] Database yang dipilih dapat dinyalakan secara lokal dengan langkah yang terdokumentasi (Compose atau service lokal), dan koneksi dapat diverifikasi.
- [AC-4] Backend dapat dibangun (`go build ./cmd/server` dan `go build ./cmd/console`) dan berjalan tanpa crash setelah database tersedia, listen di `:8080`.
- [AC-5] Frontend dependency install dan production build berhasil (`npm ci && npm run build` di `sub_app/webapp`).
- [AC-6] Frontend dapat dijalankan di dev mode pada port 5173 dan proxy `/api` serta `/admin` menuju backend 8080; atau SPA hasil build tersedia di `/app/` ketika diserve oleh backend.
- [AC-7] Prosedur verifikasi mencakup hasil HTTP untuk backend dan frontend, serta langkah troubleshooting untuk dependency/configuration/database.

## Batasan Scope
- Tidak membuat model, endpoint, akun, atau skema bisnis BBH Data Board yang belum ditentukan.
- Tidak membuat migrasi dummy hanya demi menandai setup selesai. Repo belum memiliki direktori migration; startup server saat ini membutuhkan database, tetapi tidak menjalankan migrasi otomatis. Kebutuhan migrasi schema ditunda sampai ada schema aplikasi yang disepakati.
- Swagger generation bersifat opsional untuk first run; dokumentasi Swagger belum ada dan tidak menghalangi server start.
- Pilih satu database default untuk panduan first run; PostgreSQL adalah opsi yang sesuai dengan konfigurasi contoh awal repo. Dukungan MySQL tetap dipertahankan bila sudah dipakai pengguna.

## Analisa Proyek
| Aspek | Temuan |
|---|---|
| **Backend** | Go 1.25+ / Gin + GORM / PostgreSQL or MySQL |
| **CLI** | `console` binary — migration, plugin generation, seeding |
| **Frontend** | React 19 + TypeScript + Vite (SPA di `/app/`) |
| **Env** | `.env.example` kontradiktif: blok atas pakai Postgres + Docker names; blok bawah pakai MySQL + localhost. **Tidak ada file `.env`**. |
| **Docker Compose** | Tidak ditemukan (`make-sync.yaml` juga tidak ada). |
| **Migrations** | Folder `migrations/` belum ada sama sekali. |
| **Swagger** | Belum ada folder `docs/swagger/`, swag CLI belum terinstall. |
| **Build Output** | `bin/` belum ada. |
| **Tools Lokal** | Go 1.27.3 ✓, Node v24 ✓, npm ✓, pnpm ✓, docker compose ✓ |

## Langkah Setup (Panduan User)

### Langkah 1 — Prerequisites
Gunakan salah satu database lokal yang didukung oleh framework:
- **PostgreSQL** (disarankan untuk first-run) — install di host atau gunakan container `docker run`.
- **MySQL / MariaDB** — juga didukung; sesuaikan langkah ini dengan driver yang dipilih.

Pastikan Go ≥ 1.25, Node.js ≥ 18, dan npm/pnpm terinstall.

### Langkah 2 — Konfigurasi Environment Lokal
Salin `.env.example` ke `.env`, lalu sesuaikan nilai berikut agar cocok untuk pengembangan lokal:

```env
APP_NAME=BBH Data Board
PORT=8080

# Pilih salah satu database. Contoh: PostgreSQL lokal.
DB_TYPE=postgres
DB_HOST=127.0.0.1
DB_PORT=5432
DB_USER=<user>
DB_PASSWORD=<pass>
DB_NAME=bbh_data_board

JWT_SECRET=<generate strong random secret>
CORS_ALLOWED_ORIGINS=http://localhost:5173

STORAGE_DRIVER=local
STORAGE_ROOT=./storage
STORAGE_PUBLIC_URL=http://localhost:8080/assets

LOG_LEVEL=info
KEYDB_HOST=
```

Jika menggunakan MySQL, ganti `DB_TYPE=mysql` dan konfigurasi DSN sesuai driver MySQL.
Kosongkan `KEYDB_HOST` bila tidak ada cache Redis/KeyDB — non-fatal warning hanya muncul saat startup.

### Langkah 3 — Nyalakan Database
Untuk PostgreSQL via Docker:
```bash
docker run -d --name bbb-db \
  -e POSTGRES_USER=<user> -e POSTGRES_PASSWORD=<pass> -e POSTGRES_DB=bbh_data_board \
  -p 5432:5432 postgres:16-alpine
```

Verifikasi koneksi setelah container berjalan. Jika memakai database host lokal (native install), pastikan service aktif dan kredensial sesuai `.env`.

### Langkah 4 — Build Backend
```bash
cd <repo-root>
go mod tidy
./build.sh   # build.sh: go mod tidy + build server/console + swag init
```
Hasil: `bin/server`, `bin/console`, dan (jika swag CLI tersedia) `docs/swagger/swagger.json`.

Alternatif tanpa build.sh:
```bash
go build -o bin/server ./cmd/server
go build -o bin/console ./cmd/console
```

### Langkah 5 — Jalankan Server
```bash
bin/server    # atau: go run ./cmd/server
```
Server listening di `http://0.0.0.0:8080`.

### Langkah 6 — Instal & Jalankan Frontend
Di terminal lain:
```bash
cd sub_app/webapp && npm ci
```
Jalankan dev server:
```bash
npx vite         # listen di localhost:5173
```
Atau production build (hasil akan diserve plugin `pages`):
```bash
npx vite build    # → dist/
```

Frontend dev server mem-proxy `/api` dan `/admin` ke `http://localhost:8080`. Akses UI melalui `http://localhost:5173`.

### Langkah 7 — Verifikasi First Run
| Endpoint | Ekspektasi |
|---|---|
| `http://localhost:8080/` | Menampilkan index.html SPA (plugin `pages`) |
| `http://localhost:8080/app/` | SPA loaded |
| `http://localhost:8080/doc.json` | JSON Swagger (opsional) |
| `http://localhost:5173` | Dev frontend, proxy API ke 8080 |

## Permasalahan Ditemukan dalam Repo

| No | Item | Detail | Rekomendasi |
|---|---|---|---|
| P-1 | Dua blok env kontradiktif | Blok atas Postgres+Docker names, blok bawah MySQL+localhost | Bersihkan `.env.example`; pisahkan template dev/prod atau beri komentar jelas per block |
| P-2 | Tidak ada migrasi awal | Folder `migrations/` belum ada | Tentukan skema minimal BBH Data Board sebelum membuat migration; tidak bisa `migrate up` sampai schema disepakati |
| P-3 | Tidak ada Docker Compose | Framework menargetkan container tetapi tidak menyediakan compose file | Pertimbangkan menambahkan `compose.yml` minimalis jika ingin first-run lebih simpel |
| P-4 | `swag` CLI tidak terinstall | `build.sh` akan skip swagger generation | Install swag jika Swagger docs diinginkan; non-blocking |

## Catatan Tambahan
- Plugin `pages` adalah catch-all bawaan framework untuk serve SPA dari `./sub_app/webapp/dist`. Jika folder dist kosong, halaman akan error 5xx pada route SPA. Pastikan build frontend selesai sebelum akses `/app/` via backend.
- CORS default (ketika `CORS_ALLOWED_ORIGINS` kosong) mengizinkan semua origin — cukup untuk development lokal. Saat produksi harus diisi explicit list.
- Swagger endpoint `/swagger/*any` tetap berfungsi tanpa JSON; jika JSON tidak ditemukan muncul pesan `swagger JSON not found`.
- KeyDB/Redis opsional. Ketiadaan host hanya menghasilkan log warning, bukan crash.
- `syscall.Umask(0o002)` pada entrypoint memastikan file baru dibuat group-writable (umask 002). Ini aman untuk development di Linux local.


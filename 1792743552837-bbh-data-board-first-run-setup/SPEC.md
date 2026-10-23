---
title: "First-Run Setup — Analisa & Plan (Docker Compose v2)"
description: "Panduan first-run menggunakan Docker Compose untuk BBH Data Board: app, postgres, pgadmin, keydb, dan swag CLI."
version: 2
status: complete
created: "2026-10-23"
updated: "2026-10-23"
tags:
    - setup
    - first-run
    - donisgo
    - docker-compose
---
# SPEC: First-Run Setup

## Ringkasan
Menyiapkan **BBH Data Board** (backend Go berbasis framework donisgo + SPA React/Vite) agar bisa dijalankan pertama kali secara lokal **menggunakan Docker Compose**, reproducible, dan dapat diverifikasi.

## Keputusan Desain (Disetujui)
| # | Keputusan | Nilai |
|---|---|---|
| D-1 | Orkestrasi | Docker Compose (bukan setup host-native) |
| D-2 | Service | `app`, `postgres`, `pgadmin`, `keydb` |
| D-3 | Port Postgres ke host | `5433:5432` (menghindari konflik dengan postgres lokal di host) |
| D-4 | Migrasi saat first run | **Di-skip** — repo belum punya folder `migrations/`, dan server tidak butuh tabel untuk start |
| D-5 | pgAdmin | Auto-connect ke service `postgres` via `servers.json` |
| D-6 | swag CLI | Tersedia dalam image app dev container untuk generate manual via `docker compose exec app swag ...`. Tidak wajib di-build saat image dibuat. |
| D-7 | Mailpit / SMTP test service | **Tidak disertakan** pada tahap ini |
| D-8 | Database default | PostgreSQL |
| D-9 | Node.js di container `app` | **Tidak disertakan** — image `app` Go-only; deps & Vite build dijalankan di host (repo di-bind mount, `sub_app/webapp/dist` hasil build host langsung terlihat oleh container) |

## Goal
1. Seluruh infrastruktur lokal (database + cache + DB GUI) menyala dengan `docker compose up`. Container `app` berjalan dalam mode **IDLE** (`sleep infinity`) dan TIDAK menjalankan server otomatis — fokusnya development, jadi semua proses (build, test, run) dijalankan manual via `docker compose exec app ...`.
2. Developer dapat menjalankan perintah individual tanpa restart stack: `docker compose exec app go build ...`, `docker compose exec app go test ./...`, `docker compose exec app go run ./cmd/server`.
3. Frontend (deps npm & build Vite) dijalankan **di host** (`cd sub_app/webapp && npm ci && npm run build`) — karena repo di-bind mount, hasil `dist/` langsung terlihat oleh container tanpa Node.js di dalamnya.
4. Developer tetap bisa menjalankan Vite dev server di host (port 5173) untuk workflow hot-reload frontend.

### Kriteria Sukses (Acceptance Criteria)
- [AC-1] `docker compose up` menyalakan 4 service; Postgres dan KeyDB `healthy`; `app` berstatus running dengan command `sleep infinity` (IDLE, tanpa proses server).
- [AC-2] `.env` untuk konteks Docker berisi `DB_HOST=postgres`, `DB_PORT=5432`, `KEYDB_HOST=keydb` — sesuai service name di network compose.
- [AC-3] `postgres` dapat diakses dari host via port `5433`, dan via `postgres:5432` dari dalam network compose.
- [AC-4] `app` berhasil di-build sebagai image development **Go-only (tanpa Node.js)**: toolchain Go + swag CLI; image siap untuk `docker compose exec`; port `8080:8080` terpasang untuk server yang dijalankan manual. Frontend di-build dari host.
- [AC-5] Saat server dijalankan manual via `docker compose exec app go run ./cmd/server`, `http://localhost:8080/` dan `http://localhost:8080/app/` menampilkan SPA; `http://localhost:8080/doc.json` mengembalikan JSON Swagger valid.
- [AC-6] Log app tidak memuat error koneksi database; warning KeyDB boleh muncul bila cache tidak tersedia (non-fatal).
- [AC-7] pgAdmin di `http://localhost:5050` sudah memiliki server `postgres` terdaftar otomatis dan dapat connect tanpa setup manual.
- [AC-8] Swagger JSON dapat di-generate manual via `swag` di dalam container dan tersedia di `docs/swagger/swagger.json`.
- [AC-9] Frontend dev mode tetap dapat dijalankan di host (port 5173) dengan proxy `/api` dan `/admin` mengarah ke `localhost:8080`.
- [AC-10] Tersedia `.dockerignore` sehingga build context tidak membawa `node_modules/`, `.git/`, `.historic/`, dan `bin/`.

## Batasan Scope
- Tidak membuat model, endpoint, akun, atau skema bisnis BBH Data Board yang belum ditentukan.
- Tidak menjalankan migrasi atau membuat migrasi dummy. Startup server saat ini butuh koneksi DB tetapi tidak menjalankan migrasi otomatis; repo belum punya folder `migrations/` dan `console migrate up` akan gagal tanpa target migrasi. Skema/migration dibahas pada Work Order terpisah saat disepakati.
- Docker Compose untuk local development; bukan konfigurasi production-ready. Kredensial default hanya boleh untuk local development.
- Compose hanya menyediakan empat service yang dipilih (`app`, `postgres`, `pgadmin`, `keydb`); Mailpit tidak termasuk.
- `app` adalah development container IDLE (`sleep infinity`), bukan server daemon. Proses Go dijalankan manual via `docker compose exec app <command>`.
- Dockerfile menyediakan toolchain development **Go saja tanpa Node.js** dan tidak perlu membangun binary saat image dibuat; artifact bisa dibangun sesuai kebutuhan melalui exec.
- Frontend (npm/Vite) dijalankan dari host; hasil build `sub_app/webapp/dist` terlihat di container via bind mount. Vite dev server dihost, tidak ditambahkan sebagai service Compose.


## Analisa Proyek
| Aspek | Temuan |
|---|---|
| **Backend** | Go 1.25+ / Gin + GORM / PostgreSQL or MySQL |
| **CLI** | `console` binary — migration, plugin generation, seeding |
| **Frontend** | React 19 + TypeScript + Vite (SPA di `/app/`) |
| **Env** | `.env.example` memuat dua konteks konfigurasi tanpa pemisah yang tegas: bagian PostgreSQL memakai service names (`postgres`, `keydb`), dan bagian MySQL memakai localhost. Untuk Compose, gunakan PostgreSQL + service names. **Tidak ada file `.env`**. |
| **Docker Compose** | Tidak ditemukan; `docker compose` tersedia di host. | Tambahkan Compose untuk empat service. |
| **pgAdmin** | Belum ada konfigurasi. | Tambahkan `servers.json` agar server `postgres` otomatis terdaftar. |
| **Docker build** | Belum ada `Dockerfile` atau `.dockerignore`. | Image menyediakan development toolchain **Go-only** (plus swag CLI, tanpa Node.js); app idle dan perintah dijalankan lewat `exec`. |
| **Migrations** | Folder `migrations/` belum ada sama sekali. | Skip di first run; jangan buat migration palsu. |
| **Swagger** | Belum ada folder `docs/swagger/`, swag CLI belum terinstall. | Sediakan swag di image development dan generate manual saat diperlukan. |
| **Build Output** | `bin/` belum ada. | Artifact dibangun on-demand via `docker compose exec app ...`. |
| **Tools Lokal** | Go 1.27.3 ✓, Node v24 ✓, npm ✓, pnpm ✓, docker compose ✓ |

## Rancangan Docker Compose (Normatif)

Buat file Compose untuk empat service berikut; detail versi image dapat dipin saat implementasi:

| Service | Konfigurasi utama |
|---|---|
| `postgres` | PostgreSQL, environment dari `.env`, named volume untuk persistensi, host port `5433:5432`, healthcheck `pg_isready`. |
| `keydb` | Image KeyDB kompatibel Redis, expose host port `6379` untuk debugging opsional, healthcheck ping jika image mendukung. |
| `pgadmin` | Port `5050:80`, login dari `PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD`, volume data persisten, mount konfigurasi server read-only agar auto-connect ke host `postgres`, port `5432`. |
| `app` | Dibangun dari root repo dengan `Dockerfile` Go-only (tanpa Node.js); host mapping `${APP_PORT:-8080}:8080` (variabel `APP_PORT` agar bebas bentrok port host; port container tetap 8080); env DB menunjuk `postgres:5432` dan KeyDB menunjuk `keydb:6379`; `depends_on` Postgres & KeyDB dengan `condition: service_healthy`; restart `"no"`; **command default `sleep infinity` (IDLE)** — container tidak menjalankan server otomatis. Source repo di-mount sebagai volume. Proses Go dijalankan manual: `docker compose exec app go run ./cmd/server`, `go build ...`, `go test ./...`; build frontend (`npm ci`, `npm run build`) dijalankan di host. |

- Tambahkan `.dockerignore` untuk setidaknya `.git/`, `.historic/`, `node_modules/`, `sub_app/webapp/node_modules/`, `sub_app/webapp/dist/` (akan dibuat saat build), `bin/`, `.env`, dan artefak lokal.
- `.env.example` untuk Compose harus menyediakan konfigurasi DB PostgreSQL konsisten serta variabel pgAdmin (`PGADMIN_DEFAULT_EMAIL`, `PGADMIN_DEFAULT_PASSWORD`). File `.env` lokal tidak boleh di-commit.
- Password default bersifat local-only dan harus diganti sebelum penggunaan bersama/production.
- Development image menyediakan toolchain **Go tanpa Node.js** (sesuai D-9), serta `swag` CLI bila diperlukan. Frontend deps & Vite build dijalankan di host.
- **Pola Development:** `app` adalah container IDLE dengan default command `sleep infinity`. Ia tidak menjalankan Go server atau command build secara otomatis. Semua proses dimulai eksplisit dari host melalui `docker compose exec app <command>`.
- Source repo dimount ke working directory `/src`, sehingga perubahan file langsung terlihat di container.
- Contoh: `docker compose exec app go build ./...`, `docker compose exec app go test ./...`, `docker compose exec app go run ./cmd/server`; frontend dari host: `cd sub_app/webapp && npm ci && npm run build`.
- Untuk hot reload frontend, jalankan Vite dari host pada port 5173 dengan proxy existing ke `localhost:8080`.

## Alur Penggunaan dan Verifikasi

1. Copy `.env.example` menjadi `.env`, lalu isi kredensial lokal (termasuk password pgAdmin).
2. Jalankan stack dengan `docker compose up -d --build`.
3. Periksa `docker compose ps` — Postgres dan KeyDB healthy; `app` running dan idle.
4. Verifikasi pgAdmin di `http://localhost:5050`, login sesuai `.env`, dan server `postgres` sudah terdaftar serta dapat tersambung.
5. Jalankan build atau test secara manual: `docker compose exec app go build ./...` atau `docker compose exec app go test ./...`.
6. Saat ingin menjalankan backend, buka terminal dan jalankan `docker compose exec app go run ./cmd/server`; proses ini berjalan foreground dan dapat dihentikan tanpa menghentikan infra. Port host dipetakan via `APP_PORT` (default 8080; bila port host 8080 terpakai, set `APP_PORT` di `.env`, mis. 8082) ke port container 8080.
7. Saat server berjalan, verifikasi `http://localhost:$APP_PORT/`, `http://localhost:$APP_PORT/app/`, dan `http://localhost:$APP_PORT/doc.json`.
8. Untuk frontend dev server: `cd sub_app/webapp && npm ci && npm run dev -- --host 0.0.0.0`; buka `http://localhost:5173`. Proxy Vite (`/api`, `/admin`) memakai `BACKEND_PORT` (default 8080) — set `BACKEND_PORT=$APP_PORT` bila backend tidak di host 8080.
9. `docker compose down` menghentikan service tanpa menghapus data; `docker compose down -v` menghapus volume dan seluruh data lokal (destruktif, perlu peringatan jelas).

## Temuan Repo dan Penanganan

| No | Item | Penanganan dalam rencana |
|---|---|---|
| P-1 | `.env.example` memiliki konfigurasi PostgreSQL/Docker dan MySQL/host-native dalam satu file | Tetapkan penggunaan Compose PostgreSQL sebagai jalur first-run; perjelas/pisahkan konteks environment agar tidak menggabungkan variabel yang kontradiktif. |
| P-2 | Tidak ada migrasi awal | Skip migrations, jangan membuat dummy schema. Server hanya perlu DB reachable untuk start; schema aplikasi menjadi pekerjaan terpisah. |
| P-3 | Belum ada Compose/Dockerfile | Sediakan orkestrasi empat service dan Dockerfile development Go-only (single-stage, tanpa Node.js) dengan pola IDLE. |
| P-4 | `swag` CLI belum tersedia dan docs swagger tidak ada | Sediakan `swag` di image development; generate docs manual via `docker compose exec app swag ...` saat diperlukan, output ke `./docs/swagger/swagger.json`. |
| P-5 | Belum ada `.dockerignore` | Tambahkan agar build context tidak membawa cache, credentials, dan data Historic. |
| P-6 | Potensi race saat DB startup | Healthcheck Postgres dan `depends_on: service_healthy` untuk app. |

## Catatan Teknis
- Plugin `pages` serve SPA dari `./sub_app/webapp/dist`; Dockerfile wajib menyalin dist ke lokasi yang sama relatif terhadap working directory runtime.
- `PORT=8080` dibaca oleh server; binding container perlu listen pada interface yang bisa dijangkau dari port mapping (bukan hanya loopback).
- pgAdmin auto-registration menggunakan format `servers.json` yang tepat dan hostname Compose `postgres`; secrets tidak boleh ditulis ke file tracked bila memungkinkan.
- KeyDB opsional bagi backend (inisialisasi gagal bersifat non-fatal), tetapi tetap dinyalakan karena termasuk stack yang diminta.
- `/doc.json` mengambil JSON dari `./docs/swagger/swagger.json` atau `./docs/swagger.json`; jalur pertama menjadi target build.
- Database tidak auto-migrate; pengguna first run tidak seharusnya mengasumsikan ada tabel bisnis.
- Compose dan kredensial yang disetujui hanya untuk local development; jangan expose port/services ke jaringan publik tanpa hardening.


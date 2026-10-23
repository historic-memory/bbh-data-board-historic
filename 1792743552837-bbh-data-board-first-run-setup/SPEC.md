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
| D-6 | swag CLI | **Di-install di build stage Dockerfile** agar `docs/swagger/swagger.json` ter-generate |
| D-7 | Mailpit / SMTP test service | **Tidak disertakan** pada tahap ini |
| D-8 | Database default | PostgreSQL |

## Goal
1. Seluruh stack (backend + database + cache + DB GUI) menyala lokal dengan satu perintah `docker compose up`.
2. Backend Go berjalan di dalam container, terhubung ke `postgres` dan `keydb` lewat service name, melayani HTTP di port 8080, serta menyerve SPA hasil build.
3. Developer tetap bisa menjalankan Vite dev server di host (port 5173) untuk workflow hot-reload.

### Kriteria Sukses (Acceptance Criteria)
- [AC-1] `docker compose up` menyalakan 4 service tanpa error; `app` menunggu `postgres` benar-benar siap via healthcheck sehingga tidak crash karena race condition.
- [AC-2] `.env` untuk konteks Docker berisi `DB_HOST=postgres`, `DB_PORT=5432`, `KEYDB_HOST=keydb` — sesuai service name di network compose.
- [AC-3] `postgres` dapat diakses dari host via port `5433`, dan via `postgres:5432` dari dalam network compose.
- [AC-4] `app` berhasil build di dalam Docker (multi-stage: Node untuk frontend + Go untuk binary) dan listen di `:8080`.
- [AC-5] `http://localhost:8080/` mengembalikan `index.html` SPA, dan `http://localhost:8080/app/` menampilkan aplikasi frontend.
- [AC-6] `docker compose logs app` tidak memuat error koneksi database; warning KeyDB boleh muncul bila cache tidak tersedia (non-fatal).
- [AC-7] pgAdmin di `http://localhost:5050` sudah memiliki server `postgres` terdaftar otomatis dan dapat connect tanpa setup manual.
- [AC-8] Build menghasilkan `docs/swagger/swagger.json` sehingga `/doc.json` mengembalikan JSON valid (bukan `swagger JSON not found`).
- [AC-9] Frontend dev mode tetap dapat dijalankan di host (port 5173) dengan proxy `/api` dan `/admin` mengarah ke `localhost:8080`.
- [AC-10] Tersedia `.dockerignore` sehingga build context tidak membawa `node_modules/`, `.git/`, `.historic/`, dan `bin/`.

## Batasan Scope
- Tidak membuat model, endpoint, akun, atau skema bisnis BBH Data Board yang belum ditentukan.
- Tidak menjalankan migrasi atau membuat migrasi dummy. Startup server saat ini butuh koneksi DB tetapi tidak menjalankan migrasi otomatis; repo belum punya folder `migrations/` dan `console migrate up` akan gagal tanpa target migrasi. Skema/migration dibahas pada Work Order terpisah saat disepakati.
- Docker Compose untuk local development; bukan konfigurasi production-ready. Kredensial default hanya boleh untuk local development.
- Compose hanya menyediakan empat service yang dipilih (`app`, `postgres`, `pgadmin`, `keydb`); Mailpit tidak termasuk.
- Build app memakai multi-stage Dockerfile: frontend build dengan Node, backend build dengan Go, Swagger docs dihasilkan dengan `swag` CLI, lalu hasil runtime minimal disiapkan untuk server.
- Vite dev server tetap menjadi opsi host-native untuk hot reload; tidak ditambahkan sebagai service Compose.


## Analisa Proyek
| Aspek | Temuan |
|---|---|
| **Backend** | Go 1.25+ / Gin + GORM / PostgreSQL or MySQL |
| **CLI** | `console` binary — migration, plugin generation, seeding |
| **Frontend** | React 19 + TypeScript + Vite (SPA di `/app/`) |
| **Env** | `.env.example` memuat dua konteks konfigurasi tanpa pemisah yang tegas: bagian PostgreSQL memakai service names (`postgres`, `keydb`), dan bagian MySQL memakai localhost. Untuk Compose, gunakan PostgreSQL + service names. **Tidak ada file `.env`**. |
| **Docker Compose** | Tidak ditemukan; `docker compose` tersedia di host. | Tambahkan Compose untuk empat service. |
| **pgAdmin** | Belum ada konfigurasi. | Tambahkan `servers.json` agar server `postgres` otomatis terdaftar. |
| **Docker build** | Belum ada `Dockerfile` atau `.dockerignore`. | Multi-stage build Go + Node dan pengecualian build context. |
| **Migrations** | Folder `migrations/` belum ada sama sekali. | Skip di first run; jangan buat migration palsu. |
| **Swagger** | Belum ada folder `docs/swagger/`, swag CLI belum terinstall. | Install CLI di build stage dan generate JSON saat image dibangun. |
| **Build Output** | `bin/` belum ada. | Artifact diproduksi di dalam image Compose. |
| **Tools Lokal** | Go 1.27.3 ✓, Node v24 ✓, npm ✓, pnpm ✓, docker compose ✓ |

## Rancangan Docker Compose (Normatif)

Buat file Compose untuk empat service berikut; detail versi image dapat dipin saat implementasi:

| Service | Konfigurasi utama |
|---|---|
| `postgres` | PostgreSQL, environment dari `.env`, named volume untuk persistensi, host port `5433:5432`, healthcheck `pg_isready`. |
| `keydb` | Image KeyDB kompatibel Redis, expose host port `6379` untuk debugging opsional, healthcheck ping jika image mendukung. |
| `pgadmin` | Port `5050:80`, login dari `PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD`, volume data persisten, mount konfigurasi server read-only agar auto-connect ke host `postgres`, port `5432`. |
| `app` | Dibangun dari root repo dengan multi-stage `Dockerfile`; port `8080:8080`; env DB menunjuk `postgres:5432` dan KeyDB menunjuk `keydb:6379`; `depends_on` Postgres dengan `condition: service_healthy`; restart policy yang sesuai development. |

- Tambahkan `.dockerignore` untuk setidaknya `.git/`, `.historic/`, `node_modules/`, `sub_app/webapp/node_modules/`, `sub_app/webapp/dist/` (akan dibuat saat build), `bin/`, `.env`, dan artefak lokal.
- `.env.example` untuk Compose harus menyediakan konfigurasi DB PostgreSQL konsisten serta variabel pgAdmin (`PGADMIN_DEFAULT_EMAIL`, `PGADMIN_DEFAULT_PASSWORD`). File `.env` lokal tidak boleh di-commit.
- Password default bersifat local-only dan harus diganti sebelum penggunaan bersama/production.
- Build stage Go memasang `swag` CLI dan menjalankan generator terhadap `cmd/server/main.go`; runtime image harus membawa `docs/swagger/swagger.json` di path yang diperiksa bootstrap.
- `app` start langsung menjalankan server; tidak memanggil `console migrate up`.
- Untuk hot reload, jalankan Vite dari host pada port 5173 dengan proxy existing ke `localhost:8080`, dan biarkan app container menyala.

## Alur Penggunaan dan Verifikasi

1. Copy `.env.example` menjadi `.env`, lalu isi kredensial lokal (termasuk password pgAdmin).
2. Jalankan seluruh stack dengan `docker compose up --build -d`.
3. Periksa `docker compose ps` — Postgres harus healthy dan app harus running.
4. Verifikasi app: `http://localhost:8080/` dan `http://localhost:8080/app/` menampilkan SPA; `http://localhost:8080/doc.json` mengembalikan JSON Swagger.
5. Verifikasi pgAdmin di `http://localhost:5050`, login sesuai `.env`, dan server `postgres` sudah terdaftar serta dapat tersambung.
6. KeyDB tersedia untuk app di hostname `keydb:6379` pada network Compose.
7. Untuk menjalankan Vite dev server di host: `cd sub_app/webapp && npm ci && npm run dev -- --host 0.0.0.0`; buka `http://localhost:5173`.
8. `docker compose down` menghentikan service tanpa menghapus data; `docker compose down -v` menghapus volume DB/pgAdmin dan seluruh data lokal (destruktif, perlu peringatan jelas).

## Temuan Repo dan Penanganan

| No | Item | Penanganan dalam rencana |
|---|---|---|
| P-1 | `.env.example` memiliki konfigurasi PostgreSQL/Docker dan MySQL/host-native dalam satu file | Tetapkan penggunaan Compose PostgreSQL sebagai jalur first-run; perjelas/pisahkan konteks environment agar tidak menggabungkan variabel yang kontradiktif. |
| P-2 | Tidak ada migrasi awal | Skip migrations, jangan membuat dummy schema. Server hanya perlu DB reachable untuk start; schema aplikasi menjadi pekerjaan terpisah. |
| P-3 | Belum ada Compose/Dockerfile | Sediakan orkestrasi dan multi-stage build empat service. |
| P-4 | `swag` CLI belum tersedia dan docs swagger tidak ada | Install `swag` di build stage dan generate docs di image build; pastikan file runtime berada di `./docs/swagger/swagger.json`. |
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



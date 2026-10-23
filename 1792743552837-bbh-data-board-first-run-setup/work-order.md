---
title: WO-01 — First Run Setup via Docker Compose
description: Menyiapkan development stack BBH Data Board melalui Docker Compose; service app idle dan semua command dijalankan manual via exec.
status: complete
created: "2026-10-23"
updated: "2026-10-23"
tags:
    - work-order
    - docker-compose
    - setup
    - first-run
---
# WO-01 — First Run Setup via Docker Compose

**Referensi SPEC:** `.historic/1792743552837-bbh-data-board-first-run-setup/SPEC.md` (v2)

## Tujuan Work Order
Menyiapkan alur development lokal dengan tepat empat service Docker Compose: `app`, `postgres`, `pgadmin`, dan `keydb`. Service `app` adalah development container yang tetap **IDLE** (`sleep infinity`) setelah start; ia tidak menjalankan server/build/test otomatis. Developer menjalankan command secara eksplisit melalui `docker compose exec app ...`.

**Checkpoint adalah goal:** setiap checkpoint dinyatakan selesai hanya jika semua kriteria penerimaannya terpenuhi dan bukti verifikasi dicatat. Kerjakan berurutan; checkpoint berikutnya bergantung pada checkpoint sebelumnya.

## Batasan
- Tidak mengubah source code aplikasi.
- Tidak membuat atau menjalankan migrasi database. Tidak ada schema bisnis yang disepakati.
- Tidak menambahkan service tambahan, termasuk Mailpit.
- Hanya untuk local development; jangan menggunakan kredensial default di production.
- Postgres dipetakan ke host `5433:5432`; antar-container menggunakan `postgres:5432`.
- `app` menggunakan `DB_HOST=postgres` dan `KEYDB_HOST=keydb`.
- `app` tidak menjalankan proses aplikasi otomatis; default command `sleep infinity`, lifecycle `restart: "no"`, source repo di-mount agar perubahan terlihat langsung.
- Build, test, migration, dan run server dilakukan manual dengan `docker compose exec app ...`; `docker compose exec -d app ...` hanya untuk proses development yang perlu berjalan di background.
- pgAdmin harus punya server `postgres` terdaftar otomatis; prompt password saat koneksi pertama, jangan commit password plaintext dalam `servers.json`.
- Development image menyediakan toolchain **Go saja (tanpa Node.js)** — sesuai keputusan D-9: deps npm & build Vite dijalankan di host karena repo di-bind mount; Swagger CLI tersedia untuk generate docs secara manual bila dibutuhkan.

## Checkpoint Goals

### CP-1 — Environment Compose Siap dan Aman
**Goal:** developer dapat membuat `.env` dari satu template Compose yang jelas; rahasia lokal tidak ikut ke git atau Docker build context.

**Deliverable:** `.env.example` diperjelas untuk jalur Docker Compose dan `.dockerignore` di root.

**Kriteria penerimaan:**
- `.env.example` menyediakan konfigurasi Postgres dan service names Compose (`DB_HOST=postgres`, `DB_PORT=5432`, `KEYDB_HOST=keydb`).
- Template menyediakan variabel pgAdmin (`PGADMIN_DEFAULT_EMAIL`, `PGADMIN_DEFAULT_PASSWORD`) dan port/service settings yang diperlukan.
- Konfigurasi MySQL/host-native yang lama dipisahkan atau diberi label jelas agar tidak terbaca sebagai konfigurasi Compose aktif.
- `.dockerignore` mengecualikan minimal `.git/`, `.historic/`, `.env`, `bin/`, `node_modules/`, `sub_app/webapp/node_modules/`, `sub_app/webapp/dist/`, dan `storage/`.
- `.env` tetap di-ignore oleh git dan tidak dikirim sebagai build context.

**Verifikasi:** buat `.env` lokal, jalankan `docker compose config --quiet`, lalu cek `git check-ignore .env`. Pastikan tidak ada file `.env` yang di-track.

**Bukti:** hasil validasi config sukses, `.env` di-ignore, dan daftar pola `.dockerignore` diperiksa.

### CP-2 — PostgreSQL dan KeyDB Sehat
**Goal:** service database dan cache berjalan, health/readiness terverifikasi, serta database dapat diakses dari host pada port yang disepakati.

**Deliverable:** service `postgres` dan `keydb` di file Compose.

**Kriteria penerimaan:**
- PostgreSQL memakai named volume dan healthcheck `pg_isready`.
- PostgreSQL dipetakan ke host sebagai `5433:5432`, bukan `5432:5432`.
- KeyDB berada di network Compose dan menerima koneksi pada port 6379.
- Service dapat dinyalakan tanpa service `app` atau pgAdmin.
- Uji ulang `docker compose down` lalu start kembali mempertahankan data database.

**Verifikasi:** `docker compose up -d postgres keydb`, `docker compose ps`, `pg_isready`, `keydb-cli ping`, dan pemeriksaan port host 5433.

**Bukti:** Postgres berstatus `healthy`, KeyDB membalas `PONG`, dan host port 5433 dapat diakses.

### CP-3 — pgAdmin Auto-Connect
**Goal:** pgAdmin tersedia di browser dan memiliki konfigurasi server Postgres yang siap digunakan.

**Deliverable:** service `pgadmin` dan `pgadmin/servers.json` yang di-mount read-only.

**Kriteria penerimaan:**
- pgAdmin tersedia di `http://localhost:5050` dengan kredensial dari `.env`.
- Konfigurasi otomatis memakai hostname `postgres` dan port internal `5432`.
- Data pgAdmin menggunakan named volume.
- Password database tidak disimpan plaintext dalam file tracked; pgAdmin meminta password saat koneksi pertama jika diperlukan.

**Verifikasi:** HTTP status `200`/`302` pada port 5050 dan pemeriksaan konfigurasi terdaftar dalam container.

**Bukti:** endpoint pgAdmin merespons dan konfigurasi server menampilkan host `postgres`.

### CP-4 — App Development Image Siap Dipakai
**Goal:** image `app` terbangun sebagai development environment **Go-only (tanpa Node.js, plus swag CLI)** dengan source code ter-mount dan proses utama idle, siap menerima command manual.

**Deliverable:** Dockerfile development Go-only dan service `app` di Compose dengan working directory `/src` serta source mount.

**Kriteria penerimaan:**
- Image menyediakan versi Go yang kompatibel dengan `go.mod` dan swag CLI.
- Image **tidak** menyertakan runtime Node.js/npm; frontend deps & Vite build dijalankan di host (D-9).
- Container `app` memakai working directory `/src` dan repo host di-mount ke `/src`.
- Command default container `sleep infinity`; setelah `docker compose up`, container status `running` namun tidak menjalankan server, build, atau test.
- Restart policy app tidak membuat proses aplikasi jalan otomatis.
- Port `8080:8080` dipetakan untuk server yang nanti dijalankan manual.
- `docker compose exec app go version` dan `swag --version` berhasil; perintah `node`/`npm` tidak tersedia di container (Go-only).
- Frontend (`npm ci`, `npm run build`) dapat dijalankan di host dan hasil `sub_app/webapp/dist` terlihat di container.
- `.env`, `.git`, dan `.historic` tidak tersalin di image/build context.

**Verifikasi:** build image; start app; periksa `docker compose ps`, command PID utama, dan versi toolchain melalui `docker compose exec app ...`.

**Bukti:** image build berhasil; app tetap idle/running; semua toolchain command sukses.

### CP-5 — Infra Siap dan App Bisa Dijalankan Manual
**Goal:** Compose menyalakan service idle beserta dependency sehat; developer dapat menjalankan command aplikasi secara manual dari container.

**Deliverable:** service `app` di Compose dengan dependency dan environment yang sesuai serta dokumentasi command `exec`.

**Kriteria penerimaan:**
- `app` bergantung pada `postgres` dengan `condition: service_healthy`; KeyDB juga menunggu healthcheck bila healthcheck disediakan.
- Port host `8080` diteruskan ke port container `8080`.
- Backend menerima `DB_HOST=postgres`, `DB_PORT=5432`, `KEYDB_HOST=keydb`, `KEYDB_PORT=6379`.
- `docker compose up -d` membuat semua empat service running/healthy tanpa memulai Go server.
- `docker compose exec app go test ./...` dapat dieksekusi manual.
- `docker compose exec app go run ./cmd/server` dapat memulai server secara manual; bila dijalankan foreground, menghentikan command menghentikan server tanpa mematikan container.
- Saat server manual aktif, `GET http://localhost:8080/app/` sukses dan `GET http://localhost:8080/doc.json` mengembalikan JSON Swagger valid jika docs telah digenerate.
- Startup Compose tanpa menjalankan `migrate up` dan tanpa koneksi server awal tidak menghasilkan crash.

**Verifikasi:**
```bash
docker compose up -d --build
docker compose ps                     # empat service berjalan; app idle
docker compose exec app go version
docker compose exec app go test ./...
# Di terminal terpisah, jalankan server manual:
docker compose exec app go run ./cmd/server
# Selama proses berjalan, cek endpoint dari host.
```

**Bukti:** `docker compose ps` menunjukkan app running/idle; Go test command sukses; setelah server dijalankan manual, endpoint HTTP merespons. Catat kegagalan test existing sebagai baseline terpisah.

**Catatan command:** Gunakan lowercase `docker` saat menjalankan command. `docker compose exec` secara default foreground; gunakan opsi detached hanya bila developer memang ingin proses server berjalan di latar belakang.

### CP-6 — Alur Development Manual Dapat Diulang
**Goal:** developer bisa menghidupkan stack, menjalankan build/test/server secara eksplisit di container, dan mengakses service tanpa proses aplikasi otomatis.

**Deliverable:** instruksi development yang diperbarui dalam dokumentasi proyek serta bukti verifikasi acceptance criteria SPEC.

**Kriteria penerimaan:**
- Instruksi menjelaskan copy `.env.example` ke `.env`, start Compose, URL tiap service, command `docker compose exec app ...`, stop, dan troubleshooting.
- `docker compose down` lalu `docker compose up -d --build` dapat dijalankan ulang tanpa menghapus data dan app tetap idle setiap kali start.
- Verifikasi meliputi health Postgres/KeyDB, pgAdmin, Go build/test manual (container), build frontend di host, server start manual dan HTTP response, serta Vite dev server/proxy di host.
- `docker compose down -v` diberi peringatan eksplisit bahwa perintah ini menghapus volume dan data.
- Tidak ada service selain empat service yang disepakati.

**Verifikasi:** ikuti panduan dari kondisi stack berhenti; catat hasil tiap pemeriksaan dalam tabel PASS/FAIL.

**Bukti:** tabel hasil verifikasi seluruh AC SPEC dengan residual warning (jika ada), termasuk bukti app idle setelah start dan command server hanya aktif setelah diminta.

## Bukti Verifikasi (2026-10-23)

Perubahan kunci: **app container Go-only tanpa Node.js** (D-9); frontend deps & Vite build dijalankan di host.

| CP | Hasil | Bukti |
|---|---|---|
| CP-1 | PASS | `.env.example` Compose-first (duplikasi legacy MySQL di-comment paling bawah); `.dockerignore` dibuat; `docker compose config --quiet` → OK; `git check-ignore .env` → `.env`. |
| CP-2 | PASS | `postgres` + `keydb` `healthy`; `keydb-cli ping` → `PONG`; host `5433`/`16379` aktif; uji persistensi: tabel marker bertahan setelah `docker compose down` → `up -d` (1 baris tetap ada), lalu dibersihkan. |
| CP-3 | PASS | `http://localhost:5050` → HTTP `200`; log pgAdmin: `Added 1 Server Group(s) and 1 Server(s)`; `PGADMIN_DEFAULT_EMAIL=admin@bbh.dev` (pgAdmin menolak `dev@localhost`). |
| CP-4 | PASS | Build image sukses (Go 1.27 + swag v1.16.4); `node` **tidak ada** di container; `app` running `sleep infinity` dengan `restart: "no"`; port host `8082 → 8080`; `go version` → go1.27.3. |
| CP-5 | PASS | `go build ./...` OK; `go test ./...` semua paket `ok`; server manual (`docker compose exec -d app go run ./cmd/server`) → `/` 200, `/app/` 200, `/doc.json` 200 (setelah `swag init`); pkill server tanpa mematikan container. |
| CP-6 | PASS | Section "Local development via Docker Compose (IDLE)" ditambahkan ke `README.md` (alur exec manual, build frontend di host, peringatan `down -v`, troubleshooting); stack dapat di-`down`/`up -d --build` ulang tanpa kehilangan data dan app tetap idle. |

### Penyimpangan / Keputusan Tambahan
- **D-9:** image `app` Go-only; `npm ci` & `npm run build` dijalankan di host. `vite.config.ts` proxy memakai `BACKEND_PORT` (default 8080) agar tetap berfungsi bila `APP_PORT` diubah.
- **Port host 8080 terpakai** proses lain di mesin dev → host mapping app dibuat variabel `APP_PORT`; nilai aktif di `.env` lokal: `APP_PORT=8082`. Port container tetap `8080`.
- **Image KeyDB** diganti dari `eqb/keydb` (tidak ada) ke `eqalpha/keydb:latest`.
- **pgAdmin email** default diganti ke `admin@bbh.dev` (validator pgAdmin mewajibkan domain bertitik).
- Dockerfile men-set `git safe.directory` untuk `/src` agar `go build` tidak gagal VCS stamping pada bind mount.
- `docs/swagger/` hasil `swag init` bersifat artefak (di-generate manual), tidak di-commit.

## Pemetaan Checkpoint ke SPEC v2

| SPEC AC | Checkpoint |
|---|---|
| AC-1 — infra sehat + app idle | CP-2, CP-4, CP-5 |
| AC-2 — service names dalam env | CP-1, CP-5 |
| AC-3 — host port 5433 / internal 5432 | CP-2 |
| AC-4 — image dev dan port app tersedia | CP-4 |
| AC-5 — server manual dan HTTP SPA/Swagger | CP-5 |
| AC-6 — tidak ada startup DB error | CP-5 |
| AC-7 — pgAdmin auto-connect di 5050 | CP-3 |
| AC-8 — Swagger JSON tersedia setelah generate | CP-4, CP-5 |
| AC-9 — Vite dev server dan proxy | CP-6 |
| AC-10 — dockerignore | CP-1 |

## Risiko dan Mitigasi

| Risiko | Mitigasi |
|---|---|
| Port 5433, 5050, 6379, atau 8080 sudah digunakan | Periksa port sebelum start; bila perlu ubah host mapping dan catat perubahan di hasil WO. |
| App tidak idle / command otomatis berjalan | Tetapkan `command: ["sleep", "infinity"]`, `restart: "no"`; verifikasi proses dan `docker compose ps`. |
| App start sebelum Postgres siap | Healthcheck + `depends_on: condition: service_healthy`; app sendiri tetap idle. |
| Build swag gagal | Sediakan swag CLI di dev image; generate secara manual dan catat kegagalan bila toolchain tidak tersedia. |
| pgAdmin tidak membaca servers config | Validasi format dan lokasi mount; pastikan file read-only dapat dibaca container. |
| Volume lama menyimpan state berbeda | Dokumentasikan reset volume dengan peringatan destruktif; jangan otomatis menghapus data. |
| Source mount atau working directory salah | Pastikan repo host dimount ke `/src` dan command dijalankan dari root project di container. |

## Definition of Done
- [x] CP-1 sampai CP-6 semuanya PASS dengan bukti yang tercatat.
- [x] Semua SPEC AC terverifikasi atau setiap pengecualian disetujui dan dicatat.
- [x] Container `app` tetap idle setelah `docker compose up`; app server/build/test hanya berjalan setelah perintah `exec` eksplisit.
- [x] Tidak ada source code aplikasi, migrasi, atau schema bisnis yang diubah/dibuat.
- [x] Hanya empat service yang disepakati tersedia.
- [x] Risiko data loss akibat `down -v` dijelaskan dengan jelas.

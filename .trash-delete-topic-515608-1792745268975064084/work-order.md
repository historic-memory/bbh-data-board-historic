---
title: "WO-01 — First Run Setup via Docker Compose"
description: "Implementasi first-run BBH Data Board melalui Docker Compose untuk app, postgres, pgadmin, dan keydb. Setiap checkpoint adalah goal yang harus dibuktikan."
status: draft
created: "2026-10-23"
updated: "2026-10-23"
spec: "1792743552837"
tags:
    - work-order
    - docker-compose
    - setup
    - first-run
---
# WO-01 — First Run Setup via Docker Compose

**Referensi SPEC:** `.historic/1792743552837-bbh-data-board-first-run-setup/SPEC.md` (v2)

## Tujuan Work Order
Menyiapkan alur first-run lokal yang reproducible dengan tepat empat service Docker Compose: `app`, `postgres`, `pgadmin`, dan `keydb`.

**Checkpoint adalah goal:** setiap checkpoint dinyatakan selesai hanya jika semua kriteria penerimaannya terpenuhi dan bukti verifikasi dicatat. Kerjakan berurutan; checkpoint berikutnya bergantung pada checkpoint sebelumnya.

## Batasan
- Tidak mengubah source code aplikasi.
- Tidak membuat atau menjalankan migrasi database. Tidak ada schema bisnis yang disepakati.
- Tidak menambahkan service tambahan, termasuk Mailpit.
- Hanya untuk local development; jangan menggunakan kredensial default di production.
- Postgres dipetakan ke host `5433:5432`; antar-container menggunakan `postgres:5432`.
- `app` menggunakan `DB_HOST=postgres` dan `KEYDB_HOST=keydb`.
- pgAdmin harus punya server `postgres` terdaftar otomatis; prompt password saat koneksi pertama, jangan commit password plaintext dalam `servers.json`.
- Swagger CLI dipasang di build stage dan docs dihasilkan saat image build.

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

### CP-4 — Image App Build dan Memuat Artifact
**Goal:** image backend/frontend dapat dibangun dan berisi seluruh artifact runtime yang diperlukan.

**Deliverable:** Dockerfile multi-stage di root repo.

**Kriteria penerimaan:**
- Frontend dibangun dengan Node dari `sub_app/webapp` dan menghasilkan `dist/index.html`.
- Backend server dan console dibangun dengan Go.
- `swag` CLI dijalankan terhadap `cmd/server/main.go` dan menghasilkan `docs/swagger/swagger.json`.
- Runtime image menyertakan binary server, `sub_app/webapp/dist/`, serta `docs/swagger/swagger.json` pada path yang digunakan aplikasi.
- Build image selesai; `.env`, `.git`, dan `.historic` tidak ada di image.

**Verifikasi:** build image lalu periksa keberadaan binary, SPA index, dan Swagger JSON dengan perintah inspeksi container.

**Bukti:** image build sukses dan seluruh artifact runtime ditemukan.

### CP-5 — App Terhubung ke Service dan Menyerve UI
**Goal:** backend berjalan di container, menunggu database sehat, dan menyajikan SPA serta Swagger JSON.

**Deliverable:** service `app` di Compose dengan dependency dan environment yang sesuai.

**Kriteria penerimaan:**
- `app` bergantung pada `postgres` dengan `condition: service_healthy`.
- Port host `8080` diteruskan ke port container `8080`.
- Backend tersambung ke `postgres:5432` dan KeyDB `keydb:6379`.
- Backend tetap start tanpa menjalankan `migrate up`.
- `GET http://localhost:8080/app/` sukses dan mengembalikan SPA.
- `GET http://localhost:8080/doc.json` mengembalikan JSON valid.
- Log app tidak menunjukkan kegagalan koneksi database atau panic.

**Verifikasi:** `docker compose up -d --build`, `docker compose ps`, HTTP checks untuk `/app/` dan `/doc.json`, serta pemeriksaan logs.

**Bukti:** app berjalan, `/app/` merespons sukses, JSON Swagger valid, dan tidak ada error DB.

### CP-6 — First Run End-to-End Dapat Diulang
**Goal:** pengguna baru dapat menyalakan stack dari kondisi lokal bersih dengan instruksi yang jelas dan hasil konsisten.

**Deliverable:** instruksi first-run yang diperbarui dalam dokumentasi proyek serta bukti verifikasi acceptance criteria SPEC.

**Kriteria penerimaan:**
- Instruksi menjelaskan copy `.env.example` ke `.env`, start stack, URL tiap service, credentials, stop, dan troubleshooting.
- `docker compose down` lalu `docker compose up --build -d` dapat dijalankan ulang tanpa menghapus data.
- Verifikasi meliputi app, Postgres health, pgAdmin, KeyDB, Swagger, dan Vite dev server/proxy di host.
- `docker compose down -v` diberi peringatan eksplisit bahwa perintah ini menghapus volume dan data.
- Tidak ada service selain empat service yang disetujui.

**Verifikasi:** ikuti panduan dari kondisi stack berhenti; catat hasil tiap pemeriksaan dalam tabel PASS/FAIL.

**Bukti:** tabel hasil verifikasi seluruh AC SPEC v2 dengan residual warning (jika ada).

## Pemetaan Checkpoint ke SPEC v2

| SPEC AC | Checkpoint |
|---|---|
| AC-1 — stack start dan readiness | CP-2, CP-5 |
| AC-2 — service names dalam env | CP-1, CP-5 |
| AC-3 — host port 5433 / internal 5432 | CP-2, CP-3 |
| AC-4 — image app dan port 8080 | CP-4, CP-5 |
| AC-5 — SPA di `/` dan `/app/` | CP-4, CP-5 |
| AC-6 — log app tanpa error DB | CP-5 |
| AC-7 — pgAdmin auto-connect di 5050 | CP-3 |
| AC-8 — Swagger JSON tersedia | CP-4, CP-5 |
| AC-9 — Vite dev server dan proxy | CP-6 |
| AC-10 — dockerignore | CP-1 |

## Risiko dan Mitigasi

| Risiko | Mitigasi |
|---|---|
| Port 5433, 5050, 6379, atau 8080 sudah digunakan | Periksa port sebelum start; bila perlu ubah host mapping dan catat perubahan di hasil WO. |
| App start sebelum Postgres siap | Healthcheck + `depends_on: condition: service_healthy`. |
| Build swag gagal | Pin versi CLI dan pastikan dependency tersedia di build stage; CP-4 tidak boleh PASS tanpa JSON valid. |
| pgAdmin tidak membaca servers config | Validasi format dan lokasi mount; pastikan file read-only dapat dibaca container. |
| Volume lama menyimpan state berbeda | Dokumentasikan reset volume dengan peringatan destruktif; jangan otomatis menghapus data. |
| SPA dist salah lokasi runtime | Samakan working directory/path dengan `plugins/pages` dan verifikasi dari dalam container. |

## Definition of Done
- [ ] CP-1 sampai CP-6 semuanya PASS dengan bukti yang tercatat.
- [ ] Semua SPEC AC v2 terverifikasi atau setiap pengecualian disetujui dan dicatat.
- [ ] Tidak ada source code aplikasi, migrasi, atau schema bisnis yang diubah/dibuat.
- [ ] Hanya empat service yang disepakati tersedia.
- [ ] Risiko data loss akibat `down -v` dijelaskan dengan jelas.

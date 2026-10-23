---
title: WO-01 — Auth Plugin (Admin & Member, JWT API + CLI)
description: Implementasi plugin auth — migration UUIDv7, model Go, HTTP API JWT, middleware guard, dan CLI management admin/member, dipecah menjadi checkpoint sebagai goal.
status: draft
created: "2026-10-23"
updated: "2026-10-23"
tags:
    - work-order
    - auth
    - jwt
    - plugin
    - cli
    - http-api
---
# WO-01 — Auth Plugin (Admin & Member, JWT API + CLI)

**Referensi SPEC:** `.historic/1792753542116-bbh-data-board-auth-plugin-admin-schema/SPEC.md` (v2)

## Tujuan Work Order
Mengimplementasikan plugin `auth` sesuai SPEC v2: dua tabel (`admins`, `members`) dengan **UUID v7 sebagai base ID**, HTTP API JWT (login/logout/me/change-password + CRUD admin & member), middleware guard, dan CLI console untuk manajemen admin/member. **Checkpoint adalah goal** — setiap checkpoint selesai hanya jika seluruh kriteria penerimaannya terpenuhi dan bukti verifikasi dicatat.

## Batasan
- Tidak mengubah source code di luar scope plugin `auth`, registrasi plugin di `cmd/`, dan migration plugin.
- Tidak mengaktifkan `AutoMigrate` saat runtime. Perubahan schema hanya via migration SQL eksplisit.
- Tidak mengubah `docker-compose.yml`, `Dockerfile`, atau menambah service.
- ID selalu **UUID v7** dari `internal/uuid` (`NewString()`), disimpan `TEXT`, tanpa default DB.
- Password **selalu** bcrypt; tidak pernah dicetak ke stdout/log/history.
- `change-password` tidak memvalidasi password lama (keputusan D-8).
- Tidak mengimplementasikan refresh-token rotation, blacklist, OAuth/SSO/MFA, registrasi publik, atau reset password via email.
- Jangan pernah mengembalikan `password_hash` di response API.
- Semua proses dijalankan lewat container: `docker compose exec app ...`.

## Prasyarat
- Stack Docker Compose (topik 1792743552837) berjalan: `postgres`, `keydb`, `app` (idle).
- `.env` berisi `AUTH_JWT_SECRET`, `DB_*` menunjuk service `postgres`.
- Migration core sudah bersih (`console migrate list`).

## Checkpoint Goals

### CP-1 — Scaffold Plugin & Registrasi
**Goal:** plugin `auth` ter-scaffold, terdaftar di server dan console, dan terbangun tanpa error.

**Deliverable:** direktori `plugins/auth/` hasil scaffold (`plugin.go`, `handlers/`, `services/`, `migrations/postgres/`, `events.go`); registrasi di `cmd/server/main.go` dan `cmd/console/main.go`.

**Kriteria penerimaan:**
- Scaffold dibuat lewat console di dalam container dengan `docker compose exec app bin/console plugin new --id auth --template crud` setelah memastikan path binary; jika binary belum ada, gunakan `go run ./cmd/console plugin new --id auth --template crud`.
- Plugin `auth` didaftarkan pada `RegisterPlugins` di `cmd/server/main.go` dan `RegisterAdditionalPlugins` di `cmd/console/main.go`.
- Perintah `docker compose exec app go build ./...` sukses setelah registrasi.
- `docker compose exec app bin/console auth --help` (atau `go run ./cmd/console auth --help`) menampilkan group command plugin.

**Verifikasi:**
```bash
docker compose exec app go run ./cmd/console plugin new --id auth --template crud
docker compose exec app go build ./...
docker compose exec app go run ./cmd/console auth --help
```

**Bukti:** daftar direktori `plugins/auth/`, cuplikan registrasi di kedua `main.go`, output `go build` sukses dan `auth --help`.

### CP-2 — Migration UUIDv7 untuk `admins` & `members`
**Goal:** schema database terbentuk lewat migration eksplisit dengan `id TEXT PRIMARY KEY` berisi UUID v7, dan dapat di-rollback.

**Deliverable:** `plugins/auth/migrations/postgres/<ts>_create_auth_tables.up.sql` dan `.down.sql`.

**Kriteria penerimaan:**
- Migration files dibuat di `plugins/auth/migrations/postgres/` dengan pola timestamped `*.up.sql` dan `*.down.sql`; gunakan `console migrate make --plugin auth <name>` jika target plugin sudah terdaftar, lalu isi kedua file (scaffold hanya menulis placeholder).
- `up.sql` membuat `admins` (id TEXT PK, name, email UNIQUE, password_hash, is_root BOOL DEFAULT FALSE, is_active BOOL DEFAULT TRUE, created_at/updated_at/deleted_at TIMESTAMPTZ) dan `members` (id TEXT PK, name, email UNIQUE, password_hash, is_active, timestamps) + index `deleted_at`.
- Tidak ada `gen_random_uuid()` / ekstensi DB / default `id`.
- `down.sql` menghapus `members` lalu `admins`.
- `console migrate list --plugin auth` menampilkan target `auth`; `migrate up --plugin auth` menerapkan; `migrate down --plugin auth` me-rollback satu langkah.
- Tabel terlihat di pgAdmin/psql dengan kolom sesuai SPEC.

**Verifikasi:**
```bash
docker compose exec app go run ./cmd/console migrate make --plugin auth create_auth_tables
docker compose exec app go run ./cmd/console migrate list --plugin auth
docker compose exec app go run ./cmd/console migrate up --plugin auth
docker compose exec app go run ./cmd/console migrate down --plugin auth
docker compose exec app go run ./cmd/console migrate up --plugin auth
```

**Bukti:** status migrasi per langkah, deskripsi tabel (`\d admins`, `\d members`) atau screenshot pgAdmin.

### CP-3 — Model Go, Repository, dan UUIDv7 Hook
**Goal:** model `Admin`/`Member` dengan timestamps GORM dan `id` UUID v7 otomatis terisi pada setiap `Create`.

**Deliverable:** struct model + hook `BeforeCreate` (atau generator terpusat) + fungsi hashing/verifikasi password bcrypt.

**Kriteria penerimaan:**
- Model memetakan tabel `admins`/`members`; field `is_root` hanya pada `Admin`.
- `BeforeCreate` mengisi `id` via `internal/uuid.NewString()` bila kosong.
- Util bcrypt: `HashPassword`, `CheckPassword` (cost default), password minimum 8 karakter divalidasi.
- Soft delete aktif (`gorm.DeletedAt`); email unik dienforce.
- Unit test: hash≠plaintext, verify sukses/gagal, `Create` menghasilkan id format UUID v7.
- `docker compose exec app go test ./plugins/auth/...` sukses.

**Verifikasi:**
```bash
docker compose exec app go build ./...
docker compose exec app go test ./... -run 'Model|Password|UUID'
```

**Bukti:** nama test + hasil `ok`; contoh id yang dihasilkan berformat v7 (nibble versi = `7`).

### CP-4 — HTTP API: Login, Logout, Me, Change Password
**Goal:** admin dan member dapat login dan mengelola sesi sendiri via HTTP API dengan JWT.

**Deliverable:** handler auth + route `POST /api/admin/login`, `POST /api/admin/logout`, `GET /api/admin/me`, `POST /api/admin/change-password`, dan padanan `/api/member/*`.

**Kriteria penerimaan:**
- Login valid → `200 {access_token, expires_in}`; kredensial salah / akun non-aktif / soft-deleted → `401` JSON.
- Token berisi subjek (id) + tipe subjek; `GET /me` mengembalikan profil tanpa `password_hash`.
- `change-password` (self-service) menerima `{password}` tanpa password lama; sukses → `200`; password lama tidak lagi valid setelahnya.
- Logout mengembalikan `200` (stateless) dan tidak error.
- Request tanpa/invalid token pada `/me` → `401`.
- Password/hash tidak pernah muncul di body response maupun log.

**Verifikasi:** contoh request via RESTerm/Bruno/curl ke `http://localhost:${APP_PORT:-8080}` untuk skenario sukses dan gagal.

**Bukti:** contoh response `200`/`401` untuk tiap endpoint, bukti `password_hash` absen, bukti login ulang dengan password lama gagal.

### CP-5 — Middleware Guard & Otorisasi Root
**Goal:** route admin-only terlindungi; hanya `is_root=true` dapat mengelola admin.

**Deliverable:** middleware JWT (`Authorization: Bearer <token>`) + guard privilege, terpasang via `RegisterMiddleware`.

**Kriteria penerimaan:**
- Middleware menolak tanpa token / token invalid / expired dengan `401` JSON.
- Route manajemen admin hanya boleh root; admin non-root → `403`.
- Privilege `is_root` dibaca dari DB (bukan hanya dari claim), sehingga pencabutan langsung berlaku.
- Non-root tidak dapat mengangkat dirinya sendiri jadi root.

**Verifikasi:** uji dengan token root vs token non-root vs tanpa token.

**Bukti:** tabel hasil (200/401/403) per skenario token.

### CP-6 — HTTP API CRUD Admin & Member
**Goal:** operasi CRUD admin/member tersedia via REST sesuai authorization matrix.

**Deliverable:** route `/api/admins` dan `/api/members` (list, create, detail, update, delete) + DTO request/response.

**Kriteria penerimaan:**
- `POST /api/admins` membuat admin (root-only), `id` UUID v7, response tanpa `password_hash`.
- `GET /api/admins` root-only; `GET /api/members` admin terautentikasi.
- `PUT/PATCH` memperbarui field yang dikirim; `is_root` hanya bisa diubah oleh root.
- `DELETE` melakukan soft delete; akun terhapus tidak dapat login.
- Email duplikat → `409`; id tidak ditemukan → `404`; payload invalid → `400`.
- Semua endpoint terdaftar di Swagger (`/doc.json`).

**Verifikasi:** rangkaian request CRUD lengkap + cek `/doc.json`.

**Bukti:** ringkasan status code per operasi + bukti endpoint muncul di `/doc.json`.

### CP-7 — CLI Management Admin & Member
**Goal:** operasional admin/member lengkap tersedia dari console, termasuk reset password tanpa password lama.

**Deliverable:** command group `console auth ...`.

**Kriteria penerimaan:**
- `auth create-admin --name --email [--password] [--root]` → insert + cetak id/email (password via prompt aman bila flag kosong).
- `auth update-admin --id [--name] [--root|--no-root] [--active true|false]` → update field yang dikirim; id tidak ditemukan → error non-zero.
- `auth delete-admin --id` → soft delete; akun tidak bisa login lagi.
- `auth change-password --id --new-password` (atau prompt) → ganti hash **tanpa** password lama.
- Padanan member: `create-member`, `update-member`, `delete-member`, `change-password-member`.
- Semua command exit `0` saat sukses, non-zero saat gagal; tidak mencetak password/hash.
- Admin root pertama dapat dibuat via `create-admin --root`.

**Verifikasi:**
```bash
docker compose exec app bin/console auth create-admin --name "Root" --email "root@bbh.dev" --root
docker compose exec app bin/console auth update-admin --id <uuid> --no-root
docker compose exec app bin/console auth change-password --id <uuid> --new-password "..."
docker compose exec app bin/console auth delete-admin --id <uuid>
```

**Bukti:** output tiap command + verifikasi login gagal setelah delete + bukti tidak ada kebocoran password.

### CP-8 — End-to-End, Docs, dan Handover
**Goal:** alur auth terbukti end-to-end dari CLI sampai API, dan terdokumentasi.

**Deliverable:** bukti verifikasi seluruh AC SPEC + catatan penggunaan (login → token → akses protected route).

**Kriteria penerimaan:**
- Alur penuh: `create-admin --root` → login API → akses `/api/admin/me` → CRUD → change-password → login ulang dengan password baru.
- Semua AC SPEC (AC-1..AC-17) terverifikasi atau penyimpangannya dicatat & disetujui.
- Endpoint auth muncul di Swagger; tidak ada `password_hash` bocor.
- Stack dapat di-`down`/`up` ulang tanpa kehilangan data; migration bersih.
- Tabel PASS/FAIL lengkap dengan residual warning (jika ada).

**Verifikasi:** jalankan skenario end-to-end dari stack berhenti.

**Bukti:** tabel hasil AC PASS/FAIL + transkrip ringkas alur end-to-end.

## Pemetaan Checkpoint ke SPEC
| SPEC AC | Checkpoint |
|---|---|
| AC-1 (schema UUIDv7) | CP-2, CP-3 |
- AC-2, AC-3 (migrate create/up) | CP-2 |
| AC-4..AC-8 (login/logout/me/change-password admin & member) | CP-4 |
| AC-9 (otorisasi root) | CP-5, CP-6 |
| AC-10..AC-14 (CLI admin/member) | CP-7 |
| AC-15 (middleware JWT) | CP-5 |
| AC-16 (target migrasi & rollback) | CP-2 |
| AC-17 (root awal via CLI) | CP-7, CP-8 |

## Risiko & Mitigasi
| Risiko | Mitigasi |
|---|---|
| `id` tidak terisi karena tidak ada default DB | Hook `BeforeCreate` + test yang memastikan id terisi |
| Kebocoran `password_hash` di response | DTO eksplisit + test assert field absen |
| Privilege root dari claim usang | Selalu re-check `is_root` dari DB per request |
| Password tersimpan di shell history | Jalur default prompt (no-echo); hindari cetak nilai |
| Email duplikat / soft-delete bentrok unique | Tangani `409`; kebijakan email reserved setelah soft-delete |
| Migration discovery tidak menemukan `auth` | Pastikan plugin terdaftar di console sebelum `migrate list/up` |

## Definition of Done
- [ ] CP-1 sampai CP-8 semuanya PASS dengan bukti yang tercatat.
- [ ] Semua SPEC AC (AC-1..AC-17) terverifikasi atau pengecualian disetujui & dicatat.
- [ ] Tidak ada `AutoMigrate` runtime; schema murni dari migration SQL.
- [ ] `id` selalu UUID v7 aplikasi (`TEXT` PK), tanpa default DB.
- [ ] `password_hash` tidak pernah keluar via API, CLI, atau log.
- [ ] `change-password` berjalan tanpa meminta password lama.
- [ ] Tidak ada perubahan di luar scope plugin `auth`, registrasi `cmd/`, dan migration plugin.

---
title: "Donisgo Multi-Database: Auth di DB Utama + Core Business PostgreSQL Terisolasi (per Cabang)"
description: "Spesifikasi dukungan multi-database di framework donisgo: plugin auth tetap memakai database utama framework, sedangkan plugin core business (bisa multi-cabang) memakai PostgreSQL terpisah dengan satu database per cabang."
version: 1
status: complete
created: "2026-10-23"
updated: "2026-10-23"
tags:
    - donisgo
    - multi-database
    - plugin
    - business-core
    - postgres
    - branch
    - isolasi
---
# SPEC: Donisgo Multi-Database — Auth di DB Utama + Core Business PostgreSQL Terisolasi per Cabang

## Ringkasan
Framework **donisgo** saat ini hanya mendukung **satu koneksi database** (env `DB_*`) yang dipakai bersama oleh semua plugin. Use case BBH Data Board:
- **Auth** (plugin `auth`, tabel `admins`, `members`, dst.) tetap **memakai database utama framework**.
- **Core business** dibuat sebagai **plugin terisolasi** (mis. `business`) yang memakai **PostgreSQL terpisah** dari DB utama.
- **Banyak cabang** → setiap cabang memiliki **database PostgreSQL sendiri** (bukan schema), sehingga datanya terisolasi penuh antar cabang.

SPEC ini mencakup: (1) abstraksi multi-koneksi di framework, (2) desain plugin `business` dengan DB terpisah + migrasi sendiri, (3) strategi routing koneksi per cabang, dan (4) perubahan infrastruktur (env, compose) yang diperlukan.

## Keputusan Desain (Disetujui)
| # | Keputusan | Nilai |
|---|---|---|
| D-1 | Arsitektur DB | Auth memakai DB utama framework (`default`). Core business memakai PostgreSQL **terpisah** (`business`) |
| D-2 | Pola Cabang | **Satu database terpisah per cabang** (bukan schema bersama) — isolasi penuh data antar cabang |
| D-3 | Plugin Business | Plugin baru dengan ID `business` (diisolasi dari inti framework): route, service, model, migrasi, dan CLI command sendiri |
| D-4 | **Registry Koneksi Tabular** | Metadata koneksi di tabel `db_registry` (tanpa rahasia). Password & kredensial tersimpan terpisah di tabel `db_registry_secrets` (CIPHERTEXT). `db.GetGorm(name)` query `db_registry` → `db_registry_secrets[registry_id]` → dekripsi password → build DSN. Jika tabel belum ada → warning log, fallback ke env `DB_*` (backward compat). Tabel terdefinisi sebagai **satu kesatuan logis**: query JOIN `WHERE db_r.id = db_s.registry_id AND status='active' ORDER BY name`. |
| D-5 | Config Location & CLI Provisioning | Koneksi utama (`default`) tetap konfigurasi via **`.env`** (`DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`). Daftar branch ditambah via `bin/business create-branch --code X`: prompt pilih server tujuan (`--host`/`--port` atau ambil dari env `BUSINESS_DB_*`) → `CREATE DATABASE` di server itu → **INSERT INTO `db_registry`** entri cabang → **otomatis jalankan migrasi** ke database baru (satu perintah, tanpa migrate manual). Admin bisa juga menambah/menonaktifkan cabang via SQL langsung (`UPDATE db_registry SET status='inactive' WHERE ...`). |
| D-6 | Migrasi | `console migrate up --plugin business` **satu kali running** otomatis meng-apply migrasi ke database `business` **dan semua database cabang** (default, tanpa flag tambahan). `--connection <name>` tersedia untuk menyasar koneksi spesifik. Guard isolasi mencegah nyasar DB. |
| D-7 | Migrasi | `console migrate up --plugin business` **satu kali running** otomatis meng-apply migrasi ke database `business` **dan semua database cabang** (default, tanpa flag tambahan). `--connection <name>` tersedia untuk menyasar koneksi spesifik (mis. `--connection default` atau `--connection branch-<code>`) |
| D-8 | ServiceDeps | `ServiceDeps` bertambah: `DB *gorm.DB` (tetap, ke database utama) dan `Connections map[string]*gorm.DB` (seluruh koneksi yang tersedia) agar plugin bisa pilih. Backward compatible |
| D-9 | Runtime | Plugin `business` memilih koneksinya sendiri (bukan menerima `deps.DB`); code path separuh lain (auth) tetap memakai `deps.DB` agar perubahan tidak merusak plugin existing |
| D-10 | Migrasi per Cabang | Satu set file migrasi (`plugins/business/migrations/postgres/*.sql`) di-apply ke **setiap database cabang** dengan tabel tracking migrations **per database cabang** |
| D-11 | Isolasi di Compose | `docker-compose.yml` menambah service Postgres terpisah (mis. `postgres-business`) dengan volume sendiri; env business & cabang di `.env` menunjuk ke service tersebut |
| D-12 | Naming Koneksi & Secret Split | Canonical name = field `name` di `db_registry`. Kredensial dipisah: `db_registry` host/Port/user_name/database_name (metadata boleh baca luas), `db_registry_secrets` berisi `password_enc CIPHERTEXT` (AES-256-GCM envelope via `internal/secrets`), `key_version`, `rotated_at`. App runtime melakukan JOIN untuk resolve password. Tidak ada pooling kredensial antar entri. |
| D-13 | **Guard Isolasi Migrasi** | Setiap plugin **mendeklarasikan koneksi targetnya** (`ConnectionName()`, default `default`). `console migrate` **menolak** menjalankan migrasi plugin ke koneksi yang tidak cocok (mis. `--plugin business --connection default` → error, tidak dieksekusi). Migrasi DB utama (`default`) **tidak akan pernah** menyentuh DB business/cabang, dan sebaliknya. Validasi berdasarkan `role` di tabel `db_registry`: hanya koneksi dengan `role=business` atau `role=branch` yang bisa diakses CLI bisnis; koneksi `role=main` hanya untuk console core. |
| D-14 | **Default Aman & Interaktif Select** | `console migrate up` tanpa flag menyasar koneksi `default`. Auto-iterasi cabang (**`--plugin business`**) membaca daftar dari tabel `db_registry` (`role='branch' AND status='active'`) + entri `role='business'`. Plugin lain ter-scope ke `default`. CLI bisnis mendukung `--select` untuk prompt interaktif memilih koneksi sebelum migrasi. |
| D-15 | **Enkripsi Password (`internal/secrets`)** | Kolom `password_enc` di `db_registry_secrets` menyimpan CIPHERTEXT format: `{ v: 1, alg: "AES-GCM", iv: base64nonce, cipher: base64ciphertext }`. Menggunakan `internal/secrets.EncryptBlob()`/`DecryptBlob()` yang sudah ada. Master key = `APP_KEY` (32 byte decoded dari base64 env var). Kunci TIDAK pernah tersimpan di DB — hanya dari docker secret/env/KMS. `DecryptBlob()` support pass-through untuk data plaintext lama (migrasi otomatis saat read). |
| D-16 | **SSL/TLS ke DB** | Default `sslmode` = `verify-full` (tipe branch/cabang). Connection `default` (read `.env`) tetap menggunakan konfigurasi eksisting (dev boleh `disable`). Untuk compose lokal, CA cert dan client cert mount via volume; di production wajib TLS penuh. CLI create-branch membaca `sslmode` default dari env atau flag `--sslmode`. |
| D-17 | **CLI Secure** | Command `create-branch` TIDAK menerima `--password` sebagai argumen (bisa terekam shell history). Input password via: (a) stdin (`--password-stdin`), (b) prompt interaktif non-echo (`term.ReadPassword`). Semua parameter password dilog sebagai `REDACTED`. GORM logger di-setup dengan parameterized query agar nilai tidak ter-print. |
| D-18 | **Audit Trail** | Tabel `db_registry_audit` mencatat semua CRUD pada registry tanpa menyimpan value sensitif. Field: `action varchar(20)` (create/update/suspend/delete/rotate), `changed_fields text[]` (nama kolom saja), `actor varchar(150)`, `at timestamptz`. Trigger SQL otomatis isi audit pada INSERT/UPDATE/DELETE di `db_registry` dan `db_registry_secrets`. |


## Goal
1. Framework donisgo bisa membuka **lebih dari satu koneksi database** tanpa mengubah cara plugin existing (auth) bekerja.
2. Core business hidup di **plugin terisolasi** (`business`) dengan **PostgreSQL sendiri** dan **migrasi sendiri**.
3. Setiap **cabang** punya **database tersendiri** yang dibuat/disiapkan secara otomatis/teratur.
4. Seluruh konfigurasi dapat di-set via `.env` dan berjalan mulus di Docker Compose lokal.

### Kriteria Sukses (Acceptance Criteria)
- [AC-1] `internal/db` menyediakan `db.GetGorm("business")` yang query entri bernama `"business"` dari tabel `db_registry` (di database utama), resolve password dari `db_registry_secrets` (dekripsi `internal/secrets`), build DSN dari `host/port/user_name/database_name`, dan mengembalikan `*gorm.DB` terhubung; koneksi singleton per nama. `db.GetGorm()`/`db.GetGorm("default")` tetap membaca env `DB_*` langsung (backward compatible). Unit test lulus.
- [AC-2] `ServiceDeps` memuat `Connections map[string]*gorm.DB` berisi setidaknya `default` dan `business` saat keduanya terkonfigurasi; `DB` tetap mengarah ke `default` (auth lanjut bekerja tanpa modifikasi signifikan).
- [AC-3] Plugin `business` terdaftar di `cmd/server/main.go` dan `cmd/console/main.go`; `RegisterServices` meminta koneksi bernama `business` (bukan memakai `deps.DB`).
- [AC-4] Migrasi `plugins/business/migrations/postgres/<timestamp>_create_business_tables.up.sql` (+ `.down.sql`) dibuat via `console migrate make --plugin business`; path migrasi plugin terdeteksi seperti plugin lain.
- [AC-5] `console migrate up --plugin business` (tanpa flag tambahan) dalam **satu running** menerapkan migrasi business ke database `business` **dan setiap database cabang** yang terdaftar di tabel `db_registry` dengan `role='branch'` AND `status='active'`; tabel tracking muncul di **masing-masing** database sasaran.
- [AC-6] Jika salah satu cabang gagal migrasi, cabang lain tetap dilanjutkan (non-blocking) dan hasil akhir melaporkan ringkasan per cabang (sukses/gagal) dengan exit code non-zero bila ada yang gagal. CLI bisnis (`bin/business`) juga mendukung `--select` untuk prompt interaktif memilih koneksi sebelum eksekusi.
- [AC-7] Routing koneksi cabang: `db.GetGorm("cbg-jakarta")` (field `name` di tabel `db_registry`) mengembalikan koneksi ke DB cabang (read DSN dari tabel); koneksi singleton per nama. Daftar semua koneksi tersedia via `db.Connections()` — digunakan CLI bisnis untuk `--select` dan `/api/business/ping`. Hanya entri `status='active'` yang dimasukkan ke pool.
- [AC-8] Server HTTP tetap bisa start hanya dengan DB `default` (tanpa business/branch terkonfigurasi) — business plugin gagal init **non-fatal** dengan log warning yang jelas.
- [AC-9] `docker compose up` menyalakan service `postgres-business` (healthy) dengan volume terpisah; env `BUSINESS_DB_HOST` menunjuk service tersebut.
- [AC-10] Dokumentasi `.env.example` dan `README.md` memuat variabel `DB_*`, `BUSINESS_DB_*`, dan penjelasan tabel `db_registry` (cara tambah/suspend cabang). Tidak ada `BRANCH_DB_LIST` maupun file YAML registry.
- [AC-11] `bin/business create-branch --code <code>`: konek ke server tujuan (ambil dari flag `--host/--port` atau default dari env `BUSINESS_DB_*`) → `CREATE DATABASE bbh_cbg_<code>` → **INSERT INTO `db_registry`** (name, role=branch, host, port, user_name, database_name, sslmode, status=active) + `SaveRegistrySecret()` (password terenkripsi ke `db_registry_secrets`) → **otomatis jalankan migrasi** ke database baru; satu perintah selesai penuh (database + schema).
- [AC-12] **Guard isolasi:** `console migrate up --plugin business --connection default` **ditolak** dengan error jelas dan **tidak mengeksekusi** SQL apa pun ke DB utama; berlaku juga sebaliknya (plugin `auth`/core tidak bisa dipaksa jalan ke `business`/`branch-<code>`).
- [AC-13] `console migrate up` (tanpa `--plugin`/`--connection`) hanya memproses koneksi `default`; log output mencantumkan koneksi target tiap target migrasi sehingga tidak ada kebingungan DB mana yang diubah.
- [AC-14] Saat `--plugin business` dijalankan, migrasi **tidak menyentuh** DB `default`/auth; saat migrasi `default` dijalankan, **tidak menyentuh** DB business/cabang (diverifikasi lewat tabel `migrations` di masing-masing DB yang tetap terpisah).
- [AC-15] **Enkripsi at rest**: kolom `password_enc` pada `db_registry_secrets` berisi envelope AES-GCM (`{"v":1,"alg":"AES-GCM","iv":"...","cipher":"..."}`), bukan plaintext. `SELECT * FROM db_registry` tidak memuat field password apa pun. Enkripsi/dekripsi memakai `internal/secrets` dengan master key `APP_KEY` (32 byte base64). Bila `APP_KEY` tidak diset → error jelas, **bukan panic**, dan server tetap start untuk koneksi `default`.
- [AC-16] **TLS**: entri registry baru memiliki `sslmode` default `verify-full`. Compose lokal mendokumentasikan opsi `require`/`disable` untuk dev; README menegaskan production wajib `verify-full`.
- [AC-17] **CLI aman**: `bin/business` tidak memiliki flag `--password`; password diterima via `--password-stdin` atau prompt `term.ReadPassword` (tanpa echo). Tidak ada password di help text, error message, atau log (`grep -r` pada log = 0 hit).
- [AC-18] **Audit trail**: setiap INSERT/UPDATE/DELETE pada `db_registry`/`db_registry_secrets` menghasilkan baris di `db_registry_audit` dengan `action`, `actor`, `changed_fields` (nama kolom saja, **tanpa nilai**). Data plaintext lama tetap terbaca (pass-through) dan dapat dimigrasi via `RotateSecret()`.

## Batasan Scope
- **Fase 1 (SPEC ini):** Infrastruktur multi-koneksi + plugin `business` + migrasi per-connection + satu database per cabang. Belum termasuk model bisnis detail (belum ditentukan entitas apa pun).
- Tidak mengubah `auth` menjadi database terpisah — auth **tetap** di DB utama.
- Tidak menambah routing data per-cabang di lapisan HTTP (middleware pemilih cabang berdasar `X-Branch-Code`) pada fase ini; akan jadi work order terpisah.
- Migrasi per cabang masih memakai **satu set file migrasi** yang sama (tracking terpisah per DB); tidak membuat file migrasi berbeda per cabang.
- Tidak menangani sharding otomatis, replikasi, atau HA database.
- Koneksi cabang dibuka **lazy**; database cabang harus ada saat dipakai (dibuat via CLI atau provisioning manual).

## Temuan Repo — Kondisi Saat Ini
| Aspek | Temuan |
|---|---|
| **Koneksi tunggal** | `internal/db/db.go` (`GetDB`) dan `internal/db/gorm.go` (`GetGormDB`) memakai env `DB_TYPE, DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME, DB_SSLMODE` via `godotenv.Overload()`; keduanya **singleton** (`sync.Once`) |
| **Service deps** | `internal/plugins/types.go` — `ServiceDeps{DB *gorm.DB; Store storage.Store}`; semua plugin menerima satu DB yang sama di `RegisterAllServices` |
| **Registrasi plugin** | `internal/plugins/registry.go` — `RegisterPlugins`, `RegisterAllServices(db, store)`, `RegisterAllRoutes`; dipanggil dari `internal/app/bootstrap.go` (`attachPlugins`) dan `internal/console/root.go` |
| **Migrasi** | `internal/console/migrate.go` — target `core`/plugin id, flag `--db` hanya untuk memilih jenis DB (postgres/mysql), **bukan** koneksi; semua migrasi jalan ke satu DB |
| **Plugin auth** | `plugins/auth/` memakai `deps.DB` (DB utama) — tidak perlu diubah untuk tetap bekerja |
| **Migrasi auth** | Sudah ada `plugins/auth/migrations/postgres/2026_10_23_113516_create_auth_tables.*` — contoh pola untuk business |
| **Compose** | `docker-compose.yml` punya service `postgres` (POSTGRES_USER/PASSWORD/DB dari `DB_*`) + volume; perlu service kedua untuk business |
| **Env** | `.env` & `.env.example` berisi blok `DB_*` (postgres) dan komentar blok MySQL; belum ada `BUSINESS_DB_*` atau konfigurasi cabang |

## Rancangan Teknis

### 1) Registry Koneksi Tabular (`internal/db/registry.go`)
- **Tabel `db_registry`** (migration SQL di `plugins/business/migrations/postgres/`):

**Tabel metadata** (`db_registry` — boleh baca luas):
```sql
CREATE TABLE IF NOT EXISTS db_registry (
    id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    name           varchar(100) NOT NULL UNIQUE,
    role           varchar(20)  NOT NULL CHECK (role IN ('main','business','branch')),
    host           varchar(255) NOT NULL,
    port           int          NOT NULL DEFAULT 5432,
    user_name      varchar(150) NOT NULL,
    database_name  varchar(150) NOT NULL,
    sslmode        varchar(50)  NOT NULL DEFAULT 'verify-full',
    max_open_conns int          NOT NULL DEFAULT 25,
    max_idle_conns int          NOT NULL DEFAULT 5,
    label          varchar(150),
    status         varchar(20)  NOT NULL DEFAULT 'active' CHECK (status IN ('active','suspended')),
    created_at     timestamptz  NOT NULL DEFAULT now(),
    updated_at     timestamptz  NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_db_registry_role_status ON db_registry(role, status);
```

**Tabel rahasia** (`db_registry_secrets` — akses dibatasi):
```sql
CREATE TABLE IF NOT EXISTS db_registry_secrets (
    registry_id  uuid PRIMARY KEY REFERENCES db_registry(id) ON DELETE CASCADE,
    password_enc text         NOT NULL,      -- AES-GCM envelope via internal/secrets
    key_version  int          NOT NULL DEFAULT 1,
    rotated_at   timestamptz,
    updated_at   timestamptz NOT NULL DEFAULT now()
);
-- REVOKE ALL pada PUBLIC; app hanya bisa akses via role terdefinisikan.
```

**Audit trigger** (auto-record tanpa value sensitif):
```sql
CREATE TABLE IF NOT EXISTS db_registry_audit (
    id             bigserial PRIMARY KEY,
    registry_id    uuid REFERENCES db_registry(id),
    action         varchar(20) NOT NULL CHECK (action IN ('create','update','suspend','delete','rotate')),
    actor          varchar(150) DEFAULT current_user,
    changed_fields text[]
);
```

Trigger `trg_db_registry_audit` otomatis isi tabel audit saat INSERT/UPDATE/DELETE pada kedua tabel.

- **Struct Go**:
  ```go
  type DBRegistryEntry struct {
      Name, Role, Host, Port, UserName, DatabaseName, SSLMode, Label, Status string
      MaxOpenConns, MaxIdleConns int
  }
  type DBRegistrySecret struct { RegistryID uuid; PasswordEnc string; KeyVersion int }
  ```
  Pool melakukan JOIN `db_registry LEFT JOIN db_registry_secrets ON ...`. Jika secrets row belum ada → error jelas `no credentials registered for connection "X"`.
- **Struct Secret Helper**:
  ```go
  func sealPassword(plain string) (string, error) { return secrets.EncryptBlob([]byte(plain)) }
  func openPassword(enc string) (string, error) {
      b, err := secrets.DecryptBlob(enc) // pass-through untuk plaintext lama
      return string(b), err
  }
  ```
  Menggunakan `internal/secrets` yang sudah ada (AES-256-GCM envelope, master key dari `APP_KEY`). **Tidak perlu dependency baru**.

- **Loader** `LoadRegistry(main *gorm.DB) ([]*DBRegistryEntry, error)`:
  - Query `SELECT r.*, s.password_enc FROM db_registry r LEFT JOIN db_registry_secrets s ON s.registry_id=r.id WHERE r.status='active' ORDER BY r.name`.
  - Return slice terurut (name ASC) → determinisme prompt `--select` & iterasi cabang.
  - Jika tabel belum ada → warning log + return empty slice (graceful, bukan fatal).

- **Pool** singleton per nama:
  - `GetGorm(name)` → `default` langsung dari env `DB_*` (backward compat); lainnya JOIN → buka secrets → enkripsi → cache koneksi.
  - `Connections() []string` → semua nama aktif (untuk `--select`).
  - `NamesByRole(role)` → filter `role='branch'`/`'business'` (auto-iterasi migrate).
  - `Invalidate(name)` → tutup & hapus cache (admin update kredensial).

- **DSN builder** dengan masking: `host={host} port={port} user={user_name} password=REDACTED dbname={database_name} sslmode={sslmode}`. Password tidak pernah masuk ke string DSN/log.
- `openGormDBFromEnv()` tetap dipakai untuk `default` (baca `DB_*` seperti biasa).
- Tidak ada dependency `gopkg.in/yaml.v3` — konfigurasi dikelola via tabel.

### 2) ServiceDeps (`internal/plugins/types.go`)
- Tambah field: `Connections map[string]*gorm.DB`.
- `RegisterAllServices(db, store)` dipanggil saat semua koneksi tersedia (default + business + branch lazy-capable). Implementasi mengekspos pemetaan nama → koneksi.

### 3) Plugin `business`
- Scaffold via `console plugin new --id business --template service` (atau pola sama dengan auth).
- Struktur:
  - `RegisterServices(deps)`: `p.bizDB = deps.Connections["business"]`; error non-fatal (warning) bila belum terkonfigurasi.
  - `migrations/postgres/*.sql` — fase pertama berisi tabel placeholder atau menunggu spesifikasi entitas business berikutnya.
  - `ConsoleCommands`: `console business ...` (lihat AC-11 untuk fase berikutnya).
  - `RegisterRoutes`: route dasar `/api/business/ping` untuk verifikasi koneksi DB terpisah.

### 4) Console migrate multi-koneksi
- Perilaku default `console migrate up --plugin business`: **satu kali running** untuk meng-apply migrasi bisnis ke database `business` + **semua** koneksi branch dari tabel `db_registry` (WHERE `role='branch' AND status='active'`). Tanpa flag tambahan.
- Flag `--connection <name>` tersedia untuk menyasar koneksi spesifik saja (`default`, `business`, `cbg-jakarta`); untuk plugin selain `business` perilaku tetap seperti sekarang (backward compatible, hanya ke koneksi default kecuali flag di-set).
- Eksekusi antar database cabang **tidak atomik** (tidak bisa transaksi lintas DB): migrasi per cabang independen, kegagalan satu cabang tidak menghentikan cabang lain, hasil dirangkum.
- **Guard isolasi (D-13):** migrasi divalidasi terhadap `ConnectionName()` plugin + `role` di tabel `db_registry` — ketidakkocokan → **error, tidak eksekusi**. `console migrate up` polos hanya menyasar `default` (D-14). Setiap target migrasi men-log koneksi targetnya agar jelas DB mana yang tersentuh.
- Tracking tabel `migrations`/`migration_targets` tetap dibuat **di masing-masing database sasaran**.
- CLI bisnis (`bin/business`) juga mendukung `--select` — prompt interaktif menampilkan semua koneksi dari tabel `db_registry` (status='active'); user memilih nomor target sebelum eksekusi.

### 5) Infrastruktur (compose, env)
- `.env` menyimpan kredensial nyata via `DB_*` (default/main app), `BUSINESS_DB_USER/PASSWORD` (opsional, digunakan saat CLI create-branch). Tidak ada file YAML registry — semua koneksi dikelola via tabel `db_registry`.
- `docker-compose.yml`:
  - Service `postgres-business` (image sama, volume berbeda, healthcheck `pg_isready`, expose host port mis. `5434:5432`).
  - `app` `depends_on` menambah `postgres-business` (condition healthy).

## Alur Penggunaan yang Diharapkan
1. Set `.env` → `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` (seperti biasa).
2. `docker compose up -d --build` → service `postgres-business` healthy.
3. `docker compose exec app console migrate up --plugin business` → **satu running** meng-apply migrasi bisnis ke DB business (yang sudah terdaftar di `db_registry`) + semua cabang aktif.
4. Saat ada cabang baru: jalankan `bin/business create-branch --code X --label "Z"` → prompt pilih server tujuan → buat DB → INSERT ke `db_registry` → **auto migrate**. Atau tambah manual via SQL langsung (`INSERT INTO db_registry VALUES (...)`). Lalu `console migrate up --plugin business` — skema cabang baru ikut kena update.
5. `docker compose exec app go run ./cmd/server` → server start; auth tetap di DB utama; business plugin terhubung ke DB business.
6. `bin/business ping --select` → prompt interaktif pilih target dari tabel `db_registry` (daftar active connections) sebelum operasi berikutnya.
7. Admin panel bisa pakai SQL biasa untuk manage cabang: `UPDATE db_registry SET status='suspended' WHERE name='cbg-old';` — instan tanpa restart server.

## Risiko & Catatan
- **Kompleksitas multi-koneksi per cabang** bisa tumbuh: pertimbangkan batas maksimum cabang dan siklus hidup koneksi (idle/close).
- **Kredensial per cabang** disimpan **terenkripsi** di `db_registry_secrets` (AES-256-GCM via `internal/secrets`, master key `APP_KEY`). `APP_KEY` wajib disimpan di docker secret/KMS dan **tidak boleh** ikut dalam dump database — bila hilang, seluruh kredensial cabang tidak dapat didekripsi.
- Migrasi multi-cabang butuh idempotensi & tracking per DB — gunakan tabel migrations yang sudah ada di `internal/console/migrate.go`.
- Perubahan `ServiceDeps` harus tetap backward compatible agar plugin lain tidak rusak.
- Jangan push ke remote; commit lokal saja saat implementasi.
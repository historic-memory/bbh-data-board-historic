---
title: Auth Plugin — Admin & Member Schema (v2)
description: Spesifikasi plugin auth untuk JWT authentication + HTTP API (login/refresh/me) + CLI manajemen admin/member dengan tabel admins dan members.
status: complete
created: "2026-10-23"
updated: "2026-10-23"
tags:
    - auth
    - jwt
    - admin
    - member
    - plugin
    - http-api
---
# SPEC: Auth Plugin — Admin & Member Schema (v2)

## Ringkasan
Membangun plugin `auth` yang menyediakan **JWT authentication** + **HTTP API** (login, logout, me, change-password, CRUD admin/member) + **CLI console** untuk mengelola tabel `admins` dan `members`. Admin memiliki flag `is_root` (boolean). Seluruh kredensial di-hash dengan bcrypt. Migrasi database dilakukan secara eksplisit lewat sistem migration bawaan framework (`console migrate`). Plugin dibuat via scaffolding `console plugin new --id auth --template crud`.

## Keputusan Desain (Disetujui)
| # | Keputusan | Nilai |
|---|---|---|
| D-1 | Plugin ID | `auth` — terdaftar di `cmd/server/main.go` dan `cmd/console/main.go` via plugins framework |
| D-2 | Tabel | `admins`, `members` — SQL migrations milik plugin, tracked oleh sistem migrasi framework; runtime tidak menjalankan AutoMigrate |
| D-3 | Auth Token | JWT HS256 (`internal/auth`): access token (+ claims: sub, level), refresh token opaque (hash disimpan di DB, opsional di fase 1). Secret dari env `AUTH_JWT_SECRET` |
| D-4 | Password Hash | Bcrypt (`golang.org/x/crypto/bcrypt`), cost 10 (default Go) |
| D-5 | HTTP API Routes | POST `/api/admin/login`, POST `/api/admin/logout`, GET `/api/admin/me`, POST `/api/admin/change-password`; POST `/api/member/login`, POST `/api/member/logout`, GET `/api/member/me`, POST `/api/member/change-password` |
| D-6 | HTTP API CRUD Admin/Member | CRUD admin: hanya root; CRUD member: authenticated admin; route di `/api/admins` dan `/api/members` dengan REST methods |
| D-7 | Flag Root | Boolean `is_root` pada tabel `admins`. Admin dengan `is_root=true` punya hak administratif penuh; `is_root=false` tidak dapat mengelola admin atau privilege root |
| D-8 | Change Password | CLI admin: target ID + new password (tanpa old password); HTTP: admin yang sudah login boleh mengganti password akun sendiri dengan new password |
| D-9 | Migration | Eksplisit menggunakan `plugins/auth/migrations/postgres/*.up.sql` dan `*.down.sql`; gunakan `console migrate make --plugin auth <name>` setelah plugin terdaftar, lalu `console migrate up --plugin auth`; tidak ada AutoMigrate saat runtime |
| D-10 | ID Utama | UUID v7 sebagai base ID semua tabel (`admins`, `members`) — digenerate di aplikasi via `internal/uuid/uuidv7.go` (`NewString()`), kolom disimpan sebagai `TEXT` PK (portabel lintas DB, tidak bergantung `gen_random_uuid()`/ekstensi). Setiap `INSERT` wajib mengisi `id` eksplisit |
| D-11 | Member Management | Ya, sama seperti admin: CLI commands dan HTTP API (admin/root saja untuk CRUD) |
| D-12 | Middleware | `auth_required` middleware untuk guarded routes (`/api/admins/*`, `/api/members/*`, `/api/admin/me`, dst.). Token dikirim via header `Authorization: Bearer <token>` |
| D-13 | Swagger | Endpoint login tidak require auth; endpoint lainnya terdokumentasi di swagger via security schema `bearer` |

## Goal
1. Dua tabel database (`admins`, `members`) dibuat melalui migration SQL eksplisit.
2. HTTP API lengkap untuk login, logout, me, change-password, dan CRUD admin/member.
3. CLI console command untuk seluruh operasi administratif (CRUD + change-password tanpa verifikasi lama).
4. Middleware JWT guard aktif pada route protected.
5. Semua endpoint terdokumentasi di Swagger (doc.json).

### Kriteria Sukses (Acceptance Criteria)
- [AC-1] Migrasi SQL menghasilkan tabel `admins` dan `members` dengan skema berikut (kolom `id` bertipe `TEXT` PK berisi string UUID v7 yang digenerate aplikasi):

**admins:**
- `id` TEXT PRIMARY KEY (UUID v7, application-generated)
- `name` VARCHAR(128) NOT NULL
- `email` VARCHAR(255) UNIQUE NOT NULL
- `password_hash` VARCHAR(255) NOT NULL
- `is_root` BOOLEAN NOT NULL DEFAULT FALSE
- `is_active` BOOLEAN NOT NULL DEFAULT TRUE
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT NOW()
- `updated_at` TIMESTAMPTZ NOT NULL DEFAULT NOW()
- `deleted_at` TIMESTAMPTZ NULL

**members:**
- `id` TEXT PRIMARY KEY (UUID v7, application-generated)
- `name` VARCHAR(128) NOT NULL
- `email` VARCHAR(255) UNIQUE NOT NULL
- `password_hash` VARCHAR(255) NOT NULL
- `is_active` BOOLEAN NOT NULL DEFAULT TRUE
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT NOW()
- `updated_at` TIMESTAMPTZ NOT NULL DEFAULT NOW()
- `deleted_at` TIMESTAMPTZ NULL

- [AC-2] Setelah plugin `auth` diregistrasikan di console, `console migrate make --plugin auth create_auth_tables` membuat pasangan file UP/DOWN pada direktori migration plugin untuk diisi SQL schema.
- [AC-3] `console migrate up --plugin auth` berhasil menjalankan migration — tabel ada di pgAdmin/psql setelahnya.
- [AC-4] POST `/api/admin/login` dengan body `{email, password}` → sukses return `{access_token, expires_in}`, gagal return 401 JSON error.
- [AC-5] POST `/api/admin/logout` invalidasi session (jika menggunakan refresh/blacklist) atau stateless return OK.
- [AC-6] GET `/api/admin/me` dengan valid Bearer token → return profil admin saat ini; 401 jika token invalid/expired.
- [AC-7] POST `/api/admin/change-password` dengan body `{password}` → mengganti hash admin yang sedang login; 200 OK.
- [AC-8] POST `/api/member/login` bekerja sama seperti admin login — return token member.
- [AC-9] GET `/api/admins` hanya bisa diakses oleh admin `is_root=true`; CRUD member hanya oleh admin terautentikasi; request tanpa privilege cukup mendapat 403.
- [AC-10] CLI command `console auth create-admin --name "..." --email "..." --password "..." [--root]` berhasil insert admin + output id/email.
- [AC-11] CLI command `console auth update-admin --id <uuid> [--name "..."] [--root] [--no-root] [--active true\|false]` berhasil update.
- [AC-12] CLI command `console auth delete-admin --id <uuid>` soft-delete admin.
- [AC-13] CLI command `console auth change-password --id <uuid> --new-password "..."` ganti hash tanpa verifikasi lama.
- [AC-14] CLI command `console auth create-member`, `update-member`, `delete-member`, dan `change-password-member` tersedia; password change cukup `--new-password`, tanpa password lama.
- [AC-15] Middleware JWT berhasil memvalidasi token dari header `Authorization: Bearer ...` dan reject tanpa token/atau expired dengan 401.
- [AC-16] Migrasi `auth` terdaftar sebagai target `auth`; `console migrate up --plugin auth` berhasil menerapkan schema, `console migrate down --plugin auth` dapat me-rollback migration terakhir.
- [AC-17] Minimal satu admin root dapat dibuat melalui CLI command `create-admin --root`; seeding credential default bukan bagian scope.

## Batasan Scope
- Login admin dan member, logout stateless, profil `/me`, change-password, CRUD admins/members sesuai authorization matrix.
- Refresh-token rotation/revocation, token blacklist/session storage di luar scope awal; logout tidak mencabut access JWT yang sudah diterbitkan.
- Tidak mengimplementasikan OAuth, SSO, MFA, social login, registrasi publik, atau reset-password via email.
- Tidak menjalankan migrasi otomatis saat server boot; perubahan schema hanya melalui explicit migration commands.
- Tidak mengubah Dockerfile, docker-compose, atau menambah service.
- Email divalidasi format dasar; tidak ada MX/domain verification.
- CLI `change-password` menetapkan password baru berdasarkan admin target ID dan tidak meminta password lama; hash lama tidak pernah ditampilkan.

## Analisa Repo Existing
| Aspek | Temuan | Implikasi |
|---|---|---|
| `internal/auth` | Sudah ada JWT signing/parsing HS256, `AccessClaims{AdminID, Level}`, opaque refresh token utilities, dan secret dari `AUTH_JWT_SECRET`. | Perlu keputusan implementasi agar claims dapat mengidentifikasi subject type (`admin`/`member`) dan `is_root`; refresh token rotation tetap di luar scope. |
| `internal/plugins.Plugin` | Interface standar: `ID()`, `RegisterServices(ServiceDeps)`, `RegisterMiddleware()`, `RegisterRoutes(...)`, `Seed()`, `ConsoleCommands()`. | Plugin `auth` mengimplementasikan seluruh hook; hanya migration/CLI/API, bukan auto-migrate di `RegisterServices`. |
| `cmd/server/main.go` | App memakai `RegisterPlugins` hook. | Daftarkan plugin `auth` agar routes dan middleware aktif. |
| `cmd/console/main.go` | Register plugins melalui `RegisterAdditionalPlugins`. | Daftarkan plugin `auth` agar command plugin tersedia. |
| `internal/console/migrate.go` | Migration SQL dibaca per target dari `plugins/<id>/migrations/<db>/`; plugin harus terdaftar. | Tambahkan migration SQL di `plugins/auth/migrations/postgres/`, jalankan `console migrate up --plugin auth`; buat migration lewat `console migrate make --plugin auth <name>`. |
| `internal/console/plugin.go` | `console plugin new --id auth --template crud` scaffold directory handlers, services, migrations/postgres dan file migration awal. | Gunakan scaffold sebagai awal, lalu implementasikan auth sesuai SPEC. |
| `.env.example` | Sudah ada `AUTH_JWT_SECRET`. | Secret tersedia; perlu verifikasi keamanannya sebelum produksi, tanpa menambah layanan infra. |

## Rancangan Skema Database

### Table `admins`
```sql
CREATE TABLE admins (
    id            TEXT PRIMARY KEY,              -- UUID v7 (application-generated)
    name          VARCHAR(128) NOT NULL,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,         -- bcrypt hashed
    is_root       BOOLEAN NOT NULL DEFAULT FALSE,
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ NULL
);
CREATE INDEX idx_admins_deleted_at ON admins(deleted_at);
```

### Table `members`
```sql
CREATE TABLE members (
    id            TEXT PRIMARY KEY,              -- UUID v7 (application-generated)
    name          VARCHAR(128) NOT NULL,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,         -- bcrypt hashed
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ NULL
);
CREATE INDEX idx_members_deleted_at ON members(deleted_at);
```

## Interaksi Plugin dengan Framework
1. Buat scaffolding dengan `docker compose exec app bin/console plugin new --id auth --template crud`.
2. Plugin mengimplementasikan routes, middleware, migrations, dan `ConsoleCommands`; `RegisterServices` hanya menyiapkan dependency runtime, tanpa membuat schema.
3. Daftarkan plugin pada `cmd/server/main.go` dan `cmd/console/main.go`; migration discovery bergantung pada plugin terdaftar.
4. Buat SQL migration untuk PostgreSQL di `plugins/auth/migrations/postgres/`; contoh command scaffold: `docker compose exec app bin/console migrate make --plugin auth create_auth_tables`.
5. Terapkan dengan `docker compose exec app bin/console migrate up --plugin auth`; rollback migration terakhir memakai `migrate down --plugin auth`.
6. API public login di `/api/admin/login` dan `/api/member/login`; seluruh route terlindungi memakai Bearer JWT middleware.
7. CLI manajemen berada di command group `console auth ...` dan meminta kredensial password lewat prompt aman atau flag yang tidak dicatat shell history (implementasi harus menghindari plaintext pada output/log).

## Temuan & Risiko
| No | Item | Penanganan |
|---|---|---|
| P-1 | `AccessClaims` saat ini memiliki `AdminID` + `Level`, bukan subject type dan `is_root`. | Tentukan klaim backward-compatible/versi token yang dapat memisahkan admin dan member dan menyertakan privilege root; jangan mempercayai klaim privilege lama tanpa memvalidasi akun aktif di DB. |
| P-2 | Soft-delete + unique email mencegah reuse alamat email. | Tetapkan kebijakan: email tetap reserved setelah soft-delete (unique global) kecuali SPEC lanjutan menyetujui restore/reuse. |
| P-3 | Logout stateless tidak membatalkan access JWT. | Jelaskan response logout hanya menutup sisi klien; token valid sampai expiry. Revocation/refresh flow out of scope. |
| P-4 | Admin root pertama dibutuhkan untuk bootstrap. | Buat root awal eksplisit lewat console command `create-admin --root`; jangan menyediakan password default/seed credential. |
| P-5 | Detail authorization user/non-root untuk CRUD admin. | Hanya root boleh membuat/mengubah/menghapus admin atau mengubah `is_root`; non-root tidak boleh mengangkat dirinya sendiri. |
| P-6 | Flag password di CLI dapat tersimpan pada shell history/process list. | Sediakan prompt password interaktif (stdin no-echo) sebagai jalur default; jangan pernah mencetak password/hash. |
| P-7 | GORM tidak otomatis mengisi UUID v7 kecuali dikonfigurasi. | Set `id` secara eksplisit sebelum `Create()` (mis. hook `BeforeCreate` pada model atau generator service yang memanggil `internal/uuid.NewString()`); migration SQL tidak menyediakan default `id`. |

## Catatan Teknis
- **ID base = UUID v7** (`internal/uuid/uuidv7.go`, `NewString()`): time-ordered sehingga ramah index/B-tree. Disimpan sebagai `TEXT` PK agar portabel PostgreSQL/MySQL tanpa ekstensi (`gen_random_uuid()` tidak dipakai).
- Password minimum 8 karakter; validasi sebelum hashing dengan bcrypt default cost.
- `change-password` (CLI dan HTTP self-service) hanya menerima `new_password`, tanpa meminta password lama; response/output tidak mengembalikan hash.
- Root privilege bersumber dari field DB `is_root`; claims JWT tidak boleh menjadi satu-satunya sumber authorization karena privilege dapat dicabut.
- DTO API jangan pernah mengeluarkan `password_hash`; password tidak boleh dikirim kembali setelah create/change.
- JWT signing/validation memakai utilitas `internal/auth` yang diperluas bila diperlukan; jangan fork/duplikasi algoritma.
- Migration `down` hanya menghapus tabel `members` dan `admins`; pastikan urutan FK jika kelak ditambahkan (belum ada FK pada schema ini).


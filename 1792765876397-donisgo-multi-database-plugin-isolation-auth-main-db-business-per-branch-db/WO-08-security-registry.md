---
title: 'WO-08: Keamanan Registry — Enkripsi, TLS, Audit, CLI Aman'
description: Pisah tabel secrets, enkripsi AES-GCM via internal/secrets (APP_KEY), sslmode verify-full, audit trigger, CLI tanpa --password (pakai --password-stdin / term.ReadPassword), masking log.
status: complete
created: "2026-10-26"
updated: "2026-10-26"
revision: 2
---
# WO-08: Keamanan Registry — Enkripsi, TLS, Audit, CLI Aman

> **REVISI 2 (2026-10-26)** — Kontrak DBMS (SPEC D-19): rahasia disimpan di `dbms_secrets` (per **server**), bukan `db_registry_secrets` (per koneksi). Satu password server di-share oleh semua cabang yang mengacu `dbms_key` yang sama. Implementasi koreksi di **WO-09**.

## Goal
Menutup celah utama registry tabular: password plaintext di tabel, koneksi DB tanpa TLS, password bocor lewat log/shell history, dan tidak ada jejak audit perubahan kredensial.
**Reuse `internal/secrets`** (AES-256-GCM envelope, `APP_KEY`) — tidak perlu dependency baru.

## Ancaman yang Ditutup
| ID | Ancaman | Mitigasi |
|----|---------|----------|
| T1 | `SELECT *` / dump backup bocor → semua password cabang terbuka | Enkripsi `password_enc` + tabel secrets terpisah |
| T2 | `sslmode=disable` → sniffing jaringan | Default `verify-full` untuk branch, CA cert mount |
| T3 | Password di log aplikasi/GORM | Parameterized queries + log `REDACTED` |
| T4 | `--password` di CLI → shell history / ps | `--password-stdin` + `term.ReadPassword` |
| T5 | App role baca semua baris registry | `REVOKE ALL` di `dbms_secrets` |
| T6 | Tidak ada jejak siapa ubah kredensial | Tabel `db_registry_audit` + trigger |
| T7 | APP_KEY di tempat sama dengan dump | Key hanya dari docker secret/env, tidak di DB |

## Task

### 1. Migration: split tables + audit
File: `plugins/business/migrations/postgres/<timestamp>_dbms_secrets.up.sql` (+ `.down.sql`).

```sql
-- Server DBMS (metadata, tanpa rahasia) — didaftarkan sekali, acuan key
CREATE TABLE IF NOT EXISTS dbms (
    id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    key            varchar(100) NOT NULL UNIQUE,
    driver         varchar(50)  NOT NULL DEFAULT 'postgres',
    host           varchar(255) NOT NULL,
    port           int          NOT NULL DEFAULT 5432,
    user_name      varchar(150) NOT NULL,
    sslmode        varchar(50)  NOT NULL DEFAULT 'verify-full',
    max_open_conns int          NOT NULL DEFAULT 25,
    max_idle_conns int          NOT NULL DEFAULT 5,
    label          varchar(150),
    status         varchar(20)  NOT NULL DEFAULT 'active' CHECK (status IN ('active','suspended')),
    created_at     timestamptz  NOT NULL DEFAULT now(),
    updated_at     timestamptz  NOT NULL DEFAULT now()
);

-- Rahasia server (ciphertext saja)
CREATE TABLE IF NOT EXISTS dbms_secrets (
    dbms_id      uuid PRIMARY KEY REFERENCES dbms(id) ON DELETE CASCADE,
    password_enc text NOT NULL,
    key_version  int  NOT NULL DEFAULT 1,
    rotated_at   timestamptz,
    updated_at   timestamptz NOT NULL DEFAULT now()
);
REVOKE ALL ON dbms_secrets FROM PUBLIC;

-- Koneksi logis (TANPA kredensial — hanya dbms_key + database_name)
CREATE TABLE IF NOT EXISTS db_registry (
    id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    name          varchar(100) NOT NULL UNIQUE,
    role          varchar(20)  NOT NULL CHECK (role IN ('main','business','branch')),
    dbms_key      varchar(100) NOT NULL REFERENCES dbms(key) ON UPDATE CASCADE,
    database_name varchar(150) NOT NULL,
    label         varchar(150),
    status        varchar(20)  NOT NULL DEFAULT 'active' CHECK (status IN ('active','suspended')),
    created_at    timestamptz  NOT NULL DEFAULT now(),
    updated_at    timestamptz  NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_db_registry_role_status ON db_registry(role, status);

-- Audit (tanpa value sensitif)
CREATE TABLE IF NOT EXISTS db_registry_audit (
    id             bigserial PRIMARY KEY,
    registry_id    uuid,
    name           varchar(100),
    action         varchar(20) NOT NULL CHECK (action IN ('create','update','suspend','delete','rotate')),
    actor          varchar(150) DEFAULT current_user,
    changed_fields text[],
    at             timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_db_registry_audit_at ON db_registry_audit(at DESC);
```

Trigger `trg_db_registry_audit` (`AFTER INSERT OR UPDATE OR DELETE`) pada `db_registry`, `dbms`, dan `dbms_secrets` — isi `action` + `changed_fields` (nama kolom saja, **tidak pernah nilai**).

### 2. Helper enkripsi (`internal/db/secret.go`)
```go
func sealPassword(plain string) (string, error) {
    if plain == "" { return "", nil }
    return secrets.EncryptBlob([]byte(plain))
}
func openPassword(enc string) (string, error) {
    if enc == "" { return "", nil }
    b, err := secrets.DecryptBlob(enc) // pass-through plaintext lama (migrasi)
    return string(b), err
}
```
- `secrets.EncryptBlob` menghasilkan envelope `{v,alg,iv,cipher}` — aman disimpan sebagai `text`.
- `secrets.DecryptBlob` pass-through bila input bukan envelope → mendukung baris lama (plaintext) saat migrasi bertahap.
- Master key `APP_KEY` (32 byte base64). Bila tidak diset: `LoadRegistryEntry` mengembalikan error jelas `APP_KEY is not set; cannot decrypt db credentials` (bukan panic).

### 3. Refactor `internal/db/registry.go`
- `LoadRegistry`/`LoadRegistryEntry` query JOIN `db_registry r JOIN dbms d ON d.key = r.dbms_key LEFT JOIN dbms_secrets s ON s.dbms_id = d.id`.
- Struct entry **tidak** memuat password plaintext di field publik; simpan `PasswordEnc` pada entry **dbms** dan resolve saat build DSN via `openPassword`.
- Tambah `SaveDBMSSecret(main *gorm.DB, key, plainPassword string, keyVersion int) error` — enkripsi lalu UPSERT ke `dbms_secrets` (dipakai CLI `dbms add` & rotate).
- `RotateSecret(main *gorm.DB, key string) error` — dekripsi dengan key lama (via `key_version`) → enkripsi dengan key aktif → update `key_version`, `rotated_at`. Rotasi **satu baris `dbms_secrets`** berlaku untuk semua cabang yang mengacu `dbms_key` yang sama.

### 4. TLS
- `sslmode` default kolom = `verify-full`; compose lokal boleh override ke `require`/`disable` untuk dev.
- Compose: mount `./certs/ca.crt:/etc/ssl/certs/db-ca.crt:ro` ke service `app`; Postgres service diberi `ssl=on` + cert (opsional untuk dev).
- Dokumentasikan di README: production **wajib** `verify-full`.

### 5. CLI aman (`internal/businesscli`)
- **Hapus** flag `--password`. Ganti:
  - `--password-stdin` → baca satu baris dari stdin (cocok untuk CI/pipa).
  - Tanpa flag → prompt interaktif `term.ReadPassword(int(os.Stdin.Fd()))` (tidak echo).
- Password tidak pernah muncul di `cmd.String()`, help text, atau error message.
- `create-branch` **tidak** menerima password sama sekali (server sudah punya rahasia di `dbms_secrets` via `dbms add --password-stdin`).

### 6. Anti-leak logging
- GORM logger: `logger.Config{ParameterizedQueries: true}` untuk koneksi registry/business.
- Helper `maskDSN(entry)` → `host=... port=... user=... password=REDACTED dbname=... sslmode=...`.
- Pastikan `db_registry_audit` tidak menyimpan value: gunakan `changed_fields` dari `information_schema` diff (nama kolom saja).

## Acceptance
- [ ] `SELECT password_enc FROM dbms_secrets` menampilkan envelope JSON (`{"v":1,"alg":"AES-GCM",...}`), bukan plaintext.
- [ ] `SELECT * FROM db_registry` **tidak** memuat field password/host/port/user apa pun.
- [ ] `db.GetGorm("business")` berhasil dekripsi & konek via `dbms`; bila `APP_KEY` tidak ada → error jelas, server tetap start untuk `default`.
- [ ] `bin/business dbms add` / `create-branch` tanpa `--password`; password hanya via stdin/prompt; `--password` ditolak dengan pesan arahan.
- [ ] Log aplikasi tidak memuat password (grep log → 0 hit).
- [ ] `INSERT/UPDATE/DELETE` pada `db_registry`/`dbms` menghasilkan baris di `db_registry_audit` dengan `changed_fields` nama kolom saja.
- [ ] Rotasi password `dbms_secrets` (satu baris) langsung berlaku untuk semua cabang pada `dbms_key` yang sama.

## Dependencies
- WO-01 (tabel & Pool registry), WO-04 (CLI create-branch).
- Memakai `internal/secrets` yang sudah ada — **tidak** menambah dependency.

## Catatan Operasional
- `APP_KEY` wajib: (a) 32 byte base64, (b) disimpan di docker secret / KMS / env CI, (c) **tidak** ikut dalam dump database, (d) dirotasi berkala bersama `key_version`.
- Hose: bila `APP_KEY` hilang → seluruh kredensial cabang tidak dapat didekripsi. Simpan backup key di secret manager dengan akses terbatas.

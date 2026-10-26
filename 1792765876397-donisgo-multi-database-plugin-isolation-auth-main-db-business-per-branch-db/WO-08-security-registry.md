---
title: 'WO-08: Keamanan Registry — Enkripsi, TLS, Audit, CLI Aman'
description: Pisah tabel secrets, enkripsi AES-GCM via internal/secrets (APP_KEY), sslmode verify-full, audit trigger, CLI tanpa --password (pakai --password-stdin / term.ReadPassword), masking log.
status: complete
created: "2026-10-26"
updated: "2026-10-26"
---
# WO-08: Keamanan Registry — Enkripsi, TLS, Audit, CLI Aman

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
| T5 | App role baca semua baris registry | `REVOKE ALL` di `db_registry_secrets` |
| T6 | Tidak ada jejak siapa ubah kredensial | Tabel `db_registry_audit` + trigger |
| T7 | APP_KEY di tempat sama dengan dump | Key hanya dari docker secret/env, tidak di DB |

## Task

### 1. Migration: split tables + audit
File: `plugins/business/migrations/postgres/<timestamp>_db_registry_secrets.up.sql` (+ `.down.sql`).

```sql
-- Metadata (tanpa rahasia)
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

-- Rahasia (ciphertext saja)
CREATE TABLE IF NOT EXISTS db_registry_secrets (
    registry_id  uuid PRIMARY KEY REFERENCES db_registry(id) ON DELETE CASCADE,
    password_enc text NOT NULL,
    key_version  int  NOT NULL DEFAULT 1,
    rotated_at   timestamptz,
    updated_at   timestamptz NOT NULL DEFAULT now()
);
REVOKE ALL ON db_registry_secrets FROM PUBLIC;

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

Trigger `trg_db_registry_audit` (`AFTER INSERT OR UPDATE OR DELETE`) pada `db_registry` dan `db_registry_secrets` — isi `action` + `changed_fields` (nama kolom saja, **tidak pernah nilai**).

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
- `LoadRegistry`/`LoadRegistryEntry` query JOIN `db_registry r LEFT JOIN db_registry_secrets s ON s.registry_id = r.id`.
- Struct entry **tidak** memuat password plaintext di field publik; simpan `PasswordEnc` dan resolve saat build DSN via `openPassword`.
- Tambah `SaveRegistrySecret(main *gorm.DB, name, plainPassword string, keyVersion int) error` — enkripsi lalu UPSERT ke `db_registry_secrets` (dipakai CLI create-branch & rotate).
- `RotateSecret(main *gorm.DB, name string) error` — dekripsi dengan key lama (via `key_version`) → enkripsi dengan key aktif → update `key_version`, `rotated_at`.

### 4. TLS
- `sslmode` default kolom = `verify-full`; compose lokal boleh override ke `require`/`disable` untuk dev.
- Compose: mount `./certs/ca.crt:/etc/ssl/certs/db-ca.crt:ro` ke service `app`; Postgres service diberi `ssl=on` + cert (opsional untuk dev).
- Dokumentasikan di README: production **wajib** `verify-full`.

### 5. CLI aman (`internal/businesscli`)
- **Hapus** flag `--password`. Ganti:
  - `--password-stdin` → baca satu baris dari stdin (cocok untuk CI/pipa).
  - Tanpa flag → prompt interaktif `term.ReadPassword(int(os.Stdin.Fd()))` (tidak echo).
- Password tidak pernah muncul di `cmd.String()`, help text, atau error message.
- `create-branch` memanggil `SaveRegistrySecret()` (enkripsi) — **bukan** INSERT plaintext.

### 6. Anti-leak logging
- GORM logger: `logger.Config{ParameterizedQueries: true}` untuk koneksi registry/business.
- Helper `maskDSN(entry)` → `host=... port=... user=... password=REDACTED dbname=... sslmode=...`.
- Pastikan `db_registry_audit` tidak menyimpan value: gunakan `changed_fields` dari `information_schema` diff (nama kolom saja).

## Acceptance
- [ ] `SELECT password_enc FROM db_registry_secrets` menampilkan envelope JSON (`{"v":1,"alg":"AES-GCM",...}`), bukan plaintext.
- [ ] `SELECT * FROM db_registry` **tidak** memuat field password apa pun.
- [ ] `db.GetGorm("business")` berhasil dekripsi & konek; bila `APP_KEY` tidak ada → error jelas, server tetap start untuk `default`.
- [ ] `sslmode` default = `verify-full` pada entri baru.
- [ ] `bin/business create-branch` tanpa `--password`; password hanya via stdin/prompt; `--password` ditolak dengan pesan arahan.
- [ ] Log aplikasi tidak memuat password (grep log → 0 hit).
- [ ] `INSERT/UPDATE/DELETE` pada `db_registry` menghasilkan baris di `db_registry_audit` dengan `changed_fields` nama kolom saja.
- [ ] Baris lama berformat plaintext tetap terbaca (pass-through) lalu bisa di-rotate via `RotateSecret`.

## Dependencies
- WO-01 (tabel & Pool registry), WO-04 (CLI create-branch).
- Memakai `internal/secrets` yang sudah ada — **tidak** menambah dependency.

## Catatan Operasional
- `APP_KEY` wajib: (a) 32 byte base64, (b) disimpan di docker secret / KMS / env CI, (c) **tidak** ikut dalam dump database, (d) dirotasi berkala bersama `key_version`.
- Hose: bila `APP_KEY` hilang → seluruh kredensial cabang tidak dapat didekripsi. Simpan backup key di secret manager dengan akses terbatas.

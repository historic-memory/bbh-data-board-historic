---
title: 'WO-09: Refactor Registry DBMS-centric (koreksi kontrak --key/--branch-key/--branch-name)'
description: 'Koreksi arsitektur: server DBMS didaftarkan sekali di tabel dbms (key unik) + dbms_secrets; db_registry hanya menyimpan dbms_key + database_name. CLI: dbms add/list/rm, create-branch --key --branch-key --branch-name. Migrasi bentuk lama, update loader/pool/DSN, re-test end-to-end.'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
revision: 1
---
# WO-09: Refactor Registry DBMS-centric

## Kenapa WO ini ada (bukti pelanggaran kontrak)
Implementasi WO-01..WO-08 menyimpan **host/port/user/password di setiap baris `db_registry`**. Hasilnya, server yang sama didaftarkan berulang:

```
 name      |       host        | port | user_name |  database_name
-----------+-------------------+------+-----------+----------------
 business  | postgres-business | 5432 | postgres  | bbh_business
 cbg-jkt   | postgres-business | 5432 | postgres  | bbh_cbg_jkt   <- duplikat
 cbg-tes   | postgres-business | 5432 | postgres  | bbh_cbg_tes   <- duplikat
 default   | postgres          | 5432 | postgres  | bbh_data_board
```

Ganti password/pindah host = edit N baris. Ini **melanggar kontrak** yang sudah disepakati (konsep `dbms_key`):
> "--key key_dbms --branch-key xxxx --branch-name taratara. ... kita perlu insert dbms connection ke database utama dengan key tertentu nanti itu jadi acuan"

Target: server = **satu baris** `dbms` dengan `key`; cabang = baris `db_registry` yang **mengacu** `dbms_key` dan hanya menyimpan `database_name`.

## Kontrak CLI Final (SPEC D-20)
```bash
# 1) daftarkan server SEKALI
bin/business dbms add --key business-dbms \
  --host postgres-business --port 5432 --user postgres \
  --sslmode verify-full --label "Server Cabang" --password-stdin

# 2) daftarkan cabang -> hanya mengacu key
bin/business create-branch --key business-dbms \
  --branch-key taratara --branch-name "Cabang Taratara"

# 3) opsional
bin/business dbms list
bin/business dbms rm --key business-dbms   # ditolak bila masih dipakai
```
- `--key` = acuan server DBMS (`dbms.key`).
- `--branch-key` = nama koneksi cabang (`db_registry.name`, mis. `taratara` / `cbg-taratara`).
- `--branch-name` = label manusia (`db_registry.label`).
- **Tidak ada** `--host/--port/--user/--password` di `create-branch`.

## Skema Target (main DB)
```sql
CREATE TABLE IF NOT EXISTS dbms (
    id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    key            varchar(100) NOT NULL UNIQUE,      -- acuan
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

CREATE TABLE IF NOT EXISTS dbms_secrets (
    dbms_id      uuid PRIMARY KEY REFERENCES dbms(id) ON DELETE CASCADE,
    password_enc text NOT NULL,                        -- AES-GCM envelope
    key_version  int  NOT NULL DEFAULT 1,
    rotated_at   timestamptz,
    updated_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS db_registry (
    id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    name          varchar(100) NOT NULL UNIQUE,        -- branch-key / conn name
    role          varchar(20)  NOT NULL CHECK (role IN ('main','business','branch')),
    dbms_key      varchar(100) NOT NULL REFERENCES dbms(key) ON UPDATE CASCADE,
    database_name varchar(150) NOT NULL,
    label         varchar(150),
    status        varchar(20)  NOT NULL DEFAULT 'active' CHECK (status IN ('active','suspended')),
    created_at    timestamptz  NOT NULL DEFAULT now(),
    updated_at    timestamptz  NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_db_registry_role_status ON db_registry(role, status);
```
`db_registry_audit` tetap; trigger ditambah untuk `dbms` & `dbms_secrets`.

Resolusi DSN (semua koneksi non-default):
```sql
SELECT r.name, r.role, r.database_name, r.label, r.status,
       d.key AS dbms_key, d.driver, d.host, d.port, d.user_name, d.sslmode,
       d.max_open_conns, d.max_idle_conns, s.password_enc
FROM db_registry r
JOIN dbms d              ON d.key = r.dbms_key
LEFT JOIN dbms_secrets s ON s.dbms_id = d.id
WHERE r.status = 'active'
ORDER BY r.name;
```

## Task

### 1. `internal/db/registry_schema.go` — rewrite DDL + migrasi bentuk lama
- Definisikan `dbmsTablesSQL`, `dbmsSecretsSQL`, `dbmsIndexSQL`; ubah `registryTablesSQL` agar **tanpa** `host/port/user_name/sslmode/max_*` dan **tambah** `dbms_key`.
- `EnsureRegistrySchema(main)` urutan: `dbms` → `dbms_secrets` → `db_registry` → audit → trigger → seed → **forwardMigrateOldRegistry**.
- **Backward migration (idempotent, wajib):**
  1. Deteksi kolom lama: `SELECT count(*) FROM information_schema.columns WHERE table_name='db_registry' AND column_name='host'`.
  2. Jika ada:
     - Seed `dbms` `main-dbms` dari env `DB_*` dan `business-dbms` dari `BUSINESS_DB_*` (host default `postgres-business`).
     - `UPDATE db_registry SET dbms_key='main-dbms' WHERE role='main'`.
     - `UPDATE db_registry SET dbms_key='business-dbms' WHERE role IN ('business','branch')`.
     - Pindahkan `password_enc` dari `db_registry_secrets` ke `dbms_secrets` (per `dbms_key` target; `ON CONFLICT (dbms_id) DO NOTHING`).
     - `ALTER TABLE db_registry DROP COLUMN host, DROP COLUMN port, DROP COLUMN user_name, DROP COLUMN sslmode, DROP COLUMN max_open_conns, DROP COLUMN max_idle_conns`.
     - `DROP TABLE IF EXISTS db_registry_secrets`.
  - Semua langkah dibungkus cek kolom → aman diulang.

### 2. `internal/db/secret.go` — rekey ke DBMS
- `SaveDBMSSecret(main *gorm.DB, key, plain string, keyVersion int) error` → UPSERT `dbms_secrets` via `SELECT id FROM dbms WHERE key=?`.
- `openDBMSPassword(enc string) (string, error)` → `secrets.DecryptBlob` (pass-through plaintext lama).
- `RotateDBMSSecret(main *gorm.DB, key string) error`.
- Hapus/alias `SaveRegistrySecret` (atau pertahankan shim yang error agar tidak dipakai lagi).

### 3. `internal/db/registry.go` — loader + pool
- `DBMSEntry{Key, Driver, Host, Port, UserName, SSLMode, MaxOpenConns, MaxIdleConns, Label, Status}`.
- `DBRegistryEntry{Name, Role, DBMSKey, DatabaseName, Label, Status, Driver, Host, Port, UserName, SSLMode, MaxOpen, MaxIdle, PasswordEnc}` (hasil JOIN).
- `LoadRegistry(main)` → query JOIN di atas.
- `LoadRegistryEntry(main, name)` → JOIN + `WHERE r.name=?`.
- `LoadDBMS(main, key)`, `ListDBMS(main) []*DBMSEntry` (+ hitung `#connections` via subquery/LEFT JOIN), `AddDBMS`, `RemoveDBMS(main, key)` (guard: tolak bila `EXISTS db_registry WHERE dbms_key=key`).
- `Pool.GetGorm(name)`: `default` → env; lainnya → `LoadRegistryEntry` → `openDBMSPassword` → build DSN → cache. Error jelas `no credentials registered for dbms "X"`.

### 4. `internal/db/gorm.go` — DSN
- `openGormDB(entry *DBRegistryEntry)` build dari `entry.Host/Port/UserName/DatabaseName/SSLMode` (yang sudah hasil JOIN `dbms`) + password hasil dekripsi.
- `maskDSN` tetap `password=REDACTED`.
- Pool config dari `MaxOpenConns/MaxIdleConns` (`dbms`).

### 5. `internal/businesscli/` — CLI baru
- **`dbms.go` (baru):**
  - `dbms add --key --host --port --user --sslmode --label --password-stdin` → validasi key regex `^[a-z0-9-]+$` & unik → INSERT `dbms` → `SaveDBMSSecret` → print ringkasan (password REDACTED).
  - `dbms list` → tabel: key, driver, host, port, user, #connections, status.
  - `dbms rm --key` → guard referensi.
- **`create_branch.go` (rewrite flags):**
  - Flags: `--key` (wajib), `--branch-key` (wajib), `--branch-name`, `--db-name` (default `bbh_cbg_<branch-key>`), `--password-stdin` **tidak diperlukan** (pakai rahasia `dbms`).
  - Alur: `LoadDBMS(key)` → maintenance conn (`database=postgres`) → `CREATE DATABASE` (idempotent) → INSERT `db_registry(name, role, dbms_key, database_name, label, status)` → auto migrate → ringkasan.
  - Tolak `--host/--port/--user/--password/--code` dengan pesan arahan (breaking change).
- **`root.go`**: register `dbmsCmd`.
- **`status.go`/`ping.go`/`migrate.go`**: adaptasi struct (tampilkan `dbms_key`, host dari JOIN).
- **`registry.go`**: helper baca via JOIN.

### 6. Update dokumen & infra
- `README.md`: bagian registry → jelaskan `dbms` vs `db_registry`, contoh `dbms add` + `create-branch`.
- `.env.example`: tambah catatan `busines-dbms` seed & bahwa password disimpan di `dbms_secrets` (bukan `.env` untuk runtime).
- `docker-compose.yml`: tidak berubah wajib; pastikan `postgres-business` tetap ada.

### 7. Migrasi data lokal (dev)
- Karena data uji (`cbg-jkt`, `cbg-tes`) adalah dummy, boleh: `DROP TABLE db_registry, db_registry_secrets CASCADE;` lalu `EnsureRegistrySchema` re-create. Untuk lingkungan berisi data nyata gunakan jalur `forwardMigrateOldRegistry`.

## File Impact
| File | Aksi |
|---|---|
| `internal/db/registry_schema.go` | Rewrite DDL + forward migration |
| `internal/db/registry.go` | Struct JOIN + LoadDBMS/List/Add/Remove + Pool |
| `internal/db/secret.go` | SaveDBMSSecret/RotateDBMSSecret/openDBMSPassword |
| `internal/db/gorm.go` | DSN dari entry JOIN |
| `internal/businesscli/dbms.go` | **baru** — dbms add/list/rm |
| `internal/businesscli/create_branch.go` | Flags baru `--key/--branch-key/--branch-name` |
| `internal/businesscli/{root,registry,migrate,ping,status}.go` | Adaptasi |
| `README.md`, `.env.example` | Dokumentasi |

## Acceptance
- [ ] `go build ./...` + `go test ./...` lulus.
- [ ] `SELECT key, host, port, user_name FROM dbms` berisi **tepat satu** baris per server; `SELECT name, dbms_key, database_name FROM db_registry` menunjukkan beberapa cabang berbagi `dbms_key` yang sama.
- [ ] `SELECT * FROM db_registry` **tidak** memuat host/port/user/password.
- [ ] `password_enc` hanya ada di `dbms_secrets` (envelope AES-GCM), bukan plaintext.
- [ ] `bin/business dbms add --key business-dbms ... --password-stdin` sukses; `dbms list` menampilkan `#connections`.
- [ ] `bin/business dbms rm --key business-dbms` **ditolak** saat masih dipakai cabang.
- [ ] `bin/business create-branch --key business-dbms --branch-key taratara --branch-name "Cabang Taratara"` → DB dibuat, `db_registry` berisi `dbms_key=business-dbms`, auto-migrate penuh.
- [ ] `create-branch` tanpa `--key` → error jelas; `--key` tak terdaftar → error.
- [ ] `bin/business migrate up` (atau `./bin/console migrate up --plugin business`) menyentuh `business` + semua cabang aktif; tiap target log `dbms=<key>`.
- [ ] Migrasi bentuk lama idempotent: jalankan `EnsureRegistrySchema` dua kali → tidak error, data cabang tetap.
- [ ] Tidak ada password di log (`grep` = 0 hit).

## Dependencies
- WO-01 (registry), WO-04 (CLI), WO-08 (secrets) — **menggantikan** bagian skema/kredensial mereka.
- `internal/secrets` (AES-GCM, `APP_KEY`) — sudah ada.

## Urutan Commit (disarankan)
1. `refactor(db): dbms-centric registry schema + forward migration`
2. `feat(db): dbms loaders/pool + SaveDBMSSecret`
3. `feat(business-cli): dbms add/list/rm`
4. `refactor(business-cli): create-branch --key/--branch-key/--branch-name`
5. `docs: registry dbms vs db_registry`
6. `test: end-to-end create-branch via dbms key`

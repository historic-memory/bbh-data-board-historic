---
title: 'WO-01: Registry Koneksi Tabular di DB Utama (internal/db/registry.go)'
description: 'Tabel db_registry di database utama menyimpan semua koneksi (business + cabang). internal/db/registry.go: LoadRegistry (query tabel), Pool singleton per nama, GetGorm(name), NamesByRole. Koneksi default tetap dari env DB_*. Fallback warning jika tabel belum ada.'
status: complete
created: "2026-10-26"
updated: "2026-10-26"
revision: 2
---
# WO-01: Registry Koneksi Tabular di DB Utama

> **REVISI 2 (2026-10-26)** — Koreksi kontrak: kredensial server **tidak lagi** disimpan per-baris koneksi. Server didaftarkan sekali di `dbms` (key unik) + `dbms_secrets`; `db_registry` hanya menyimpan `dbms_key` + `database_name`. Implementasi koreksi ada di **WO-09**; dokumen ini adalah acuan skema target.

## Goal
Semua koneksi (business + cabang) disimpan di **tabel `db_registry` di database utama** — bukan file YAML, bukan env prefix. Koneksi utama (`default`) tetap dari `.env` (`DB_*`, cara normal main app). `db.GetGorm(name)` query tabel -> build DSN -> singleton pool per nama. Backward compat: bila tabel belum ada -> warning, hanya `default` yang tersedia.

## Task

### 1. Migration tabel `db_registry`
File: `plugins/business/migrations/postgres/<timestamp>_create_db_registry.up.sql` (+ `.down.sql`).

> Detail keamanan (tabel secrets enkripsi audit) didefinisikan di **WO-08**. WO-01 menyiapkan skema + loader.

Tabel ini punya **dua level** sesuai SPEC D-4/D-19:
```sql
-- Tabel server DBMS (didaftarkan sekali, acuan --key)
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

-- Rahasia server (ciphertext)
CREATE TABLE IF NOT EXISTS dbms_secrets (
    dbms_id      uuid PRIMARY KEY REFERENCES dbms(id) ON DELETE CASCADE,
    password_enc text NOT NULL,
    key_version  int  NOT NULL DEFAULT 1,
    rotated_at   timestamptz,
    updated_at   timestamptz NOT NULL DEFAULT now()
);
REVOKE ALL ON dbms_secrets FROM PUBLIC;

-- Koneksi logis (mengacu dbms.key via dbms_key, TANPA host/port/user/password)
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
```

Seed awal (idempotent): entri `default` (role=main) mengacu `dbms` dengan key=`main-dbms`, dan `business` (role=business) mengacu `business-dbms` — keduanya diset oleh `EnsureRegistrySchema` lewat env.

### 2. `internal/db/registry.go` — struct + loader
```go
type DBMSEntry struct {
    Key, Driver, Host string
    Port              int
    UserName, SSLMode string
    MaxOpenConns, MaxIdleConns int
    Label, Status     string
}
type DBRegistryEntry struct {
    Name, Role, DBMSKey, DatabaseName, Label, Status string
    // resolved via JOIN ke dbms:
    Driver, Host string
    Port         int
    UserName, SSLMode string
    MaxOpenConns, MaxIdleConns int
}
```

- `LoadRegistry(db *gorm.DB) ([]*DBRegistryEntry, error)`:
  - Query `SELECT r.name, r.role, r.database_name, r.dbms_key, r.label, d.driver, d.host, d.port, d.user_name, d.sslmode FROM db_registry r JOIN dbms d ON d.key = r.dbms_key WHERE r.status='active' ORDER BY r.name`.
  - Return slice terurut (name ASC) -> determinisme prompt `--select` & iterasi cabang.
  - **Graceful degradation**: bila error `relation "db_registry" does not exist` -> log warning `[db] db_registry table not found, only default connection available` dan return empty slice (bukan fatal).
- `LoadRegistryEntry(db *gorm.DB, name string) (*DBRegistryEntry, error)`: query satu baris `WHERE name=? AND status='active'` JOIN `dbms`; error jelas bila tidak ada.
- `LoadDBMS(db *gorm.DB, key string) (*DBMSEntry, error)`: query `dbms` via key; dipakai CLI `dbms add/list/rm`.

### 3. Pool singleton per nama
```go
type Pool struct {
    mu    sync.Mutex
    conns map[string]*gorm.DB
    main  *gorm.DB // handle ke DB utama untuk query registry
}
```
- `GetGorm(name string) (*gorm.DB, error)`:
  - `name == "default"` -> langsung `openGormDBFromEnv()` (backward compat, tanpa query).
  - Lainnya: cek cache -> jika miss, `LoadRegistryEntry(main, name)` -> `openGormDB(entry)` -> cache.
  - Singleton per nama; aman concurrent (`sync.Mutex`).
  - Error: `connection "X" not found in db_registry (or status != active)`.
- `Connections() []string` -> semua nama aktif dari registry (untuk `--select`).
- `NamesByRole(role string) []string` -> filter `role='branch'`/`'business'` (auto-iterasi migrate).
- `Invalidate(name string)` -> tutup & hapus dari cache (dipakai setelah admin update kredensial/database).

### 4. `internal/db/gorm.go` — refactor
- `openGormDB(entry *DBRegistryEntry) (*gorm.DB, error)` — load secrets via `JoinKeyToDBMSKey(entry.Name)` → decrypt `password_enc` via `secrets.DecryptBlob()` lalu build DSN:
  `host=%s port=%d user=%s password=%s dbname=%s sslmode=%s`.
- Helper `configurePool(sqlDB, maxOpen, maxIdle, lifetimeSec)` — nilai dari entry (dbms) atau default 25/5/300.
- Log DSN selalu `password=REDACTED`.
- `openGormDBFromEnv()` — ekstrak logika existing (baca `DB_*`), dipakai `default`.
- Log: `[db] connect name=<name> host=<h> port=<p> db=<d>` — **password tidak pernah di-log**.
- `GetGormDB()` tetap ada, delegasi ke `pool.GetGorm("default")`.

### 5. Backward compatibility
- `GetGormDB()`/`GetDB()` tidak berubah perilaku (baca `DB_*`).
- Server start normal tanpa tabel `db_registry` (warning saja).
- Tidak ada import `gopkg.in/yaml.v3` (tidak perlu — pakai tabel).

## Acceptance
- [ ] `go build ./...` + `go test ./internal/db/...` lulus.
- [ ] Migration `create_db_registry` apply sukses; tabel + index ada.
- [ ] `db.GetGorm("default")` tetap baca `DB_*` dari env (tanpa query registry).
- [ ] `db.GetGorm("business")` query `db_registry` JOIN `dbms` -> konek; error jelas bila tidak ada.
- [ ] `db.Connections()`/`NamesByRole("branch")` mengembalikan nama dari tabel.
- [ ] Tabel belum ada -> warning, server tetap start (bukan panic).
- [ ] Entri `status='suspended'` tidak muncul di `Connections()` dan tidak bisa di-`GetGorm`.
- [ ] Entri yang menunjuk `dbms_key` tanpa baris `dbms_secrets` → error jelas `no credentials registered for dbms "X"`.
- [ ] Beberapa koneksi dengan `dbms_key` sama berbagi satu baris `dbms` (tidak ada duplikasi host/port/user).
- [ ] Password tidak pernah muncul di log (verifikasi grep).

## Dependencies
- Tidak ada — fase pertama.

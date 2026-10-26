---
title: 'WO-01: Registry Koneksi Tabular di DB Utama (internal/db/registry.go)'
description: 'Tabel db_registry di database utama menyimpan semua koneksi (business + cabang). internal/db/registry.go: LoadRegistry (query tabel), Pool singleton per nama, GetGorm(name), NamesByRole. Koneksi default tetap dari env DB_*. Fallback warning jika tabel belum ada.'
status: complete
created: "2026-10-26"
updated: "2026-10-26"
---
# WO-01: Registry Koneksi Tabular di DB Utama

## Goal
Semua koneksi (business + cabang) disimpan di **tabel `db_registry` di database utama** — bukan file YAML, bukan env prefix. Koneksi utama (`default`) tetap dari `.env` (`DB_*`, cara normal main app). `db.GetGorm(name)` query tabel -> build DSN -> singleton pool per nama. Backward compat: bila tabel belum ada -> warning, hanya `default` yang tersedia.

## Task

### 1. Migration tabel `db_registry`
File: `plugins/business/migrations/postgres/<timestamp>_create_db_registry.up.sql` (+ `.down.sql`).

> Detail keamanan (tabel secrets, enkripsi, audit) didefinisikan di **WO-08**. WO-01 hanya menyiapkan skema + loader.

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

-- Rahasia terpisah (ciphertext) — detail di WO-08
CREATE TABLE IF NOT EXISTS db_registry_secrets (
    registry_id  uuid PRIMARY KEY REFERENCES db_registry(id) ON DELETE CASCADE,
    password_enc text NOT NULL,
    key_version  int  NOT NULL DEFAULT 1,
    rotated_at   timestamptz,
    updated_at   timestamptz NOT NULL DEFAULT now()
);
```

Seed awal (idempotent, `ON CONFLICT (name) DO NOTHING`): entri `default` (role main, dari env `DB_*`) dan `business` (role business, dari env `BUSINESS_DB_*` atau default `postgres-business:5432`).

### 2. `internal/db/registry.go` — struct + loader
```go
type DBRegistryEntry struct {
    Name         string
    Role         string // main | business | branch
    Host         string
    Port         int
    UserName     string
    DatabaseName string
    SSLMode      string
    MaxOpenConns int
    MaxIdleConns int
    Label        string
    Status       string // active | suspended

    PasswordEnc string // ciphertext dari db_registry_secrets (tidak di-log)
}
```

- `LoadRegistry(db *gorm.DB) ([]*DBRegistryEntry, error)`:
  - Query `SELECT r.*, s.password_enc FROM db_registry r LEFT JOIN db_registry_secrets s ON s.registry_id = r.id WHERE r.status='active' ORDER BY r.name`.
  - Return slice terurut (name ASC) -> determinisme prompt `--select` & iterasi cabang.
  - **Graceful degradation**: bila error `relation "db_registry" does not exist` -> log warning `[db] db_registry table not found, only default connection available` dan return empty slice (bukan fatal).
- `LoadRegistryEntry(db *gorm.DB, name string) (*DBRegistryEntry, error)`: query satu baris `WHERE name=? AND status='active'`; error jelas bila tidak ada.

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
- `openGormDB(entry *DBRegistryEntry) (*gorm.DB, error)` — dekripsi `PasswordEnc` via `openPassword()` (WO-08) lalu build DSN:
  `host=%s port=%d user=%s password=%s dbname=%s sslmode=%s`.
- Log DSN selalu `password=REDACTED`.
- `openGormDBFromEnv()` — ekstrak logika existing (baca `DB_*`), dipakai `default`.
- Helper `configurePool(sqlDB, maxOpen, maxIdle, lifetimeSec)` — nilai dari entry atau default 25/5/300.
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
- [ ] `db.GetGorm("business")` query `db_registry` -> konek; error jelas bila tidak ada.
- [ ] `db.Connections()`/`NamesByRole("branch")` mengembalikan nama dari tabel.
- [ ] Tabel belum ada -> warning, server tetap start (bukan panic).
- [ ] Entri `status='suspended'` tidak muncul di `Connections()` dan tidak bisa di-`GetGorm`.
- [ ] Entri tanpa baris `db_registry_secrets` → error jelas `no credentials registered for connection "X"`.
- [ ] Password tidak pernah muncul di log (verifikasi grep).

## Dependencies
- Tidak ada — fase pertama.

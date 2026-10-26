---
title: 'WO-02: ServiceDeps.Connections + ConnectionBinder'
description: Tambah Connections map di ServiceDeps, resolve koneksi di bootstrap (default wajib, business best-effort), optional interface ConnectionName().
status: complete
created: "2026-10-26"
updated: "2026-10-26"
---
# WO-02: ServiceDeps.Connections + ConnectionBinder

## Goal
Plugin bisa memilih koneksi DB sesuai namanya; auth (plugin existing) tetap bekerja tanpa perubahan.

## Task
1. **`internal/plugins/types.go`**:
   - `ServiceDeps` tambah field `Connections map[string]*gorm.DB` (boleh nil bila hanya default).
2. **`internal/plugins/registry.go`**:
   - `RegisterAllServices(db, store)` → `RegisterAllServices(db, store, connections map[string]*gorm.DB)` — update signature + semua call site.
3. **`internal/app/bootstrap.go`** (di `New()`/`attachPlugins`):
   - Resolve koneksi **lazy** sebelum attach plugins:
     - `default` → wajib (error fatal bila gagal — perilaku existing).
     - `business` → **best-effort**: `db.GetGorm("business")` (query tabel `db_registry`); bila entri tidak ada / DB down → `log.Printf("[WARNING] business DB unavailable: %v", err)` dan lanjut tanpa entri.
     - branch → **tidak dibuka saat boot** (lazy on demand).
   - Pass `connections` ke `RegisterAllServices`.
4. **Optional interface** di `internal/plugins/types.go`:
   ```go
   type ConnectionBinder interface { ConnectionName() string }
   func ConnectionNameOf(p Plugin) string // default "default"
   ```
   - `auth` TIDAK perlu diubah (default `default`).

## Acceptance
- [ ] `go build ./...` + `go test ./...` lulus.
- [ ] Server start normal dengan hanya `DB_*`; warning muncul bila entri `business` tak ada di `db_registry` / DB down.
- [ ] `deps.Connections["business"]` tersedia di plugin business (dites di WO-05).

## Dependencies
- WO-01 (`db.GetGorm`, pool).

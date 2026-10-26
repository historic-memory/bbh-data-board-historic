---
title: 'WO-03: Extract internal/migration + Guard --connection'
description: Extract logic migrasi ke internal/migration (migrator/ordering/tracking, semua terima *gorm.DB), flag --connection, guard isolasi D-13/D-14 di console migrate.
status: complete
created: "2026-10-26"
updated: "2026-10-26"
---
# WO-03: Extract `internal/migration/` + Console Guard `--connection`

## Goal
Logic migrasi reusable lintas binary (console + CLI business), dan migrasi main DB **tidak pernah** menyentuh business/cabang (dan sebaliknya).

## Task
1. **Package baru `internal/migration/`** — semua fungsi menerima `*gorm.DB` sebagai parameter, TIDAK memanggil `db.GetGormDB()` internal:
   - `migrator.go`: `MigrateFiles(dbConn *gorm.DB, path, targetName string) error` (loop applyUp/applyDown: list file → exec SQL → insert/delete record; dirty-state handling seperti existing).
   - `ordering.go`: ekspor `orderTargetsDynamically`, `listMigrationFiles`, `exists`, `nextMigrationNumber` (hanya butuh file system).
   - `tracking.go`: ekspor `ensureMigrationTables`, `ensureTargetEntry`, `isTargetDirty`, `setTargetDirty`, `getCurrentVersion`, `insertMigrationRecord`, `deleteMigrationRecord`.
2. **Refactor `internal/console/migrate.go`**:
   - Delegasi eksekusi ke `internal/migration`; `collectTargets()` tetap di console (butuh plugins registry).
   - Semua `applyUp/Down/DownAll` menerima `dbConn *gorm.DB` dari `db.GetGorm(connName)` — TIDAK LAGI hard-code `db.GetGormDB()`.
3. **Flag baru**: `--connection <name>` (persistent, default `"default"`).
4. **`connectionForTarget(t migrationTarget) string`**:
   - `core` → `default`;
   - plugin → `ConnectionNameOf(p)` (WO-02).
5. **Validasi SEBELUM eksekusi** (up/down/down-all/list):
   - Jika `--connection` ≠ `connectionForTarget(t)` → error `plugin %q is bound to connection %q; refusing to run against %q — no SQL executed` (exit non-zero, tidak ada SQL jalan).
   - Plugin terikat `business`: default behavior (tanpa flag) = iterasi `["business"] + NamesByRole("branch")`, per koneksi jalankan `MigrateFiles`; **non-blocking** antar DB; ringkasan per DB; exit non-zero bila ada yang gagal (AC-6).
   - Plugin lain / polos → hanya `default` (D-14).
6. **Log per target**: `[<target>] applying to connection=<name> host=<h> db=<n>`; `migrate list` tambah kolom `connection=`.

## Acceptance
- [ ] `go build ./...` + `go test ./internal/console/... ./internal/migration/...` lulus.
- [ ] `console migrate up --plugin business --connection default` → ditolak, SQL tidak jalan (AC-12).
- [ ] `console migrate up` polos → hanya `default` (AC-13).
- [ ] Migrasi `default` tidak menyentuh business/cabang dan sebaliknya (AC-14).
- [ ] `console migrate up --plugin business` → satu running: business + semua cabang (AC-5/AC-6).

## Dependencies
- WO-01 (pool, `LoadRegistry`/`NamesByRole`), WO-02 (`ConnectionNameOf`).

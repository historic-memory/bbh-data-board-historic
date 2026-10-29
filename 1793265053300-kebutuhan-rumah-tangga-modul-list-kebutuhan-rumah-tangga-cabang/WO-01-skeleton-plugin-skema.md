---
title: "WO-01: Skeleton plugin & skema kanonik"
description: "Buat plugin hsh, schema/fields.yaml, migrasi 5 tabel + enum status, dan seed daftar kategori referensi (7 kategori)."
status: complete
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - schema
  - migration
---

# WO-01: Skeleton plugin & skema kanonik

## Referensi
- SPEC: N-1, N-3, N-8, N-10, N-13, GAP-9, §Skema Kanonik.
- Pola acuan: `plugins/resident/plugin.go`, `plugins/resident/schema/fields.yaml`,
  `plugins/resident/model/models.go`, `plugins/resident/migrations/`, `internal/console/migrate.go`.

## Tugas
1. **Skeleton plugin** `plugins/hsh/` mengikuti `plugins/resident`:
   `plugin.go` (`ID() = "hsh"`, `RegisterServices/RegisterMiddleware/RegisterRoutes/Seed/ConsoleCommands`),
   `model/`, `services/`, `handlers/`, `schema/`, `migrations/`.
2. **`schema/fields.yaml`** = single source of truth (pola P-13):
   - `enums`: `need_status` → `URGENT`, `ENOUGH`, `UNKNOWN` (label Indonesia).
   - `units`: daftar REFERENSI statis (mis. `pcs`, `box`, `botol`, `pak`, `roll`, `unit`, `lusin`, `kg`, `liter`).
   - `tables`: 5 tabel di bawah, lengkap `name/type/required/unique/fk/enum/doc`.
   - `template`: kolom sheet DATA (dipakai WO-04).
3. **Migrasi** (SQL, via `internal/console/migrate.go`, prefix `hsh_`):
   - `hsh_categories` (id, code UNIQUE, name, sort_order, is_active, timestamps).
   - `hsh_items` (id, category_id FK, name, unit, est_price numeric(14,2), is_active, note, timestamps);
     index `UNIQUE (category_id, lower(name))`.
   - `hsh_item_aliases` (id, item_id FK, alias, timestamps); index `UNIQUE (lower(alias))`.
   - `hsh_periods` (id, code UNIQUE, label, started_on, ended_on, is_active, timestamps).
   - `hsh_entries` (id, period_id FK, item_id FK, status_code + `CHECK IN ('URGENT','ENOUGH','UNKNOWN')`,
     quantity numeric(12,2), unit_price numeric(14,2), note, recorded_by, timestamps);
     index `UNIQUE (period_id, item_id)`.
   - Semua `id TEXT PK` (UUIDv7 di Go), `created_at`/`updated_at timestamptz`.
4. **Seed referensi** (migrasi, bukan boot code): 7 kategori
   `baby_needs`, `milk`, `food`, `stationery`, `medicine`, `housekeeping` (perbaikan dari `Equpment`), `linen` —
   dengan `sort_order` dan `name` tampil. **Periode/entri TIDAK di-seed** (N-15, ditunda).
5. **`model/models.go`** + `model.NewID` (UUID v7, pakai `internal/uuid`).
6. **`schema/schema_test.go`**: pastikan setiap kolom `fields.yaml` terwakili di migrasi (pola `resident`).

## Bukti yang harus dilampirkan
- `docker compose exec app go build ./...` hijau; `go test ./plugins/hsh/...` hijau.
- `go build -o bin/console ./cmd/console` di container; `./bin/console migrate --plugin hsh` bersih dari DB kosong.
- `psql \dt hsh_*` → 5 tabel; `\d hsh_entries` → `CHECK` status benar.
- `SELECT code FROM hsh_categories ORDER BY sort_order` → 7 baris.
- Migrasi **down** juga diuji (drop bersih).

## Kriteria Selesai
- Plugin terdaftar (muncul di registry), no-op service aman.
- Skema sesuai SPEC; `fields.yaml` ↔ migrasi konsisten (test hijau).
- Tidak ada kode fitur di `internal/`; tidak ada perubahan pada plugin lain.

## Bukti Pelaksanaan (2026-10-29)
- **Kode:** commit `cf6367f` (tanpa push). File baru: `plugins/hsh/{plugin,cli}.go`,
  `model/{id,models}.go`, `services/service.go`, `handlers/handlers.go`,
  `schema/{fields.yaml,schema.go,schema_test.go}`, dan migrasi
  `migrations/postgres/2026_10_29_120100_create_hsh_core_tables.{up,down}.sql`.
  File diubah: `cmd/server/main.go`, `cmd/console/main.go` (registrasi plugin).
  **Tidak ada perubahan `internal/` maupun plugin lain.**
- **Build/vet/test (container `app`):** `go build ./...` hijau;
  `go vet ./plugins/hsh/... ./cmd/...` bersih; `go test ./plugins/hsh/...` hijau
  (`schema` 7 test: fields↔migrasi, prefix `hsh_`, tanpa `ref_*`, CHECK status,
  template refs, enum refs, units).
- **Migrasi up dari DB kosong** (`hsh_mig_test`): `migrate --plugin hsh up` →
  `hsh applied ... create_hsh_core_tables`. `pg_tables` → 5 tabel `hsh_*`;
  `hsh_entries_status_code_check` = `CHECK (status_code = ANY
  (ARRAY['URGENT','ENOUGH','UNKNOWN']))`; `SELECT code ... ORDER BY sort_order`
  → 7 baris (`baby_needs`..`linen`).
- **Migrasi down:** `migrate --plugin hsh down` → `rolled back version 1`;
  `count(hsh_* tables)` = **0**.
- **Registry:** `console hsh version` → `hsh plugin 0.1.0`.

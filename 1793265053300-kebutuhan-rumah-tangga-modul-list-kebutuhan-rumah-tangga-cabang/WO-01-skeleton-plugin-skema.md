---
title: "WO-01: Skeleton plugin & skema kanonik"
description: "Buat plugin hsh, schema/fields.yaml, migrasi 5 tabel + enum status, dan seed daftar kategori referensi (7 kategori)."
status: pending
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

---
title: "WO-01: Skeleton plugin & skema kanonik"
description: "Buat plugin sec, schema/fields.yaml, migrasi 5 tabel + enum status, dan seed daftar kategori referensi (2 kategori: elektronik-peralatan, mainan-rekreasi)."
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
- SPEC: S-1, S-4, S-8, S-10, S-16, GAP-9, §Skema Kanonik.
- Pola acuan: `plugins/resident/plugin.go`, `plugins/resident/schema/fields.yaml`,
  `plugins/resident/model/models.go`, `plugins/resident/migrations/`, `internal/console/migrate.go`.
- Kembar: `plugins/hsh` (WO-01-nya) — bentuk identik, prefix `sec_`.

## Tugas
1. **Skeleton plugin** `plugins/sec/` mengikuti `plugins/resident`/`plugins/hsh`:
   `plugin.go` (`ID() = "sec"`, `RegisterServices/RegisterMiddleware/RegisterRoutes/Seed/ConsoleCommands`),
   `model/`, `services/`, `handlers/`, `schema/`, `migrations/`.
2. **`schema/fields.yaml`** = single source of truth:
   - `enums`: `need_status` → `URGENT`, `ENOUGH`, `UNKNOWN` (label Indonesia).
   - `units`: daftar REFERENSI statis (mis. `pcs`, `unit`, `set`, `roll`, `box`, `pak`, `lusin`, `meter`).
   - `tables`: 5 tabel di bawah, lengkap `name/type/required/unique/fk/enum/doc`.
   - `template`: kolom sheet DATA (dipakai WO-04), termasuk `Periode`, `Kategori`, `Nama Kebutuhan`,
     `Merk/Brand`, `Spesifikasi`, `Satuan`, `Harga Satuan`, `Status`, `Jumlah`, `Catatan`.
3. **Migrasi** (SQL, via `internal/console/migrate.go`, prefix `sec_`):
   - `sec_categories` (id, code UNIQUE, name, sort_order, is_active, timestamps).
   - `sec_items` (id, category_id FK, name, brand, specification, unit, est_price numeric(14,2),
     is_active, note, timestamps); index `UNIQUE (category_id, lower(name))`.
   - `sec_item_aliases` (id, item_id FK, alias, timestamps); index `UNIQUE (lower(alias))`.
   - `sec_periods` (id, code UNIQUE, label, started_on, ended_on, is_active, timestamps).
   - `sec_entries` (id, period_id FK, item_id FK, status_code + `CHECK IN ('URGENT','ENOUGH','UNKNOWN')`,
     quantity numeric(12,2), unit_price numeric(14,2), note, recorded_by, timestamps);
     index `UNIQUE (period_id, item_id)`.
   - Semua `id TEXT PK` (UUIDv7 di Go), `created_at`/`updated_at timestamptz`.
4. **Seed referensi** (migrasi, bukan boot code): 2 kategori
   `elektronik-peralatan` (`Elektronik & Peralatan`) dan `mainan-rekreasi` (`Mainan & Rekreasi`) —
   dengan `sort_order` dan `name` tampil. **Periode/entri TIDAK di-seed** (ditunda).
5. **`model/models.go`** + `model.NewID` (UUID v7, pakai `internal/uuid`).
6. **`schema/schema_test.go`**: pastikan setiap kolom `fields.yaml` terwakili di migrasi (pola `resident`/`hsh`).

## Bukti yang harus dilampirkan
- `docker compose exec app go build ./...` hijau; `go test ./plugins/sec/...` hijau.
- `go build -o bin/console ./cmd/console` di container; `./bin/console migrate --plugin sec` bersih dari DB kosong.
- `psql \dt sec_*` → 5 tabel; `\d sec_entries` → `CHECK` status benar.
- `SELECT code FROM sec_categories ORDER BY sort_order` → 2 baris.
- Migrasi **down** juga diuji (drop bersih).

## Kriteria Selesai
- Plugin terdaftar (muncul di registry), no-op service aman.
- Skema sesuai SPEC; `fields.yaml` ↔ migrasi konsisten (test hijau).
- Tidak ada kode fitur di `internal/`; tidak ada perubahan pada plugin lain.

## Bukti Pelaksanaan
**Status: `complete`** (2026-10-29). Kode commit `5619cd4`.

### Berkas yang dibuat
- `plugins/sec/plugin.go` — `ID()="sec"`, `RegisterServices`, `RegisterRoutes` (no-op WO-01), `Seed` no-op, `ConsoleCommands`.
- `plugins/sec/cli.go` — grup `sec` + `sec version` (0.1.0).
- `plugins/sec/model/id.go` — `NewID()` = UUID v7 via `internal/uuid`.
- `plugins/sec/model/models.go` — `Category`, `Item` (+`brand`,`specification`), `ItemAlias`, `Period`, `Entry`.
- `plugins/sec/services/service.go` — `New(db)` (nil-db aman), `DB()`, `Schema()`, sentinel error, `nilIfBlank`/`trimToNil`/`forceInactive`, deteksi unique violation.
- `plugins/sec/handlers/handlers.go` — skeleton `Handler`/`New`.
- `plugins/sec/schema/fields.yaml` — SSOT: enum `need_status`, `units` REFERENSI (8), 5 tabel, `template` (10 kolom, termasuk Merk/Brand & Spesifikasi), `validation` (3 rule).
- `plugins/sec/schema/schema.go` + `schema_test.go` — loader & 8 test penjaga (kolom↔migrasi, prefix `sec_`, tanpa `ref_*`, CHECK status, kolom template, enum, units, rule validasi).
- `plugins/sec/migrations/postgres/2026_10_29_130100_create_sec_core_tables.{up,down}.sql` — 5 tabel + index + CHECK + seed 2 kategori.
- `cmd/server/main.go`, `cmd/console/main.go` — registrasi `secplugin.New()`.

### Hasil verifikasi (container `app`)
- `go build ./...` hijau; `go vet ./plugins/sec/...` bersih; `go test ./plugins/sec/...` hijau (`ok .../plugins/sec/schema`).
- `gofmt -l plugins/sec cmd` → kosong.
- `./bin/console migrate --plugin sec up` dari DB tanpa target `sec` → `sec applied 2026_10_29_130100_create_sec_core_tables.up.sql`; `list` → `current=1 dirty=false pending=0`.
- `\dt sec_*` → 5 tabel (`sec_categories`, `sec_entries`, `sec_item_aliases`, `sec_items`, `sec_periods`).
- `\d sec_entries` → `sec_entries_status_code_check CHECK (status_code = ANY (ARRAY['URGENT','ENOUGH','UNKNOWN']))`, unique `(period_id,item_id)`, FK ke `sec_items`/`sec_periods` (CASCADE).
- `SELECT code,name,sort_order FROM sec_categories` → 2 baris: `elektronik-peralatan` (10), `mainan-rekreasi` (20).
- Migrasi **down** diuji: `rolled back version 1 -> current=0`, jumlah tabel `sec_*` = 0; lalu `up` lagi → 5 tabel + 2 kategori (idempoten).
- `./bin/console sec version` → `sec plugin 0.1.0`.

### Catatan turunan
- Tabel staging impor (`sec_import_batches`/`sec_import_rows`) sengaja **belum** ada — dibuat di WO-06 (migrasi terpisah, versi 2), sama seperti `hsh`.
- `fields.yaml` sengaja belum memuat tabel staging agar `schema_test.go` (migrasi ⊇ field) hijau.

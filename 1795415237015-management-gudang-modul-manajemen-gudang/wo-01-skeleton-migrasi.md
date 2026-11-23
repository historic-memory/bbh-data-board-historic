---
title: WO-01 — Skeleton plugin warehouse + migrasi inti
description: "Skeleton plugin wh + migrasi 5 tabel inti"
status: pending
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-01 — Skeleton plugin `warehouse` + migrasi inti

## Tujuan

Buat plugin `plugins/warehouse` (terdaftar di `cmd/server` + `cmd/console`) dengan migrasi 5 tabel inti. Tanpa route (WO-02+).

## Lingkup

- **A. Skeleton** — `plugin.go` (ID `warehouse`, RegisterRoutes no-op), `cli.go` (`wh version`), `model/{id,models,enums}.go`, `services/service.go` (New/DB/Schema + sentinel error + `forceInactive` pola hsh), `handlers/handlers.go` stub, `schema/{fields.yaml,schema.go,schema_test.go}` (units: pcs/unit/set/box/lusin/gram/kg/liter + aturan template).
- **B. Migrasi** `2026_11_23_140100_create_wh_core_tables.{up,down}.sql` — `wh_categories`, `wh_items` (category_id FK, name UNIQUE per kategori), `wh_warehouses`, `wh_batches` (warehouse_id NOT NULL, qty_remaining CHECK >= 0, entered_at, expires_at NULL), `wh_movements` (direction CHECK OUT|ADJUST). UUIDv7 TEXT PK. Tanpa seed data (D-12).

## Acceptance Criteria

- AC-1 `go build/vet/test ./...` hijau.
- AC-2 `console migrate --plugin warehouse up` → 5 tabel + index + CHECK; `down` → 0 tabel; up ulang idempoten.
- AC-3 Plugin terdaftar & binary jalan.

## Non-tujuan

Route/permission/CRUD (WO-02), transaksi (WO-03), template/impor (WO-04..06), UI (WO-07), swagger (WO-08), AI chat (WO-09).

## Catatan Implementasi (Dev)

(disi bertahap per bagian A/B)


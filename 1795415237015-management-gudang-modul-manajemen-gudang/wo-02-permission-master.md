---
title: WO-02 — Permission + CRUD master data
description: "Seed permission + CRUD kategori/produk/gudang"
status: pending
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-02 — Permission + CRUD master data

## Prasyarat

WO-01 lulus.

## Lingkup

- **A. Permission** — migrasi plugin `auth`: `2026_11_23_140200_add_warehouse_permissions` — seed `warehouse.read/write/delete` (admin read+write, viewer read; root `*`). **Jangan taruh seed di plugin warehouse** (jebakan resident).
- **B. CRUD master** — `services/{category,item,warehouse}.go` + handlers + 14 route di `plugin.go`:
  - `/api/admin/wh/categories` (+`/:id`): CRUD kategori; name unique per instance → 409.
  - `/api/admin/wh/items` (+`/:id`): CRUD produk (category_id FK, name unique per kategori → 409, unit teks); search q.
  - `/api/admin/wh/warehouses` (+`/:id`): CRUD gudang.
  - Kategori/produk/gudang yang masih dipakai → 409 `ErrInUse` (pola hsh).

## Acceptance Criteria

- AC-1 Build/vet/test hijau; permission muncul via `auth permission list`.
- AC-2 CRUD 201/200/404/409; guard RBAC: viewer read 200 / write 403; tanpa token 401.
- AC-3 Hapus terpakai → 409; hapus tak terpakai → 200.

## Non-tujuan

Batch/movement/summary (WO-03).

## Catatan Implementasi (Dev)

(disi bertahap)


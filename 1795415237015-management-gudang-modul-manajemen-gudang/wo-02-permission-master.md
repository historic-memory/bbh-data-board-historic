---
title: WO-02 — Permission + CRUD master data
description: "Seed permission + CRUD kategori/produk/gudang"
status: complete
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

### A. Permission (plugin `auth`, BUKAN plugin warehouse)

- `plugins/auth/migrations/postgres/2026_11_23_140200_add_warehouse_permissions.up.sql:16` — seed `warehouse.read/write/delete` lewat migrasi **plugin auth** sesuai D-1 + jebakan 403 resident/hsh/sec: RBAC fail-closed, route ber-guard akan 403 permanen bila permission tidak di-seed dan plugin warehouse tidak boleh menanamnya sendiri (bisa di-uninstall tanpa memutus RBAC).
- `...add_warehouse_permissions.up.sql:20` — mapping ikut pola hsh/sec: grup `admin` = read+write; grup `viewer` (default) = read; `warehouse.delete` **tidak** di-group manapun (khusus root `*`). Semua lewat subquery `group_id`, jadi tidak bergantung pada UUID grup.
- `...add_warehouse_permissions.up.sql:16` — `ON CONFLICT (key) DO NOTHING` di dua jalur (permissions & group_permissions) → migrasi idempoten, aman dijalankan ulang/di DB yang sudah punya baris mirip.
- `...add_warehouse_permissions.down.sql:1` — cukup `DELETE FROM permissions`; `group_permissions` ikut terhapus lewat `ON DELETE CASCADE`, tidak perlu dua statement yang bisa desinkron.

### B. CRUD master

- `plugins/warehouse/services/category.go:41` — `CreateCategory`: validasi name non-blank (400 `ErrInvalid`) lalu insert; unique index `uq_wh_categories_name` pada `LOWER(name)` (WO-01) membuat duplikat beda huruf pun gagal → `wrapUnique` memetakannya ke `ErrDuplicate` (409).
- `plugins/warehouse/services/category.go:58` — `wantActive := row.IsActive` ditangkap **sebelum** `Create` lalu `forceInactive` bila false: kolom bertag `gorm:"default:true"` dibuang dari INSERT dan **di-back-fill dari RETURNING** (gotcha B-1b hsh) sehingga tanpa ini `is_active=false` tersimpan `true`.
- `plugins/warehouse/services/category.go:106` — `DeleteCategory` menghitung `wh_items` yang memakai kategori lebih dulu → 409 `ErrInUse`; FK violation juga dipetakan ke `ErrInUse` sebagai jaring kedua (mis. balapan), `RowsAffected == 0` → `ErrNotFound`.
- `plugins/warehouse/services/item.go:47` — `ItemView` + `itemSelect` JOIN `wh_categories` agar list/detail membawa `category_name` (UI WO-07 tidak perlu fetch kategori terpisah).
- `plugins/warehouse/services/item.go:82` — `ListItems`: filter `category_id`, `q` (ILIKE nama; pencarian alias tidak ada karena SPEC §2 tidak memakai tabel alias untuk gudang), `include_inactive`, limit default 50/maks 200; `COUNT(*)` dihitung terpisah agar `total` akurat walau dipaginasi.
- `plugins/warehouse/services/item.go:127` — `ensureCategoryExists`: category_id tak dikenal → **400** `ErrInvalid` (bukan 404) karena yang salah adalah payload, bukan resource yang dialamatkan — eksplisit diminta WO-02.
- `plugins/warehouse/services/item.go:141` — `CreateItem` juga memakai `wantActive`+`forceInactive` (gotcha sama) dan `normalizeUnit` D-8 (unit REFERENSI, disimpan huruf kecil); duplikat nama dalam satu kategori → 409 lewat `uq_wh_items_category_name`.
- `plugins/warehouse/services/item.go:184` — `UpdateItem(requireCore)`: satu fungsi untuk PUT (`requireCore=true`, wajib `category_id`+`name`) dan PATCH; field opsional lewat `trimToNil` sehingga string kosong **mengosongkan** kolom (helper WO-01).
- `plugins/warehouse/services/item.go:236` — `DeleteItem` menolak 409 bila sudah ada `wh_batches` yang menunjuk item: batch = riwayat stok, tidak boleh dihapus tidak sengaja.
- `plugins/warehouse/services/warehouse.go:10` — `WarehouseInput` sengaja tanpa `is_active`: tabel `wh_warehouses` (SPEC §2) tidak punya kolom itu, jadi tidak ada flag yang bisa dibalik dan tidak perlu `forceInactive`.
- `plugins/warehouse/services/warehouse.go:77` — `DeleteWarehouse` 409 bila ada batch: `wh_batches.warehouse_id` NOT NULL (D-2) sehingga menghapus gudang terpakai akan memutus riwayat batch.
- `plugins/warehouse/handlers/master.go:18` — 15 handler master CRUD (5 kategori + 5 item + 5 gudang) dengan anotasi godoc swagger lengkap (WO-08 tinggal regenerate) dan `newListResponse` agar list kosong = `[]`.
- `plugins/warehouse/handlers/master.go:45` — `CreateCategory`/`CreateItem`/`CreateWarehouse` memakai `writeBadPayload` untuk JSON rusak (400) dan `writeError` untuk sentinel; tidak ada pemetaan status duplikat per handler → konsisten dengan WO-01.
- `plugins/warehouse/plugin.go:71` — `RegisterRoutes` tidak lagi no-op: 15 route (GET/POST collection + GET/PUT/DELETE `/:id`) di tiga grup `api.Group("/admin/wh/...")` dengan guard `warehouse.read|write|delete`; grup `api` sudah ber-prefix `/api` sehingga path relatif.
- `plugins/warehouse/plugin.go:65` — `authservices.New(p.db)` diambil dari handle plugin (bukan global) supaya guard tetap mengikuti DB yang sama dengan service warehouse.
- `plugins/warehouse/services/master_db_test.go:88` — `TestCreateItemRejectsUnknownCategory` membuktikan **400 vs 404** secara eksplisit (dua kasus: id asing & id kosong).
- `plugins/warehouse/services/master_db_test.go:151` — `TestDeleteInUseIsRefused` membuat batch nyata via SQL untuk memaksa jalur 409 pada item & gudang (bukan sekadar kategori), lalu membersihkan semuanya lewat `t.Cleanup`.
- `plugins/warehouse/services/master_db_test.go:200` — `TestCreateHonoursInactiveFlag` + `:246` `TestCreateDefaultsToActive`: regresi gotcha GORM `default:true` untuk kategori **dan** item, termasuk cek langsung ke DB (bukan hanya nilai struct hasil return).
- `plugins/warehouse/services/master_db_test.go:270` — `TestNormalizeUnitRejectsUnknown`: D-8 diuji dua arah — `karung` → 400 `ErrUnknownUnit`, `PCS` → tersimpan `pcs`.

### Bukti (2026-11-23, container `app`, root@admin.local)

**Build/vet/test (AC-1)**
- `go build ./... && go vet ./...` → OK.
- `go test ./plugins/warehouse/... -v` → **8 test DB RUN** (bukan skip): `TestCreateCategoryRejectsDuplicate`, `TestCreateItemRejectsUnknownCategory`, `TestCreateItemRejectsDuplicateWithinCategory`, `TestDeleteInUseIsRefused`, `TestDeleteUnusedSucceeds`, `TestCreateHonoursInactiveFlag`, `TestCreateDefaultsToActive`, `TestNormalizeUnitRejectsUnknown` + 9 test schema. `go test ./...` → **0 FAIL** seluruh repo.
- `console auth permission list` → `warehouse.delete / warehouse.read / warehouse.write`.
- `auth group show --key admin` → memuat `warehouse.read,warehouse.write` (**tanpa** delete); `--key viewer` → `warehouse.read`.

**Status code (AC-2/AC-3)**
| # | Request | Expect | Got |
|---|---|---|---|
| 1 | `POST /wh/categories` | 201 | 201 |
| 2 | `POST /wh/categories` nama sama beda huruf | 409 | 409 |
| 3 | `GET /wh/categories/:id` | 200 | 200 |
| 4 | `GET /wh/categories/does-not-exist` | 404 | 404 |
| 5 | `POST /wh/items` (unit `BOX`) | 201 | 201, `unit:"box"` |
| 6 | `POST /wh/items` category_id asing | 400 | 400 |
| 7 | `POST /wh/items` duplikat dalam kategori | 409 | 409 |
| 8 | `POST /wh/items` unit `karung` | 400 | 400 |
| 9 | `PUT /wh/items/:id` tanpa `name` | 400 | 400 |
| 10 | `PUT /wh/items/:id` lengkap | 200 | 200 |
| 11 | `GET /wh/items?q=sgm` | 200 | 200, `total:1` |
| 12 | `DELETE /wh/categories/:id` (berisi item) | 409 | 409 |
| 13 | `POST /wh/warehouses` | 201 | 201 |
| 14 | `POST /wh/warehouses` nama sama beda huruf | 409 | 409 |
| 15 | `GET /wh/warehouses` | 200 | 200 |
| 16 | `GET /wh/categories` tanpa token | 401 | 401 |

**RBAC**
- Akun `viewer` group: `GET /wh/categories|items|warehouses` → **200/200/200**; `POST /wh/categories` → **403** `{"error":"izin ditolak: butuh warehouse.write"}`; `DELETE` → **403** (`butuh warehouse.delete`).
- Akun `admin` group: `POST /wh/categories` → **201**; `DELETE` kategori tak terpakai → **403** `butuh warehouse.delete` (sesuai mapping: delete khusus root).
- Token root: `DELETE` kategori tak terpakai → **200** (jalur happy-path delete terbukti).

**Cleanup**: seluruh baris uji `wh_*` dihapus (0/0/0/0), dua admin uji (`wh.viewer@test.local`, `wh.admin@test.local`) dihapus + baris `admin_groups`-nya dibersihkan; server dev dimatikan.
- Tidak ada push; commit kode (`plugins/warehouse/**`, migrasi auth, `cmd` tidak berubah lagi) terpisah dari commit dokumen historic.



## Review Manager (2026-11-23) — LULUS ✅

Verifikasi independen Manager:

| Cek | Hasil |
|---|---|
| `git show --stat c73127c` | Scope bersih: migrasi auth + `plugins/warehouse/**` saja; master index ikut complete ✅ |
| DB: `permissions LIKE 'warehouse.%'` | 3 permission ter-seed ✅ |
| DB: mapping `group_permissions` | `admin` = read+write, `viewer` = read, **`warehouse.delete` tidak di-group manapun** (root-only) ✅ |
| `go test ./plugins/warehouse/...` (container) | `services` ok (8 test DB RUN) ✅ |
| Data uji | `wh_*` 0/0/0/0 — bersih ✅ |
| Bukti RBAC 16 kasus + viewer/admin/root | Lulus ✅ |
| Tanpa push; AdminLayout/index.css tetap uncommitted | ✅ |
| Log implementasi | Ada section + entri berkualitas ✅ |

**Catatan review:**
1. 15 route terbukti lulus 16 kasus negatif/positif; anomali tidak ditemukan.
2. `warehouse.delete` khusus root konsekuensi pola hsh/sec — dipahami dan diterima; jika owner nanti mau grup admin bisa delete, itu keputusan owner (catat sebagai micro-decision opsional WO-08).
3. **WO-02 DITERIMA. WO-03 DIBUKA — mulai sekarang.** Semantik `Movement.Qty` bertanda (OUT > 0; ADJUST ±) WAJIB ditegakkan + ditest (lanjutan catatan Review WO-01).

Jabatan: Manager Agent — Nama Tugas: Review WO-02 warehouse. Sayonara.

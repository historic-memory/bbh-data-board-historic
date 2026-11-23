---
title: spec
description: "SPEC modul Management Gudang (draft — menunggu keputusan G-1..G-8)"
status: draft
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
    - gudang
---

# SPEC — Modul Management Gudang

Status: **DRAFT**. Semua keputusan di bawah masih USULAN Manager; yang belum dikunci ditandai `?`.

## 1. Ringkasan

Listing stok sederhana gaya "mini market" (M-1..M-4, sumber `conversation.md`/`notes.md`): master barang + batch masuk + catatan keluar. **Tanpa sinkronisasi antar gudang** — tiap gudang/cabang mencatat sendiri (M-3). Umur barang di gudang dihitung dari tanggal masuk batch (M-2). Tidak jualan → fokus kebutuhan internal yayasan.

## 2. Skema (usulan, prefix `wh_`)

```
wh_items        master barang
  id (UUIDv7 TEXT), name (UNIQUE per instance), unit, granularity? (SPECIFIC|MIXED — G-5)
  note, is_active, timestamps

wh_warehouses   (HANYA bila G-2 = tabel)
  id, name, note, timestamps

wh_batches      masuk barang = sumber "umur di gudang"
  id, item_id → wh_items, warehouse_id → wh_warehouses (?G-2)
  qty_in, qty_remaining (diturunkan/dikunci transaksional), entered_at (tanggal masuk)
  expires_at? (G-4 — susu/perishable), note, created_by, timestamps

wh_movements    keluar + penyesuaian
  id, batch_id → wh_batches, qty (>0), direction (OUT|ADJUST)
  reason? (G-6), destination? (catatan gudang penerima — TIDAK sinkron, M-3)
  moved_at, created_by, timestamps
```

- Tidak ada tabel stok counter — stok = `SUM(qty_remaining)` batch aktif (anti desinkron).
- Mengikuti aturan user: **fitur = plugin** `plugins/warehouse`, tanpa kode di `internal/`.
- Tanpa `ref_*` (D-3 pola resident): enum statis di kode + CHECK constraint.
- Multi-DB: tanpa dimensi cabang eksplisit — batas data = `INSTANCE_KEY` (pola hsh/sec N-1). `?` konfirmasi.

## 3. Aturan bisnis (usulan)

- **Umur di gudang** = `today - entered_at` per batch; tampil per batch, terlama dulu (FIFO/FEFO).
- **Near-expiry** (jika G-4 ya): badge/alert `expires_at` mendekati/lewat — di summary.
- Movement OUT: `qty_remaining -= qty` dalam SATU transaksi; stok tak cukup → 409. `qty_remaining >= 0` CHECK.
- ADJUST (stock opname/koreksi): selisih positif/negatif, wajib `reason`.
- `destination` hanya catatan bebas (mis. "kirim ke gudang Jawa") — M-3: pihak penerima mencatat sendiri.
- Item spesifik (G-5): bila `granularity=SPECIFIC`, satu item = satu produk+varian (susu per merk+usia, popok per ukuran). Bukan penegakan keras — pembimbing UI/impor.

## 4. API (guard RBAC `wh.read/write/delete`, seed via migrasi plugin auth)

| Route | Isi |
|---|---|
| `/api/admin/wh/items` (+`/:id`) | CRUD master barang, search |
| `/api/admin/wh/warehouses` (+`/:id`) | CRUD gudang (jika G-2 tabel) |
| `POST /api/admin/wh/batches` | masuk barang |
| `POST /api/admin/wh/movements` | keluar/adjust (409 stok kurang) |
| `GET /api/admin/wh/summary` | stok per item, umur batch, near-expiry |

## 5. UI (SPA admin panel)

Rute `/warehouse/*`, nav grup "Gudang" (`warehouse.read`):
- Daftar item + stok terkini + badge near-expiry.
- Detail item = daftar batch (qty, umur hari, expiry, terlama dulu).
- Form masuk (batch baru), modal keluar (pilih batch, reason/destination).
- Pola komponen `src/components/ui/*`, PermissionGate level halaman.

## 6. Template & impor (jika G-8 = ikuti pola existing)

`schema/fields.yaml` → generator xlsx/csv (`GET /api/admin/wh-template`) → validator (`POST .../wh-template/validate`) → importer staging 2 fase (`POST .../wh-template/import`, tabel `wh_import_batches/rows`). Bisa dipisah WO lanjutan.

## 7. Rencana Work Order (usulan)

| WO | Isi |
|---|---|
| 01 | Skeleton plugin `wh` + migrasi (items/batches/movements ± warehouses) |
| 02 | Permission (migrasi plugin auth) + CRUD item (+warehouses) |
| 03 | Batch masuk + movement keluar + summary (transaksional, 409) |
| 04 | Generator template |
| 05 | Validator |
| 06 | Importer |
| 07 | UI admin |
| 08 | Swagger/README/Bruno |

## 8. Terbuka (→ notes.md G-1..G-8)

G-1 nama/prefix, G-2 entitas gudang, G-4 expiry, G-5 granularity flag, G-6 reason keluar, G-7 satuan, G-8 jalur data awal. Setelah dijawab owner: kunci D-1..D-x, SPEC `progress` → buat `work-order.md`.


---
title: spec
description: "SPEC modul Management Gudang — keputusan terkunci D-1..D-12"
status: progress
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
    - gudang
---

# SPEC — Modul Management Gudang

Status: **PROGRESS** — keputusan terkunci D-1..D-12 (owner "ok siapkan work order" 2026-11-23, usulan Manager disetujui + tambahan EX-1 excel & EX-2 AI chat). Sisa micro-decision di WO: O-1..O-3.

## 0. Keputusan terkunci (D-1..D-12)

| # | Keputusan |
|---|---|
| D-1 | Plugin `plugins/warehouse` (ID `warehouse`), prefix tabel `wh_`, permission `warehouse.read/write/delete` (seed via migrasi plugin auth), endpoint `/api/admin/wh/...` (G-1) |
| D-2 | Tabel `wh_warehouses`; 1 cabang (`INSTANCE_KEY`) BOLEH punya banyak gudang; `wh_batches.warehouse_id` NOT NULL (G-2) |
| D-3 | Struktur inti: kategori → produk (item) → batch masuk → movement keluar; umur = per batch (G-3) |
| D-4 | `expires_at` NULLABLE di batch; near-expiry tampil di summary + UI badge (G-4) |
| D-5 | TANPA sinkronisasi antar gudang (M-3); `destination` = catatan teks bebas |
| D-6 | TANPA flag granularitas — fleksibilitas spesifik/campur = keputusan penamaan produk (M-5, G-5) |
| D-7 | `direction` enum statis `OUT\|ADJUST` (CHECK); `reason` = teks bebas (WAJIB untuk ADJUST, opsional OUT); tanpa enum kategori keluar — sederhana sesuai M-1 (G-6) |
| D-8 | Satuan = dropdown REFERENSI di template (`fields.yaml`), DB simpan teks (pola hsh) (G-7) |
| D-9 | Jalur Excel LENGKAP: generator xlsx/csv → validator → importer staging 2 fase (EX-1, owner "sertakan excel") (G-8) |
| D-10 | Integrasi AI chat via pola plugin `ingest` (segmen tools): baca stok/summary via chat; tulis = usul → user setujui → terapkan (EX-2) |
| D-11 | UI SPA `/warehouse/*`, nav grup "Gudang", PermissionGate level halaman |
| D-12 | Seed data awal DITUNDA (pola hsh O-7) — kategori/produk dibuat user via UI/impor |

## 1. Ringkasan

Listing stok sederhana gaya "mini market" (M-1..M-4, sumber `conversation.md`/`notes.md`): master barang + batch masuk + catatan keluar. **Tanpa sinkronisasi antar gudang** — tiap gudang/cabang mencatat sendiri (M-3). Umur barang di gudang dihitung dari tanggal masuk batch (M-2). Tidak jualan → fokus kebutuhan internal yayasan.

## 2. Skema (usulan, prefix `wh_`)

```
wh_categories   kategori umum (Minyak Angin, Susu, Popok, Beras, Mainan, ...)
  id (UUIDv7 TEXT), name (UNIQUE per instance), note, is_active, timestamps

wh_items        produk — nama bebas per kategori; fleksibilitas spesifik/campur = penamaan (M-5, tanpa flag)
  id (UUIDv7 TEXT), category_id → wh_categories, name (UNIQUE per kategori),
  unit, note, is_active, timestamps

wh_warehouses   gudang per cabang — SATU cabang BOLEH punya BANYAK gudang
  id, name, note, timestamps

wh_batches      masuk barang = sumber "umur di gudang"
  id, item_id → wh_items, warehouse_id → wh_warehouses (NOT NULL)
  qty_in, qty_remaining (diturunkan/dikunci transaksional), entered_at (tanggal masuk)
  expires_at? (G-4 — susu/perishable), note, created_by, timestamps

wh_movements    keluar + penyesuaian
  id, batch_id → wh_batches, qty (>0), direction (OUT|ADJUST)
  reason? (G-6), destination? (catatan gudang penerima — TIDAK sinkron, M-3)
  moved_at, created_by, timestamps
```

- Tidak ada tabel stok counter — stok = `SUM(qty_remaining)` batch aktif (anti desinkron).
- Mengikuti aturan user: **fitur = plugin** `plugins/warehouse`, tanpa kode di `internal/`.
- Tanpa `ref_*`: enum statis di kode + CHECK constraint.
- Multi-DB: tanpa dimensi cabang eksplisit — batas data = `INSTANCE_KEY` (pola hsh/sec N-1).

## 3. Aturan bisnis (usulan)

- **Umur di gudang** = `today - entered_at` per batch; tampil per batch, terlama dulu (FIFO/FEFO).
- **Near-expiry** (jika G-4 ya): badge/alert `expires_at` mendekati/lewat — di summary.
- Movement OUT: `qty_remaining -= qty` dalam SATU transaksi; stok tak cukup → 409. `qty_remaining >= 0` CHECK.
- ADJUST (stock opname/koreksi): selisih positif/negatif, wajib `reason`.
- `destination` hanya catatan bebas (mis. "kirim ke gudang Jawa") — M-3: pihak penerima mencatat sendiri.
- **Kategori → produk (M-5, dikunci)**: `wh_categories` = kelompok umum; `wh_items` (produk) memakai `name` bebas dalam kategori. Minyak angin = 1 produk gabung semua merek; susu = produk terpisah per varian (SGM 0–1 th, Morinaga 1–2 th). Tidak ada flag granularitas — keputusan penamaan saat membuat produk.

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

## 6. Template & impor (D-9 — jalur Excel penuh)

`schema/fields.yaml` → generator xlsx/csv (`GET /api/admin/wh-template`) → validator (`POST .../wh-template/validate`) → importer staging 2 fase (`POST .../wh-template/import`, tabel `wh_import_batches/rows`). Pola hsh/sec.

## 7. Rencana Work Order

| WO | Isi |
|---|---|
| 01 | Skeleton plugin `wh` + migrasi inti (5 tabel) |
| 02 | Permission (migrasi auth) + CRUD master (categories/items/warehouses) |
| 03 | Batch masuk + movement keluar + summary (transaksional, 409) |
| 04 | Generator template xlsx/csv |
| 05 | Validator |
| 06 | Importer staging |
| 07 | UI admin `/warehouse/*` |
| 08 | Swagger/README/Bruno |
| 09 | Integrasi AI chat (segmen `wh` di plugin ingest) |

## 8. Micro-decision (O-1..O-3; SEMUA TERKUNCI 2026-11-23)

- **O-1 — TERKUNCI (owner, 2026-11-23): AUTO-CREATE.** Kategori & produk yang belum terdaftar saat impor **dibuat otomatis** (nama = teks di template; `is_active=true`; produk baru wajib punya kategori). Alasan: katalog lentur, ramah pengguna. Diwujudkan di WO-06 (`findOrCreateCategory`/`findOrCreateItem`) dan dilaporkan sebagai `new_masters`.
- **O-2 — TERKUNCI (owner, 2026-11-23): FAIL-CLOSED.** Gudang yang tidak cocok `wh_warehouses` **TIDAK** dibuat otomatis → **baris ditolak** (HTTP 200 + laporan per baris; `strict=1` → 400). Alasan: gudang = struktur operasional fisik, butuh registrasi sadar (bukan typo-tolerant). Diwujudkan di WO-06 (`resolveWarehouse`).
- **O-3 — TERKUNCI (owner, 2026-11-23): WHITELIST SAJA, TANPA TOOL BARU.** Lingkup tool chat gudang = baca (summary/batches/items) lewat whitelist `segment_read`; tulis (usul → setujui) hanya `POST /wh/batches` dan `POST /wh/movements`. Master data (kategori/produk/gudang) **TIDAK lewat chat**. Alasan: mencegah agent mengarang master + memakai ulang tool segmen yang sudah ada (tanpa tool baru). Diwujudkan di WO-09.

## 9. Integrasi AI chat (D-10, detail di WO-09)

Segmen `wh` ditambahkan ke whitelist plugin `ingest` (`segment_read` GET-only fail-closed; `segment_write_propose` + `apply_record_actions` usul→setujui→terapkan dengan `confirmed_by_user`). Data stok bukan PII → tanpa deidentify (beda dari visit), tapi whitelist fail-closed + audit tetap jalan. Prinsip D-11 topic ingest: strictness & gate tetap di backend Go.


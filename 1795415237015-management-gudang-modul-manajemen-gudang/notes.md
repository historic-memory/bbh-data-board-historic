---
title: notes
description: "Catatan meeting tim & keputusan terbuka modul Management Gudang"
status: progress
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
    - gudang
---

# Catatan Meeting — Management Gudang (2026-11-23)

Sumber: `conversation.md` (meeting owner × Ari).

## Poin Meeting (terkunci, dari owner)

- **M-1** — Tidak ada pakem aplikasi pergudangan existing yang cocok → dibuat sendiri, SEDERHANA.
- **M-2** — Konsep "mini market / Indomaret": listing stok saja. Barang ada berapa, tanggal masuk, **umur di gudang (berapa hari)**.
- **M-3** — Antar gudang **TANPA sinkronisasi**: gudang Bali cukup catat keluar, gudang Jawa cukup catat masuk ("kirim biasa").
- **M-4** — Granularitas: **"kategori = nama produk itu sendiri"**.
  - **Spesifik**: susu (SGM 0–1 th, Morinaga 1–2 th — dipisah per merk+usia), popok (merk boleh campur, tapi spesifik per ukuran/umur).
  - **Campur/umum**: minyak bayi, beras, mainan.
- **M-5 (owner, 2026-11-23 — penegasan M-4)** — Dua lapis: **kategori tetap ada + nama produk tetap ada**, tapi produk **fleksibel**:
  - "Minyak angin": orang taunya minyak angin — merek apa pun digabung SATU produk "Minyak Angin" (tanpa pecah merek).
  - "Susu": memang normal dipisah — produk spesifik per varian (SGM 0–1 th, Morinaga 1–2 th).
  - Pemecahan produk = keputusan saat membuat produk (penamaan), BUKAN flag/kode.

## Keputusan DIKUNCI

- **G-5 DIKUNCI (owner, M-5)**: struktur **dua lapis kategori → produk**; TANPA flag `granularity`. Fleksibilitas spesifik/campur ditentukan saat pembuatan produk (nama bebas per kategori). Skema: `wh_categories` + `wh_items(category_id, name, unit)`.

## Keputusan terbuka (G-1..G-8) — SEMUA DITUTUP (owner "ok siapkan work order" 2026-11-23)

| # | Keputusan |
|---|---|
| G-1 | **DIKUNCI**: plugin `warehouse`, prefix `wh` (usulan disetujui) |
| G-2 | **DIKUNCI**: tabel `wh_warehouses`; 1 cabang boleh banyak gudang; batch.warehouse_id NOT NULL |
| G-3 | **DIKUNCI**: master item + batch masuk + movement keluar; umur per batch |
| G-4 | **DIKUNCI**: `expires_at` nullable + near-expiry di summary/UI |
| G-5 | **DIKUNCI**: dua lapis kategori→produk, tanpa flag (M-5) |
| G-6 | **DIKUNCI**: `reason` teks bebas (wajib ADJUST), tanpa enum — sederhana (D-7) |
| G-7 | **DIKUNCI**: satuan dropdown REFERENSI fields.yaml, DB teks (D-8) |
| G-8 | **DIJAWAB owner (EX-1)**: jalur Excel penuh (template→validate→import) |

## Tambahan owner (EX-1, EX-2)

- **EX-1** — "sertakan excel juga": modul wajib punya jalur Excel (generator template + validator + importer).
- **EX-2** — "integrasi ke AI chat": modul gudang masuk lingkup plugin `ingest` (segmen tools, pola topic `1795107489593`) — WO-09.

## Design Plan draft

Terealisasi di `spec.md` (status `progress`, D-1..D-12) dan `work-order.md` + `WO-01..WO-09`.


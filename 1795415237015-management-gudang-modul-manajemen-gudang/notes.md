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

## Keputusan terbuka (G-1..G-8) — menunggu owner

| # | Pertanyaan | Usulan Manager |
|---|---|---|
| G-1 | Nama plugin & prefix tabel/permission/endpoint | `plugins/warehouse`, prefix `wh` |
| G-2 | Gudang = entitas DB (`wh_warehouses`) atau cukup per-instance `INSTANCE_KEY`? Satu cabang bisa >1 gudang? | Usulan: tabel warehouses (record keluar/masuk menyebut gudang mana). Owner menanyakan 1 cabang banyak gudang → usulan dipertegas: pakai tabel |
| G-3 | Struktur inti: master item + **batch masuk** (qty, tanggal, gudang) + transaksi keluar; umur per batch | Ya (usulan) |
| G-4 | Perlu **tanggal kadaluarsa** di batch (susu/perishable, FEFO/alert near-expiry)? | Usul keras: ya |
| G-5 | ~~Flag granularity?~~ → **DIKUNCI**: dua lapis kategori→produk, tanpa flag (lihat M-5) | Dikunci owner 2026-11-23 |
| G-6 | Transaksi keluar ke mana — dipakai cabang/anak, hangus/rusak? Perlu enum reason? | Usulan: `reason` bebas + enum kategori keluar |
| G-7 | Satuan: bebas atau dropdown REFERENSI seperti `hsh` (`fields.yaml`)? | Dropdown konsisten pola existing |
| G-8 | Data awal: template impor + validator + importer seperti modul lain, atau manual entry dulu? | Ikuti pola existing (template → validate → import) |

## Design Plan draft

Lihat `spec.md` (status `draft`). Setelah G-1..G-8 dijawab owner → kunci keputusan D-x di SPEC → buat `work-order.md`.


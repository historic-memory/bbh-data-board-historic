---
title: notes
description: "Catatan modul: list kebutuhan sekunder (elektronik, peralatan, mainan) per cabang BBH — analisis data mentah + keputusan."
status: progress
created: "2026-10-29"
---
# Notes — Kebutuhan Sekunder (ex "Elektronik")

## Konteks & perubahan scope
- Awalnya dibuka sebagai modul **"Kebutuhan Elektronik"** (2026-10-29).
- **Koreksi scope (owner, 2026-10-29)**: isinya **lebih dari sekadar elektronik** — mencakup
  **mainan** dan **peralatan tahan lama**. Label final: **"Kebutuhan Sekunder"**
  (Electronics, Toys & Durable Goods).
- **Batas dengan `hsh` (Kebutuhan Rumah Tangga)**: `hsh` = kebutuhan **habis pakai/pokok**
  (susu, makan, obat, kebersihan); modul ini = **barang tahan lama** (elektronik, peralatan, mainan).
- Dibangun sebagai **plugin baru** `plugins/sec/` (aturan user: fitur = plugin, bukan `internal/`).

## Keputusan terkunci (2026-10-29)
- **S-1 Nama/prefix**: judul topic "Kebutuhan Sekunder — modul list kebutuhan sekunder cabang";
  plugin `plugins/sec`; tabel `sec_*`; permission `sec.read/write/delete`; endpoint `/api/admin/sec/...`
  + template flat `/api/admin/sec-template`.
- **S-2 Tanpa dimensi cabang** (paralel `hsh` N-1): batas data = `INSTANCE_KEY`.
- **S-3 Periode**: entitas + riwayat snapshot (paralel `hsh` N-2).
- **S-4 Kategori**: **tabel** `sec_categories` (+ alias), bukan enum. Awal: `elektronik-peralatan`,
  `mainan-rekreasi`.
- **S-5 Nama item apa adanya** + tabel `sec_item_aliases` (paralel `hsh` N-4).
- **S-6 Atribut aditif**: `brand`, `specification`, `unit`, `est_price`, `note` di item;
  `status_code`, `quantity`, `unit_price`, `note` di entri.
- **S-7 Harga** di item (`est_price`) + override opsional per entri (`unit_price`).
- **S-8 Status** enum statis `URGENT|ENOUGH|UNKNOWN`; **file ini TIDAK punya legend warna** →
  semua entri awal = `UNKNOWN` (warna file tidak diinterpretasi).
- **S-9 Permission** `sec.*` di-seed via migrasi plugin `auth` (fail-closed bila lupa).
- **S-10 ID** UUIDv7 TEXT (dibuat di Go).
- **S-11 Template** digenerate dari `schema/fields.yaml`.
- **S-12 Importer** 2 fasa (staging → upsert), idempotensi `(period_code, item match)`.
- **S-13 Item nyasar** (Sepeda BMX) → kategori **Mainan & Rekreasi** (bukan dibuang).
- **S-14 Tanpa alur pemenuhan** (paralel `hsh` N-18).
- **S-15 Gambar produk (19 media tertanam)** diabaikan / di luar ruang lingkup.

## Analisis data mentah
- Aset: `List Kebutuhan Elektronik BBH - Updated 22 SEPT 26 2026.xlsx` (5.173.857 byte).
- **XLSX asli** (bukan `.xls` BIFF8) → `excelize` bisa baca langsung. Ukuran besar karena
  **19 gambar** tertanam di `xl/media/*`.
- 1 sheet `Sheet1`, dimensi `B2:F316`, 17 merge, **tanpa legend warna**.
- Header baris 3: `B=No`, `C=item`, `D=Merk/brand - Type`, `E=Spesifikasi/Specification`
  (multi-baris), `F=Harga/Price` (numerik, Rupiah). Baris 2 = judul board (merge `B2:F2`).
- **18 item**, semua berharga. Isi kolom C diwarnai tanpa legend (15× magenta `FF00FF`,
  1× navy `FF002060` item 12, 1× `FF66FF` item 17) → **tidak** ditafsirkan sebagai status.
- Kualitas data: item 11 & 12 menyelipkan merk di kolom C (D kosong), item 8 & 18 tanpa D,
  duplikasi teks di `E303==E304`, item 18 (Sepeda BMX) bukan elektronik.

## Langkah berikutnya
1. ✅ Susun `spec.md` (draft) — dieskalasi ke owner untuk review.
2. Setelah review → susun `work-order.md`.


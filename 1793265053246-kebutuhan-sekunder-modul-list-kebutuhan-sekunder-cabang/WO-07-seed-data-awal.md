---
title: "WO-07: Seed data awal (DITUNDA)"
description: "Impor berkas 'Updated 22 SEPT 26' + laporan normalisasi. DITUNDA sampai diminta owner."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - seed
  - deferred
---

# WO-07: Seed data awal — **DITUNDA**

## Status
**DITUNDA** (keputusan owner, mengikuti pola `hsh`: *"bikin sistem dulu aja"*).
Dikerjakan hanya setelah skema/master/template/importer siap **dan** owner meminta.

## Referensi
- SPEC: §Seed Awal, GAP-2, GAP-7.
- Aset: `List Kebutuhan Elektronik BBH - Updated 22 SEPT 26 2026.xlsx`.

## Rencana (bila diaktifkan)
1. **2 kategori**: `elektronik-peralatan`, `mainan-rekreasi`.
2. **1 periode**: `2026-09`.
3. **18 item + entri** (status awal `UNKNOWN`), semua `est_price` terisi:
   AC Midea 1PK & 1/2PK, Mesin Cuci Samsung 12kg, Dispenser Sanken, Water Heater Ariston,
   Air Purifier Xiaomi, CCTV EZVIZ, Exhaust Fan Plafon/Dinding/Fresh Air, Dehumidifier Notale,
   Dish Dryer NORGE, Cooker Hood Rinnai, Setrika Philips, Wall Power Fan Maspion, Smart TV Samsung 55",
   Dryer Gas Foggia, **Sepeda BMX Anak (→ `mainan-rekreasi`)**.
4. **Normalisasi (GAP-2/GAP-7)**:
   - pisahkan merk/varian dari nama item (item 11 & 12 — kolom D kosong);
   - lengkapi `brand` untuk item yang kosong (item 8, 18);
   - catat ejaan lama sebagai **alias**;
   - pindahkan item nyasar (Sepeda BMX) ke `mainan-rekreasi`.
5. Impor via endpoint WO-06; lampirkan **laporan** (dibuat/diperbarui/ditolak) + ringkasan normalisasi.

## Bukti yang harus dilampirkan (bila dikerjakan)
- Laporan impor batch; `SELECT count(*)` item/entri/alias; daftar normalisasi yang dilakukan.

## Bukti Pelaksanaan
_(diisi saat WO dikerjakan — saat ini DITUNDA)_

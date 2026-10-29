---
title: "WO-07: Seed data awal (DITUNDA)"
description: "Impor berkas 'List Kebutuhan RT BBH SEPT 19 OKT 26 d.xls' ke master+entri, termasuk laporan normalisasi. DITUNDA sampai sistem siap (N-15)."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - seed
  - deferred
---

# WO-07: Seed data awal — **DITUNDA (N-15)**

> **Keputusan owner 2026-10-29:** *"kita bikin sistem dulu aja"*. WO ini **tidak dikerjakan** sampai
> owner meminta. Isinya tetap disimpan sebagai rencana agar pelaksanaannya terstruktur.

## Referensi
- SPEC: N-15, §Seed Awal; analisis lengkap di `notes.md`.
- Aset: `List Kebutuhan RT  BBH  SEPT 19 OKT 26 d.xls` (XLS legacy BIFF8).
- Prasyarat: WO-06 (importer) selesai.

## Tugas (saat diaktifkan)
1. **Pra-konversi** `.xls` → `.xlsx` (LibreOffice di host: `soffice --headless --convert-to xlsx`);
   importer hanya menerima `.xlsx`/`.csv` (N-17).
2. Bentuk ulang data (kolom-daftar → tabel baris) menjadi template needhousehold sesuai **2 blok**:
   periode `2026-09` (r2–32) dan `2026-10-19` (r33–54).
3. **Normalisasi** (wajib, catat di laporan):
   - artikulasi bilingual tanpa separator (`Zaitun oil for Baby's bodyMinyak zaitun…`);
   - urutan terbalik (`Telon oil/Minyak telon`) & spasi ekor;
   - item nyasar antar kategori (`Hanging Air Freshener`, `Meja kursi ceria`);
   - duplikat makna obat (`PARATUSIN TAB` vs `Paratusin Tablet`) → alias.
   - typo kategori sudah dikoreksi di seed WO-01 (`Equpment`→`Equipment`).
4. Peta status warna: pink`FFFF99CC`→`URGENT`, navy`FF003366`→`ENOUGH`, tanpa fill→`UNKNOWN`.
5. Jalankan importer; verifikasi ±118 item/entri (urgen ≈81, enough ≈5).
6. Lampirkan **laporan normalisasi** (apa yang diubah & mengapa).

## Bukti yang harus dilampirkan (saat aktif)
- `.xlsx` hasil konversi (di luar repo) + hash.
- Ringkasan impor: jumlah periode/kategori/item/entri; hitungan per status.
- Laporan normalisasi (daftar perubahan nama/kategori + alias yang dibuat).

## Kriteria Selesai
- Data SEPT–OKT masuk sebagai 2 periode snapshot; laporan normalisasi terlampir.
- Tidak ada entri `URGENT`/`ENOUGH` yang salah status (sampling acak diverifikasi).

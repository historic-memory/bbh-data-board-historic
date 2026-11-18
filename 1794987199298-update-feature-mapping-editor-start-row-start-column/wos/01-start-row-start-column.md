---
title: "WO-01: Override start row & start column saat parsing berkas"
description: "Lembar kerja fitur override start row/column — latar belakang & desain lihat 01-analisa-parsing-header.md"
status: pending
created: "2026-11-18"
updated: "2026-11-18"
tags:
  - ingest
  - mapping
  - parsing
  - backend
---
# WO-01: Override start row & start column saat parsing berkas

> Latar belakang, perilaku saat ini, dan desain: lihat
> **`01-analisa-parsing-header.md`**. Lembar ini hanya daftar kerja.

## Lingkungan kerja (scope)

- **Backend Go**:
  - `ParseTable` + `parseCSV`/`parseXLSX` terima override `startRow`/
    `startCol` opsional (1-based; tanpa override = perilaku lama).
  - Override disimpan sebagai data sesi (profil jsonb append-only atau
    `ingest_uploads`) — pilih & konsisten.
  - Endpoint re-sample dari editor (`resample`) → `MappingView` baru;
    perubahan profil = transisi `revise` → `mapping_draft`.
- **Front** (`IngestMappingPage.tsx`): input `start_row`/`start_col`,
  preview ulang, simpan seperti biasa.

## Daftar kerja

- [ ] Tambah parameter override di `ParseTable` + kedua parser (CSV & XLSX).
- [ ] Validasi out-of-range → error jelas (fail-closed).
- [ ] Simpan override sebagai data sesi; bertahan antar re-parse.
- [ ] Endpoint re-sample + transisi `revise` (guard + audit tetap).
- [ ] UI: input override + tombol re-sample di editor mapping.
- [ ] Test: berkas judul-di-atas-header, berkas kolom-margin, tanpa
      override (kompatibel mundur).

## Kriteria selesai

1. Set `start_row` → header terbaca benar, sample & kolom sesuai.
2. Set `start_col` → kolom margin tidak masuk daftar mapping.
3. Tanpa override: perilaku parsing tidak berubah.
4. Override persisten per sesi, konsisten antara editor & chat.
5. Nilai di luar jangkauan ditolak dengan pesan jelas, tidak crash.
6. Audit & guard permission tetap tercatat.

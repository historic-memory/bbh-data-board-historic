---
title: "WO-01: Override start row & start column saat parsing berkas"
description: "Deteksi header otomatis gagal untuk berkas dengan judul di atas header / kolom margin — tambahkan override start_row & start_col di jalur parsing Go."
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

## Latar belakang

Deteksi header saat ini otomatis dan tidak ada override
(`plugins/ingest/services/table.go`):

- **CSV** (`parseCSV`): baris pertama = header. Tidak ada skip.
- **XLSX** (`parseXLSX`): melewati baris **kosong** di atas, baris
  non-kosong pertama dianggap header. Baris judul berisi teks
  (mis. "Laporan Kunjungan Bulan November") tetap salah terdeteksi.
- **Start column**: tidak ada — kolom nomor urut/margin kiri harus
  di-`skip` manual satu per satu di editor mapping.

Kasus berkas nyata BBH sering punya judul di atas header dan kolom
margin; parsing jadi salah atau mapping ribet.

## Tujuan

User dapat menentukan **baris awal header** (`start_row`) dan
**kolom awal data** (`start_col`) bila deteksi otomatis salah, lalu
sample & mapping dihitung ulang dari titik itu.

## Lingkungan kerja (scope)

- **Backend Go** (`plugins/ingest/services/table.go`, `mapping.go`,
  `uploads.go`):
  - `ParseTable` menerima parameter override `startRow`/`startCol`
    (opsional; tanpa override = perilaku sekarang).
  - Override adalah **data sesi** — disimpan (mis. di profil sesi atau
    `ingest_uploads`), bukan state UI. Re-parse konsisten untuk editor
    maupun chat.
  - Keputusan tetap fail-closed di Go; klien/agent tetap tidak menyentuh
    isi berkas.
- **Endpoint**:
  - Opsi A — override saat upload: `POST /sessions/:id/uploads`
    menerima field opsional.
  - Opsi B (direkomendasikan) — re-sample dari editor: endpoint
    re-parse (mis. `POST /sessions/:id/mapping/resample`) yang memicu
    `MappingView` dibangun ulang; nyambung ke kontrak state machine
    (ubah profil = `revise` → `mapping_draft`).
- **Front** (`IngestMappingPage.tsx`):
  - Input `start_row` / `start_col` di editor (atau preset cepat),
    preview ulang, lalu simpan seperti biasa.

## Kriteria selesai

1. Berkas dengan judul di atas header: set `start_row` → header terbaca
   benar, sample & kolom sesuai.
2. Berkas dengan kolom margin: set `start_col` → kolom margin tidak masuk
   daftar mapping.
3. Tanpa override: perilaku parsing tidak berubah (kompatibel mundur).
4. Override tersimpan per sesi dan bertahan di re-parse berikutnya.
5. Validasi: `start_row`/`start_col` di luar jangkauan → error jelas
   (fail-closed), tidak crash.
6. Audit tetap tercatat; guard permission tetap.

## Catatan teknis

- XLSX: override berarti mengabaikan skip-blank otomatis.
- CSV multi-baris quoted tetap harus benar (parsing RFC 4180).
- Urutan kolom setelah `start_col` mengikuti berkas; kolom sebelum
  `start_col` diabaikan seluruhnya.

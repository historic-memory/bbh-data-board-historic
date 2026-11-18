---
title: "Analisa parsing header & dasar fitur start row/column"
description: "Latar belakang fitur mapping editor: perilaku deteksi header saat ini, celahnya, dan desain override start row/start column"
status: draft
created: "2026-11-18"
updated: "2026-11-18"
tags:
  - ingest
  - mapping
  - parsing
  - arsitektur
---
# Analisa parsing header & dasar fitur start row/column

## Tujuan

Latar belakang untuk fitur **override start row & start column** di
mapping editor: apa perilaku sekarang, di mana celahnya, dan bagaimana
desainnya agar tetap sesuai arsitektur ingest strict (D-11, CP-2).

## Sumber

- Kode: `plugins/ingest/services/table.go` (parsing), `mapping.go`
  (MappingView), `handlers/agent_tools.go` (tool agent), `model/session.go`
  (state machine), `sub_app/webapp/src/pages/admin/ingest/`
  (editor mapping).
- Analisa terkait: topic `1794972055068` → `02-mekanisme-mapping-berkas.md`.
- Temuan (2026-11-18).

## Perilaku parsing saat ini

`ParseTable(filename, data, maxRows)` — jalur tunggal untuk CSV & XLSX:

- **CSV** (`parseCSV`):
  - Baris pertama = header, langsung.
  - Tidak ada mekanisme skip baris.
- **XLSX** (`parseXLSX`):
  - Melewati baris **kosong** di atas (loop `isBlankRow`), lalu baris
    non-kosong pertama dianggap header.
  - Baris judul yang berisi teks (mis. "Laporan Kunjungan Bulan
    November", judul + tanggal + logo teks) TIDAK dilewati — ia menjadi
    header salah.
- **Start column**: tidak ada konsepnya. Kolom pertama berkas selalu
  masuk; kolom nomor urut/margin kiri harus di-`skip` satu per satu di
  editor mapping.
- Sample yang dikirim ke editor/chat adalah ≤5 baris pertama SETELAH
  header (`sampleTable` → `MappingView`), selalu dideidentifikasi di Go.

## Celah

1. Berkas nyata BBH sering punya: judul laporan (1-5 baris) + header +
   kolom margin kiri + kadang baris total di bawah.
2. Saat ini user tidak punya cara mengatakan "header ada di baris ke-N,
   data mulai kolom ke-M". Hasilnya: mapping salah deteksi, atau ribet
   skip manual.
3. Override bila ada, harus jadi **data sesi** (persisten), bukan state
   UI, supaya re-parse editor maupun chat konsisten.

## Desain yang diusulkan

- `ParseTable` menerima override opsional `startRow` / `startCol`
  (0-based atau 1-based — putuskan konsisten, sarankan 1-based seperti
  spreadsheet supaya user paham).
  - Tanpa override = perilaku sekarang (kompatibel mundur).
  - `startRow` menggantikan deteksi otomatis: baris ke-`startRow`
    adalah header, data mulai baris berikutnya.
  - `startCol` memotong kolom: kolom < `startCol` diabaikan seluruhnya.
- Penyimpanan: profil sesi (jsonb, append-only per versi — sama seperti
  mapping) atau kolom di `ingest_uploads`. Keputusan teknis WO.
- Re-sample: editor mengirim override → Go re-parse → `MappingView`
  baru. Perubahan profil berarti transisi `revise` → `mapping_draft`
  (kontrak state machine tetap terjaga).
- Fail-closed tetap berlaku: parsing, validasi, dan audit semua di Go;
  klien/agent tidak pernah menyentuh isi berkas.

## Yang tidak berubah

- Ekstraksi tetap baca berkas ASLI dari storage lewat service layer
  `visit` — override hanya memengaruhi titik awal pembacaan tabel.
- Guard permission, audit, dan deidentify sample tidak berubah.
- Schema mapping v2 (kolom `{source,target,kind,...}`) tidak berubah —
  hanya sumber `header`/`rows`-nya yang berpindah titik awal.

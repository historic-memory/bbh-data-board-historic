---
title: "Desain UI/UX override start row & start column"
description: "Konsep antarmuka pemilihan titik awal header/data di editor mapping ingest"
status: draft
created: "2026-11-18"
updated: "2026-11-18"
tags:
  - ingest
  - mapping
  - ui-ux
  - frontend
---
# Desain UI/UX override start row & start column

## Tujuan

Desain antarmuka agar user dapat menunjuk baris header dan kolom data
awal secara visual di editor mapping — tanpa mengetik angka mentah.

## Mockup (sketsa sederhana)

```
┌─ Editor Mapping ──────────────────────────────────────────────────────┐
│  Sesi: 2d54fff5… · berkas: kunjungan-nov.xlsx                         │
│                                                                       │
│  [ Header: baris 4 · Kolom: B  ✕ reset → otomatis ]   (chip status)   │
│                                                                       │
│  Preview sample (klik nomor baris = pilih header,                     │
│                  klik huruf kolom = pilih kolom data pertama)         │
│                                                                       │
│        │  A     │  B        │  C         │  D       │                 │
│  ──────┼────────┼────────────┼─────────────┼───────────┤                 │
│  1     │ Laporan Kunjungan Bulan November ────── (redup, diabaikan)     │
│  2     │ Dinas Kesehatan ────────────────────── (redup, diabaikan)     │
│  3     │ No │ Nama        │ Telepon     │ Tanggal   │ ← kolom A redup │
│  [4]   │ 1  │ Ani         │ 0812…       │ 02-11     │ ← HEADER (aktif)│
│  5     │ 2  │ Budi        │ 0813…       │ 03-11     │   baris data    │
│                                                                       │
│  ⚠ Baris ini tidak tampak seperti header — pilih baris yang berisi    │
│    nama kolom.                        (pesan inline, bila salah-titik)│
│                                                                       │
│  ⛔ ada perubahan belum disimpan                    [Simpan] [Batal]  │
└───────────────────────────────────────────────────────────────────────┘
```

- `[4]` = baris terpilih sebagai header (highlight); baris 1-2 dan
  kolom A redup karena berada di luar titik awal.
- Chip status klik → reset ke "Deteksi: otomatis".
- Banner "belum disimpan" muncul hanya saat ada override aktif yang
  belum di-`PUT /mapping`.

## Prinsip

- **Show, don't tell**: user menunjuk di preview, bukan menebak nomor.
- Preview tetap terdeidentifikasi (sample ≤5 baris dari Go) — UI tidak
  menambah jalur PII.
- Fail-closed tetap terasa: input salah diberi umpan balik inline jelas.
- Konsisten pola editor yang sudah ada (`components/ui`, label Bahasa
  Indonesia, `PermissionGate`).

## Komponen

### 1. Preview spreadsheet dengan penala titik awal

- Preview sample memakai grid dengan **nomor baris** (kiri) dan
  **huruf kolom** (atas) — gaya spreadsheet (A, B, C… / 1, 2, 3…).
- **Klik nomor baris** → jadi `start_row` (kandidat header).
- **Klik huruf kolom** → jadi `start_col` (kolom data pertama).
- Baris header aktif di-highlight; kolom sebelum `start_col` ditampilkan
  redup/tercoret (diabaikan).

### 2. Chip status titik awal

- Di header editor: chip kecil `Header: baris 4 · Kolom: B` atau
  `Deteksi: otomatis` (bila tanpa override).
- Chip klik → panel reset / ubah; **reset = kembali ke perilaku
  otomatis** (tanpa override).

### 3. Alur interaksi

1. Preview salah (header salah deteksi) → user klik nomor baris yang
   benar → preview re-render.
2. Re-render client-side dulu bila memungkinkan; bila butuh data baru
   dari Go (sample baru setelah titik awal) → panggil re-sample
   (debounce, dengan indikator loading).
3. Perubahan belum disimpan → banner "ada perubahan belum disimpan"
   (pola draft editor yang sudah ada).
4. Simpan → alur `PUT /mapping` + confirm seperti biasa; perubahan
   profil = transisi `revise` → `mapping_draft`.

### 4. Umpan balik salah-titik

- `start_row` mengarah ke baris kosong / baris total → pesan inline di
  bawah chip: "Baris ini tidak tampak seperti header — pilih baris yang
  berisi nama kolom."
- `start_col` di luar jangkauan berkas → pesan inline + tombol reset.
- Semua error dari Go (fail-closed) ditampilkan apa adanya, tidak
  disembunyikan.

### 5. Aksesibilitas

- Penanda baris/kolom fokus-able keyboard + `aria-label`
  (`"Jadikan baris ini header"`).
- Ukuran target klik ≥ sel grid standar; sel tabel mapping yang sudah
  ada tidak berubah perilaku.

## Yang tidak berubah

- Editor kolom (source → target, kind, is_encrypt, skip) tetap seperti
  sekarang — fitur ini hanya memindahkan TITIK AWAL data.
- Jalur chat (`get_mapping_guess`) otomatis ikut benar karena membaca
  sample dari Go yang sama.

## Terbuka (perlu keputusan)

1. Re-render sample: murni client-side (pangkas dari sample yang sudah
   ada) vs selalu minta Go re-sample? (perlu cek batas ≤5 baris.)
2. Apakah baris total di bawah berkas ikut didukung (`end_row`)? —
   sengaja di luar scope iterasi ini.

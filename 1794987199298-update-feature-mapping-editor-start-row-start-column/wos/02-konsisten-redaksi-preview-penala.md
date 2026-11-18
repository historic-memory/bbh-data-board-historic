---
title: "WO-02: Konsisten redaksi preview penala dengan panel kolom"
description: "Preview penala titik awal memakai mask generik berbeda dari SAMPLE panel kolom — samakan kebijakan (opsi B) agar header terbaca tanpa jalur PII baru"
status: pending
created: "2026-11-18"
updated: "2026-11-18"
tags:
  - ingest
  - mapping
  - ui-ux
  - backend
---
# WO-02: Konsisten redaksi preview penala dengan panel kolom

> Latar: report Visual Agent 2026-11-18 + analisa Manager (dikonfirmasi ke
> kode). Temuan: dua kebijakan redaksi berbeda di satu halaman.
> Keputusan produk: **opsi B** (samakan dengan panel bawah + ringkasan bentuk
> nilai untuk kolom tak dikenal).

## Masalah

- Grid preview penala (`buildPreview`) memask SEMUA sel non-katalog dengan
  HMAC `generic` — terlepas dari kind kolom.
- Panel kolom (`buildColumns` → `Sample`) meredaksi per-kolom: kind `none`/
  enum tampil apa adanya (`O+ … AB+`), PII dimask.
- Akibat: setelah Reset, grid atas penuh `anon-*` sedangkan panel bawah
  terbaca — tampak tidak konsisten. Untuk berkas dengan header di luar
  katalog (mis. "Data Bayi Yayasan"), preview TIDAK PERNAH terbaca → user
  menyangka data dummy, sulit menentukan baris header.

## Solusi (opsi B, disetujui)

Preview memakai kebijakan redaksi yang SAMA dengan panel kolom:

1. **Kind per kolom dari mapping tersimpan** (`previewKindForHeader` sudah
   ada) → `none`/enum tampil asli, PII dimask per-kolom — persis seperti
   `buildColumns`.
2. **Kolom yang belum dikenal** (header salah terdeteksi / guess → skip):
   jangan kirim isi sel; kirim **ringkasan bentuk nilai** per kolom, mis.
   `{"type":"text","count":12,"distinct":9}` atau teks ringkas
   `3 teks, 2 angka` — user tetap bisa membedakan baris header vs data
   tanpa bocor isi.
3. **Baris yang dipilih sebagai header** tampil apa adanya (isi header =
   label kolom, risiko rendah) — ini yang perlu dikenali user.
4. `suggested_row` tetap; hitungan kecocokan katalog tidak berubah.

## Daftar kerja

- [ ] `buildPreview`: terima kind per kolom dari doc (kolom dikenal) +
      fallback ringkasan bentuk nilai untuk kolom tak dikenal.
- [ ] `TablePreview`/`PreviewRow`: tambah field ringkasan per kolom
      (mis. `summary`), pertahankan `suggested_row`.
- [ ] Hapus mask HMAC blanket untuk kolom yang kind-nya sudah diketahui.
- [ ] Front `OriginTuner`: tampilkan ringkasan bentuk nilai di sel kolom
      tak dikenal; tampilkan baris header terpilih apa adanya.
- [ ] Test: kind `none` tampil asli; kolom tak dikenal → ringkasan tanpa
      isi sel; baris header terpilih tampil asli.

## Kriteria selesai

1. Setelah Reset/klik, grid preview dan panel kolom memakai kebijakan
   redaksi yang sama (nilai `O+`-seperti tampil konsisten di keduanya).
2. Untuk berkas header di luar katalog: user tetap bisa memilih baris
   header (ringkasan bentuk + baris header terpilih terbaca).
3. Tidak ada jalur PII mentah baru: isi sel kolom tak dikenal tidak
   pernah dikirim ke browser.
4. `go test`, `tsc`, build, dan verifikasi browser lolos.
5. Audit & guard tidak berubah.

## Catatan

- Opsi A (tampil apa adanya untuk admin) ditolak: membuka jalur PII baru.
- Opsi C (perluas katalog `fields.yaml`) tidak menutup kasus header liar;
  bisa jadi tindak lanjut terpisah bila template berulang dipakai.

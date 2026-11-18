---
title: 'WO-03: Modal konfirmasi sebelum replace lampiran sesi di halaman chat'
description: Upload berkas baru di halaman chat langsung menghapus berkas lama tanpa konfirmasi — tambah modal confirm destructive (AlertDialog) dengan info berkas lama & baru
status: complete
created: "2026-11-18"
updated: "2026-11-18"
tags:
    - ingest
    - chat
    - ui-ux
    - frontend
---
# WO-03: Modal konfirmasi sebelum replace lampiran sesi di halaman chat

> Keputusan produk (dengan user, 2026-11-18): upload berkas baru yang akan
> **menghapus berkas lama** harus melewati modal konfirmasi. Text hint di
> panel bukan guard yang cukup — replace bersifat destructive.

## Masalah

- Panel upload (`sub_app/webapp/src/pages/admin/ingest/IngestUploadPanel.tsx`)
  langsung memanggil `uploadIngestFile` pada `onFiles`; tidak ada interupsi.
- Backend (`plugins/ingest/services/uploads.go`, `attachUploadToSession`)
  menghapus upload lama + objek storage saat sesi masih state `uploaded` —
  berkas lama benar-benar hilang tanpa konfirmasi.
- Text hint "Berkas baru menggantikan berkas sesi ini…" tidak dibaca user.

## Solusi (disetujui)

1. **Modal konfirmasi** muncul HANYA bila `uploads.length > 0` saat user
   memilih berkas baru (replace). Upload pertama tanpa modal.
2. Isi modal:
   - Nama + ukuran berkas **lama** yang akan terhapus.
   - Nama + ukuran berkas **baru**.
   - Pesan: berkas lama akan dihapus dan tidak bisa dikembalikan.
   - Tombol: **Batal** (aman) dan **Ganti berkas** (destructive, styling
     danger).
3. **Label tombol panel**: setelah ada berkas terunggah, "Pilih file" →
   "Ganti berkas" agar ekspektasi selaras.
4. **State guard tetap di server**: bila sesi sudah melewati `uploaded`,
   server tetap menolak 409 ("buat sesi baru") — modal hanya untuk alur
   replace yang diizinkan server. Pesan 409 tampil di area error panel
   seperti sekarang.
5. Komponen: pakai modal/dialog pattern yang sudah ada di webapp
   (shadcn/ui AlertDialog atau setara yang sudah dipakai), jangan pola baru.

## Daftar kerja

- [x] Tambah state pending-file di `IngestUploadPanel` + AlertDialog
      konfirmasi replace (nama lama/baru, tombol Batal & Ganti).
- [x] Label tombol dinamis: "Pilih file" (kosong) / "Ganti berkas"
      (sudah ada lampiran).
- [x] Modal HANYA untuk kondisi replace; upload pertama langsung jalan.
- [x] Pesan 409 "sesi sudah di tahap ..." tetap tampil di error panel.
- [x] Test Playwright: replace dengan modal (batal & lanjut), upload
      pertama tanpa modal, replace setelah sesi diperkenalkan → 409.

## Kriteria selesai

- Berkas lama tidak pernah terhapus tanpa konfirmasi eksplisit user.
- Upload pertama tidak terganggu modal.
- Backend tidak berubah (guard state tetap server-side).
- Test baru lulus; test ingest yang ada tetap hijau.

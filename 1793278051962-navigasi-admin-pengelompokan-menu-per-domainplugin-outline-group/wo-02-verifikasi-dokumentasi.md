---
title: "WO-02: Verifikasi & dokumentasi navigasi outline group"
description: "Memverifikasi perilaku sidebar hasil WO-01 (filter izin, collapsible, persistensi, mobile) dan mendokumentasikan cara menambah domain/plugin baru di registry."
status: planned
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - admin-panel
  - ui
  - navigation
  - verification
---

# WO-02: Verifikasi & Dokumentasi

## Referensi
- SPEC `spec.md` (khususnya N-5, N-7, N-10, N-11).
- Hasil WO-01 (`AdminLayout.tsx` + `src/navigation/menu.ts`).

## Tujuan
Membuktikan perilaku sidebar hasil WO-01 benar, dan mendokumentasikan cara menambah domain/plugin
baru sehingga sifat "bisa expand" dapat dipakai orang lain tanpa membaca kode layout.

## Deliverable
1. **Verifikasi perilaku** (checklist, dengan bukti singkat):
   - Filter izin per group & per item (uji dengan user viewer tanpa sebagian izin).
   - Group kosong tidak dirender.
   - Collapsible: buka/tutup + persist setelah reload; group rute aktif terbuka.
   - `document.title` benar di beberapa rute.
   - Drawer mobile berfungsi.
2. **Dokumentasi singkat** (di `sub_app/webapp/README.md` atau komentar registry):
   - Cara menambah domain/plugin baru: contoh langkah (daftarkan rute di `App.tsx` + tambahkan
     `NavGroup` di `menu.ts`).
   - Peringatan N-10: menyembunyikan menu bukan kontrol keamanan.

## Kriteria Penerimaan (AC)
- **AC-1** Checklist verifikasi terisi dengan hasil nyata (bukan asumsi), termasuk ≥1 skenario izin terbatas.
- **AC-2** Terdapat dokumentasi "cara menambah domain" yang bisa diikuti tanpa membuka `AdminLayout.tsx`.
- **AC-3** `npm run build` tetap sukses setelah perubahan dokumentasi (bila menyentuh kode).
- **AC-4** Tidak ada regresi: rute lama tetap dapat diakses lewat sidebar.

## Bukti yang Diminta
- Checklist hasil verifikasi (dengan perintah/nilai aktual).
- Tautan/ringkasan dokumentasi yang ditambahkan.

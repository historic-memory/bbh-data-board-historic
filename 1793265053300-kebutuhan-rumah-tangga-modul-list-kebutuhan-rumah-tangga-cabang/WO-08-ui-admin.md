---
title: "WO-08: UI admin panel (kategori, item, periode, entri)"
description: "Halaman admin untuk mengelola master & entri needhousehold: kategori, item (+alias), periode, matriks entri, serta unduh template dan impor."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - ui
  - admin-panel
---

# WO-08: UI admin panel

## Referensi
- SPEC: §UI (admin panel), §API & Permission.
- Pola acuan: `sub_app/webapp/src/pages/admin/residents/*`, `api/facilities.ts`, `lib/apiError.ts`,
  `lib/fields.ts`, `components/ui/*` (Tailwind v4), `layouts/AdminLayout.tsx`, `auth/RequireAuth`.

## Tugas
1. **API client** `src/api/needhousehold.ts` (kategori, item, alias, periode, entri, template,
   validate/import) — error via `toApiFailure()`/`apiError.ts` (pesan Indonesia).
2. **Halaman** di bawah rute ter-guard:
   - `/needs/categories` — list + form (code, name, sort_order, aktif).
   - `/needs/items` — list + filter kategori + search + pagination; form item
     (nama, kategori, satuan dari REFERENSI, harga, aktif, catatan); panel **alias** per item.
   - `/needs/periods` — list + form + aksi "jadikan aktif".
   - `/needs/entries` — **tampilan matriks** per periode: baris = item, kolom status/jumlah/harga/catatan;
     aksi massal ubah status (mencerminkan cara kerja berkas asli).
   - Tombol **Unduh Template** (xlsx/csv) + **Impor** (unggah → tampilkan laporan validasi/impor).
3. **Guard** di level halaman memakai `hasPermission` (`needhousehold.read/write/delete`);
   tombol mutasi disembunyikan bila tak berizin (jangan render tombol mati).
4. **Format**: `lib/format.ts` (`timeZone:'UTC'` untuk tanggal), `lib/fields.ts` mirror daftar enum
   status + satuan dari `fields.yaml` (sertakan header sinkron + grep drift, pola WO-04 admin panel).
5. **Nav** admin: tambah entri "Kebutuhan RT" difilter `needhousehold.read`.

## Bukti yang harus dilampirkan
- `cd sub_app/webapp && npm run build` hijau; `npm run dev` (proxy Vite) berfungsi.
- Screenshot/DOM: list kategori, form item + alias, matriks entri, modal impor dengan laporan.
- Verifikasi guard: viewer tidak melihat tombol mutasi; akses langsung halaman write → pesan 403.
- Uji unduh template & impor dari UI (round-trip) → laporan tampil.

## Kriteria Selesai
- CRUD master + periode + entri berfungsi dari UI; matriks entri meniru alur berkas asli.
- Unduh/impor template berjalan end-to-end; pesan error berbahasa Indonesia.
- Tidak ada tombol mati; bundle tetap stateless (tanpa URL backend absolut).

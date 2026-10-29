---
title: "WO-08: UI admin panel"
description: "UI admin: Kategori, Item (+alias), Periode, Entri (matriks), tombol Unduh Template + Impor."
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
- SPEC: §UI (admin panel).
- Pola acuan: `plugins/hsh` WO-08 (`sub_app/webapp/src/pages/admin/needs/*`, `src/api/hsh.ts`,
  `src/lib/hshFields.ts`), navigasi `src/navigation/menu.ts` (grup "Kebutuhan").

## Tugas
1. **API client** `src/api/sec.ts` (pola `src/api/hsh.ts`).
2. **Mirror field** `src/lib/secFields.ts` (mirror `fields.yaml` sec — enum/units/header, catat tanggal sinkron).
3. **Halaman** di `src/pages/admin/secondary/`:
   - `Kategori` — CRUD ringan (code, name, sort_order, is_active).
   - `Item` — list + filter kategori + search; form item (nama, kategori, merk, spesifikasi, satuan,
     harga, aktif, catatan); pengelolaan **alias**.
   - `Periode` — CRUD periode + aksi "jadikan aktif".
   - `Entri` — **tampilan matriks** per periode (baris=item, kolom=status/jumlah/catatan), aksi massal ubah status.
   - Tombol **Unduh Template** + **Impor** (unggah berkas → tampil laporan validasi).
4. **Routing & navigasi**: daftarkan rute `/needs-secondary/*` (atau setara) di `App.tsx`;
   tambah entri di `src/navigation/menu.ts` **grup "Kebutuhan"** difilter `sec.read`.
5. **Guard**: `PermissionGate` di level halaman; sembunyikan tombol mutasi via `hasPermission`.
   Semua pesan UI dalam Bahasa Indonesia. Tanggal pakai `timeZone:'UTC'`.

## Bukti yang harus dilampirkan
- `npm run build` hijau (tsc -b + vite).
- Uji dev (Playwright :5173): CRUD kategori/item/alias, periode, entri matriks, unduh/unggah template.
- Uji izin: viewer (`sec.read`) melihat list tapi tombol mutasi tersembunyi; non-read 403 → halaman guard.

## Kriteria Selesai
- Semua layar berfungsi terhadap API `sec`; navigasi grup "Kebutuhan" menampilkan entri sec.
- Tidak ada perubahan pada halaman/plugin lain di luar yang diperlukan.

## Bukti Pelaksanaan
_(diisi saat WO dikerjakan)_

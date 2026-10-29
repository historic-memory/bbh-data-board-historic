---
title: "WO-08: UI admin panel"
description: "UI admin: Kategori, Item (+alias), Periode, Entri (matriks), tombol Unduh Template + Impor."
status: complete
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

**Berkas yang ditambahkan/diubah**
- BARU `sub_app/webapp/src/api/sec.ts` — klien API plugin `sec` (kembar `hsh.ts`), semua path
  `/api/admin/sec*`; tipe `Item`/`ItemInput` diberi `brand` + `specification` (beda dari hsh),
  `ImportRow` juga memuat keduanya.
- BARU `sub_app/webapp/src/lib/secFields.ts` — mirror `plugins/sec/schema/fields.yaml`
  (3 status `SEC_STATUSES`, 8 satuan `SEC_UNITS`, tone badge, `formatRupiah`/`formatQty`,
  `SEC_TEMPLATE_HEADERS` 10 kolom). Header mencatat sumber + tanggal sinkronisasi.
- BARU `sub_app/webapp/src/pages/admin/secondary/` — `CategoryListPage`, `CategoryFormModal`,
  `ItemListPage`, `ItemFormModal`, `AliasModal`, `PeriodListPage`, `PeriodFormModal`,
  `EntryMatrixPage`, `TemplatePage`, `PermissionGate`, `ConfirmDeleteModal`.
- UBAH `sub_app/webapp/src/App.tsx` — 5 rute `/needs-secondary/*`.
- UBAH `sub_app/webapp/src/navigation/menu.ts` — grup baru `kebutuhan-sekunder`
  ("Kebutuhan Sekunder") difilter `sec.read`; grup `kebutuhan` (hsh) tidak diubah.

**Penyesuaian domain `sec`**
1. Form item menambah field **Merk/Brand** dan **Spesifikasi** (dipakai importer sebagai kunci
   pencocokan alternatif); tabel item memberi kolom gabungan "Merk / Spesifikasi".
2. Kotak cari item berlabel "Cari nama, alias, atau merk" (backend memang mencari ke
   name/brand/specification/alias).
3. Teks domain diganti ke kebutuhan sekunder (barang tahan lama) dan contoh placeholder
   (mis. "Kipas Angin Berdiri", "kipas angin berdiri"); tidak ada sisa istilah hsh.

**Verifikasi**
- `npm run build` (tsc -b + vite) **hijau**: 160 modul, `dist/assets/index-DbY84iUM.js`
  478,06 kB; tidak ada error TypeScript.
- Uji dev `:5173` (Playwright, akun root):
  - Navigasi menampilkan grup "Kebutuhan Sekunder" dengan 5 entri; judul dokumen berubah
    (`Item · BBH Data Board — Admin`, `Entri Kebutuhan Sekunder · …`).
  - **Item**: list memuat kolom Merk/Spesifikasi + filter kategori dari API; modal Ubah
    prefill brand/spesifikasi; ubah brand "Cosmos" → "Cosmos II" tersimpan (toast "Item
    diperbarui.", tabel langsung menampilkan "Cosmos II"); modal Alias menampilkan daftar
    alias dan penambahan alias berhasil (toast "Alias ditambahkan.").
  - **Periode**: daftar periode + tombol Entri/Ubah/Hapus; "jadikan aktif" bekerja (hanya satu
    aktif); tautan Entri membawa `?period=2026-10`.
  - **Entri (matriks)**: ringkasan (Total/Urgent/Cukup/Tidak diketahui) + baris per item dengan
    dropdown status, jumlah, harga satuan (placeholder harga referensi), catatan; Simpan baris
    berhasil (toast "Baris disimpan."); empty-state memandu bila belum ada periode.
  - **Template**: unduh XLSX/CSV, unggah berkas (CSV 2 baris) → **Uji kering** menampilkan
    laporan (Total 2 / Ditolak 1, alasan "kategori tidak dikenal") tanpa menulis master
    (toast "Uji kering selesai — tidak ada data master yang ditulis."); centang dilepas →
    tombol berubah **Impor sekarang** → 1 dibuat, 0 diperbarui, status "Tersimpan"; **Muat
    audit** menampilkan baris staging (valid/tidak valid + alasan).
- Uji izin:
  - Viewer `sec.read` (grup uji `secviewer`): list `categories/items/periods/entries` = 200;
    `POST` = 403. UI: tombol "+ Tambah item", "Ubah", "Hapus" tersembunyi, hanya "Alias" tampil;
    halaman Template menampilkan "Akses ditolak (403) — Impor membutuhkan izin `sec.write`",
    tombol unduh tetap tersedia; grup hsh/Data Warga/Administrasi ikut tersembunyi karena izin
    tidak dimiliki.
  - Admin tanpa `sec.read` (grup uji `noperm`): halaman Item menampilkan guard
    "Akses ditolak (403) — Halaman ini membutuhkan izin `sec.read`"; grup "Kebutuhan Sekunder"
    tidak dirender.
- Data & akun uji dibersihkan setelah verifikasi: `sec_*` kembali 2 kategori / 0 item /
  0 periode / 0 entri / 0 batch; admin `secviewer@test.local`, `noperm@test.local` beserta
  grup ujinya dihapus (grup tersisa: `admin`, `root`, `viewer`).

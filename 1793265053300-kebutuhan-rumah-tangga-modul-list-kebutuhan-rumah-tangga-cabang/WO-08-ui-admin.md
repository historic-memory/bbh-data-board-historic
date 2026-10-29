---
title: "WO-08: UI admin panel (kategori, item, periode, entri)"
description: "Halaman admin untuk mengelola master & entri hsh: kategori, item (+alias), periode, matriks entri, serta unduh template dan impor."
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
- SPEC: §UI (admin panel), §API & Permission.
- Pola acuan: `sub_app/webapp/src/pages/admin/residents/*`, `api/facilities.ts`, `lib/apiError.ts`,
  `lib/fields.ts`, `components/ui/*` (Tailwind v4), `layouts/AdminLayout.tsx`, `auth/RequireAuth`.

## Tugas
1. **API client** `src/api/hsh.ts` (kategori, item, alias, periode, entri, template,
   validate/import) — error via `toApiFailure()`/`apiError.ts` (pesan Indonesia).
2. **Halaman** di bawah rute ter-guard:
   - `/needs/categories` — list + form (code, name, sort_order, aktif).
   - `/needs/items` — list + filter kategori + search + pagination; form item
     (nama, kategori, satuan dari REFERENSI, harga, aktif, catatan); panel **alias** per item.
   - `/needs/periods` — list + form + aksi "jadikan aktif".
   - `/needs/entries` — **tampilan matriks** per periode: baris = item, kolom status/jumlah/harga/catatan;
     aksi massal ubah status (mencerminkan cara kerja berkas asli).
   - Tombol **Unduh Template** (xlsx/csv) + **Impor** (unggah → tampilkan laporan validasi/impor).
3. **Guard** di level halaman memakai `hasPermission` (`hsh.read/write/delete`);
   tombol mutasi disembunyikan bila tak berizin (jangan render tombol mati).
4. **Format**: `lib/format.ts` (`timeZone:'UTC'` untuk tanggal), `lib/fields.ts` mirror daftar enum
   status + satuan dari `fields.yaml` (sertakan header sinkron + grep drift, pola WO-04 admin panel).
5. **Nav** admin: tambah entri "Kebutuhan RT" difilter `hsh.read`.

## Bukti yang harus dilampirkan
- `cd sub_app/webapp && npm run build` hijau; `npm run dev` (proxy Vite) berfungsi.
- Screenshot/DOM: list kategori, form item + alias, matriks entri, modal impor dengan laporan.
- Verifikasi guard: viewer tidak melihat tombol mutasi; akses langsung halaman write → pesan 403.
- Uji unduh template & impor dari UI (round-trip) → laporan tampil.

## Kriteria Selesai
- CRUD master + periode + entri berfungsi dari UI; matriks entri meniru alur berkas asli.
- Unduh/impor template berjalan end-to-end; pesan error berbahasa Indonesia.
- Tidak ada tombol mati; bundle tetap stateless (tanpa URL backend absolut).

## Bukti Pelaksanaan
- **Commit:** `d554950` — `feat(hsh): UI admin kebutuhan RT — master, periode, matriks entri, template/impor (WO-08)` (15 file, +3379).
  `src/api/hsh.ts` (479 baris), `src/lib/hshFields.ts`, 11 berkas di `src/pages/admin/needs/`,
  `src/App.tsx`, `src/layouts/AdminLayout.tsx`. `dist/` tetap gitignored.
- **Build:** `cd sub_app/webapp && npm run build` → **hijau** (`tsc -b` + vite, 146 modul,
  bundle `index-*.js` 424,93 kB / 126,31 kB gzip).
- **Dev server:** `npm run dev` di :5173 dengan proxy `/api` → :8082 berjalan; seluruh halaman diuji
  lewat browser pada `http://localhost:5173/app/...` (bukan build statis).
- **Alur yang terbukti (DOM):**

  | Uji | Hasil |
  |---|---|
  | `/needs/categories` list | 7 kategori seed tampil (kode, nama, urutan, status) |
  | Tambah kategori | modal → simpan → toast "Kategori ditambahkan." → "8 dari 8 kategori" |
  | Hapus kategori | dialog konfirmasi → "Kategori dihapus." → kembali 7 |
  | Tambah item | form nama/kategori/satuan REFERENSI/harga/catatan → baris `Susu formula tahap 1 · Milk / Susu · kaleng · Rp 95.000` |
  | Panel alias | item `Biskuit bayi` → tambah alias `susu formula 1` → muncul di tabel + toast |
  | Tambah periode | `2026-10` + label + rentang + centang aktif → status `Aktif` |
  | Matriks entri | baris = item, kolom status/jumlah/harga/catatan; simpan baris → ringkasan `Total 1 · Urgent 1`, tombol Hapus muncul |
  | Aksi massal | pilih semua + `Stok cukup` → "Status Stok cukup diterapkan ke 2 baris."; ringkasan `2 / 0 urgent / 2 cukup`; jumlah 3 & catatan baris 1 dipertahankan |
  | Unduh template (UI) | `GET /api/admin/hsh-template?format=csv` → **200**, `text/csv`, `filename="template_hsh_v1.csv"` |
  | Uji kering (dry-run) | `ui-import.csv` 3 baris → status **staging**, `total 3 / dibuat 0 / ditolak 1`; baris 4 ditolak "Kategori: kategori tidak dikenal…" |
  | Impor sungguhan | `Tersimpan`, `dibuat 2`, baris: "periode dibuat; item baru dibuat + alias" / "item baru dibuat + alias" |
  | Audit batch | tombol "Muat audit" memuat counter batch + 3 baris staging (kategori, status, valid, alasan) |

- **Guard izin (akun uji `hsh.ui@test.local`, grup `hsh_viewer_ui` dengan hak tunggal `hsh.read`):**
  - Sidebar hanya menampilkan menu `hsh.*` (+ Dashboard); menu `resident`/`admin`/`rbac` tidak muncul.
  - `/needs/categories`: tanpa tombol Tambah/Ubah/Hapus (`hasTambah=false hasUbah=false hasHapus=false`).
  - `/needs/items`: tanpa Tambah/Ubah, panel **Alias** tetap dapat dibuka (baca saja).
  - `/needs/entries`: input status `disabled`, tanpa tombol Simpan, tanpa bilah aksi massal.
  - `/needs/template`: seksi impor diganti **Alert 403** "Impor membutuhkan izin `hsh.write`";
    unduh template tetap tersedia. API-nya sendiri tetap fail-closed (403 `butuh hsh.write`, sudah
dibuktikan di WO-06).
- **Perbaikan yang ditemukan saat uji:** sidebar tidak bisa scroll ketika menu bertambah (item
  terakhir di luar viewport) → `aside` dibuat flex-column dengan `nav` `flex-1 overflow-y-auto`.
  Tanpa ini, tiga menu terakhir tidak dapat diklik sama sekali.
- **Deviasi kecil (disengaja):** mirror enum/satuan diletakkan di **`src/lib/hshFields.ts`**,
  bukan ditambahkan ke `src/lib/fields.ts`. `fields.ts` menyatakan dirinya MIRROR dari
  `plugins/resident/schema/fields.yaml`; mencampur dua sumber di satu berkas akan membuat header
  sinkronisasi itu tidak benar. Header baru memuat instruksi grep drift yang sama polanya.
- **Kontrak & format:** semua panggilan memakai `apiLocal` (route `hsh` milik instance lokal),
  error diterjemahkan lewat `toApiFailure`/`errorMessage` (Indonesia), tanggal lewat
  `formatDate` (`timeZone: 'UTC'`), uang lewat `Intl` IDR tanpa desimal. Tidak ada URL backend
  absolut di bundle (axios `baseURL: ''` + proxy Vite).
- **Pembersihan:** data uji (2 periode, 6 entri, 4 item, 4 alias, 2 batch), admin uji, dan grup uji
  dihapus. DB akhir: `0 periode / 0 entri / 0 item / 0 alias / 0 batch / 7 kategori`, 20 admin,
  3 grup (baseline).

---
title: "WO-07: Dashboard"
description: "Dashboard ringkas: jumlah resident per status & cabang, total, daftar terbaru, dan tautan cepat — mengganti placeholder GET /api/ (GAP-3)."
status: progress
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - ui
  - dashboard
---

# WO-07: Dashboard

## Referensi
- SPEC: GAP-3, A-11, AC-6.
- Sumber data: **`GET /api/admin/residents/stats?branch_key=`** (B-3, sudah tersedia sejak WO-01) → `{total, by_status, by_branch}`. Fallback agregasi client-side tidak lagi diperlukan kecuali endpoint gagal.
- `src/pages/Dashboard.tsx` saat ini memanggil `GET /api/` → **selalu 404**.

## Langkah
1. Hapus panggilan `GET /api/`.
2. Kartu ringkasan: Total resident; per status (`DIRAWAT`, `DIKEMBALIKAN`,
   `TRANSFER`, `MENINGGAL`) dengan badge warna konsisten WO-04.
3. Distribusi per cabang (`branch.key`/`label` → jumlah) sebagai tabel kecil atau
   bar horizontal CSS (tanpa library chart).
4. Daftar 5 resident terbaru (`limit=5`, urut `created_at` dari API) dengan tautan
   ke detail.
5. Tautan cepat: "Tambah Resident" (`resident.write`), "Unduh Template XLSX"
   (`GET /api/admin/resident-template?format=xlsx`), "Kelola Admin" (`admin.read`).
6. Skeleton + empty state bila belum ada data.

## Kriteria Penerimaan
- [AC-1] Dashboard tidak lagi memanggil `/api/`; tidak ada 404 di console.
- [AC-2] Angka per status konsisten dengan hasil filter list WO-04 (diverifikasi
  pada data uji yang sama).
- [AC-3] Bila `stats` gagal (5xx), dashboard menampilkan pesan error ringkas + tombol coba lagi; **tidak** crash dan tidak menampilkan angka palsu.
- [AC-4] Tautan cepat hanya muncul bila permission terkait dimiliki; unduh
  template menghasilkan file XLSX valid.
- [AC-5] `npm run build` lulus.

## Bukti
Screenshot dashboard berisi data nyata + perbandingan angka dengan list API.

## Catatan
Tanpa library chart pada iterasi ini; visual sederhana (bar CSS) agar bundle kecil.

## Laporan Implementasi (2026-10-28)

### File
| Berkas | Isi |
|---|---|
| `src/pages/admin/DashboardPage.tsx` (ditulis ulang) | Ringkasan total/per status, sebaran cabang, 5 resident terbaru, tautan cepat |
| `src/api/residents.ts` (ubah) | `downloadResidentTemplate(format, noPii)` — unduh blob + object URL |

`internal/` & `plugins/` 0 baris.

### Catatan implementasi
1. **Angka dari `GET /api/admin/residents/stats`** — tidak ada agregasi
   client-side, jadi konsisten dengan filter list. Panggilan `GET /api/` (GAP-3)
   hilang sepenuhnya: `console` bersih tanpa 404 saat halaman dimuat.
2. **Semua status enum ditampilkan** walau nol (`Meninggal 0`) agar bentuk kartu
   stabil antar-periode; status di luar enum tetap ditampilkan jika ada datanya.
3. **`total - sum(by_status)`** ditampilkan sebagai “N resident belum punya status”
   (sesuai doc endpoint: resident tanpa status tidak masuk bucket mana pun).
4. **Sebaran cabang** = bar CSS dari `by_branch`, label diambil dari
   `/api/admin/endpoints` dengan fallback key.
5. **Unduh template** memakai `responseType: 'blob'` + object URL, sehingga token
   tidak pernah masuk URL dan bundle tetap tanpa URL absolut.
6. **Gagal stats = tidak ada angka palsu**: halaman menampilkan Alert +
   “tidak ada nilai perkiraan” + tombol Coba lagi. Kegagalan data pelengkap
   (5 terbaru / daftar cabang) tidak menjatuhkan ringkasan utama.
7. **Tautan cepat digating**: “Tambah resident” (`resident.write`), “Unduh template”
   (`resident.read`), “Kelola admin” (`admin.read`).

### Gate (browser, `http://100.125.175.34:5173/app/dashboard`)
Data uji: 3 resident (DIRAWAT/DIKEMBALIKAN/TRANSFER). Semua dihapus setelah uji;
`stats` kembali `{"total":0,"by_status":{},"by_branch":{}}`.

| # | Uji | Hasil |
|---|---|---|
| 1 | Muat halaman | Tanpa 404; `console` bersih (0 error) |
| 2 | Kartu | Total `3`; Dirawat `1`, Dikembalikan `1`, Transfer `1`, Meninggal `0` |
| 3 | Konsistensi dengan list | `?status=DIRAWAT` → total 1; `DIKEMBALIKAN` 1; `TRANSFER` 1; `MENINGGAL` 0 — **identik** |
| 4 | Sebaran cabang | `Instance ini` 3, bar penuh (1 cabang) |
| 5 | 5 resident terbaru | 3 baris (data kurang dari 5), link ke detail, `StatusBadge` sesuai `current_status` |
| 6 | Unduh template | Respons 200 `application/vnd.openxmlformats-…sheet`, toast “Template XLSX diunduh.”; file `/tmp/tpl.xlsx` terdeteksi `Microsoft Excel 2007+` (12943 byte) |
| 7 | Stats gagal (route di-abort) | Alert “Gagal memuat ringkasan” + “tidak ada nilai perkiraan”, tombol Coba lagi; **tidak ada kartu angka** |
| 8 | Pemulihan | Klik Coba lagi → Total `3` tampil kembali |
| 9 | Viewer tanpa `resident.write` | Tombol “+ Tambah resident” tidak ada; “Unduh template” & “Kelola admin” tetap ada |
| 10 | `npm run build` | hijau (133 modul) |
| 11 | Stateless | grep host:port di `dist` → kosong |

### Catatan jujur
- Playwright tidak memicu event `download` untuk unduhan blob+object URL; jadi
  `AC-4` “file XLSX valid” diverifikasi lewat **respons HTTP** (200, MIME benar,
  12943 byte, dikenali `file` sebagai Excel 2007+) dan toast sukses — bukan lewat
  berkas yang benar-benar tersimpan dari browser.
- `LATEST_COUNT` tetap 5 walau data uji hanya 3, jadi urutan terbaru tidak bisa
dibuktikan dengan paginasi; yang terverifikasi: jumlah baris = `min(5, total)` dan
tautan tiap baris mengarah ke detail yang benar.

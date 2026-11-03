---
title: "WO-02: UI kartu dashboard lintas modul"
description: "Tambah blok 'Kebutuhan Rumah Tangga' dan 'Kebutuhan Sekunder' (periode aktif) + tautan cepat modul kebutuhan di DashboardPage.tsx, konsumsi endpoint agregat WO-01."
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
tags:
  - work-order
  - dashboard
  - ui
---

# WO-02: UI kartu dashboard lintas modul

## Referensi
- SPEC D-7, D-8, D-9; O-1 (bar per kategori), O-2 (periode kosong = angka 0).
- `src/pages/admin/DashboardPage.tsx` (hasil WO-07 topic `1793174442984`) — kartu
  resident TIDAK berubah perilaku.
- Pola gating: `hasPermission` + `PermissionGate`.

## Langkah

1. API client: `src/api/dashboard.ts` — `getDashboardSummary()` memanggil
   `GET /api/admin/dashboard/summary`.
2. Blok hsh & sec (satu komponen bersama, param label/permission):
   - Judul + label periode aktif (`period_code`); bila `period: null` → tampil
     "Belum ada periode aktif" dengan counter 0.
   - Angka ditonjolkan: **Total** dan **Urgent** (Urgent memakai warna merah/pink
     konsisten badge URGENT existing).
   - Bar ringkas per kategori (D-7/O-1): tiap kategori 1 baris — nama + proporsi
     urgent/total (bar CSS, tanpa library chart).
   - Bila blok `null` (aktor tanpa `hsh.read`/`sec.read`) → blok TIDAK dirender
     (bukan kartu kosong).
3. Gagal memuat blok (endpoint 200 tapi data blok error-info, atau request gagal):
   alert ringkas per-blok + tombol coba lagi; **tidak ada angka palsu** (pola AC-3
   WO-07).
4. Tautan cepat baru (gated): "Kelola Kebutuhan" (`hsh.read`), "Kebutuhan Sekunder"
   (`sec.read`) — mengarah ke rute `/needs/*` & `/needs-secondary/*` yang ada.
5. Skeleton/empty state konsisten kartu resident.

## Kriteria Penerimaan

- [AC-1] `npm run build` lulus; grep host:port di `dist` tetap kosong (stateless).
- [AC-2] Dashboard menampilkan 3 kelompok: resident (lama, tidak berubah), hsh, sec
  — angka hsh/sec cocok dengan respons endpoint agregat.
- [AC-3] Admin tanpa `hsh.read`: blok hsh hilang dari layar; resident & sec tetap
  tampil (dan sebaliknya).
- [AC-4] Tanpa periode aktif: kartu tampil dengan "Belum ada periode aktif" + angka 0,
  tanpa error.
- [AC-5] Error state per-blok + coba lagi pulih; kartu resident tidak ikut rusak.
- [AC-6] Tautan cepat baru hanya muncul bila permission dimiliki.

## Hasil Implementasi (commit `66a6eb2`, 2026-11-03)

- 3 file: BARU `src/api/dashboard.ts` + `src/pages/admin/dashboard/NeedsSummaryCard.tsx`;
  UBAH `src/pages/admin/DashboardPage.tsx`.
- Satu sumber angka = endpoint agregat; kartu hsh/sec (Total+Urgent ditonjolkan,
  bar CSS per kategori, label periode, tautan "Kelola" gated); blok `null` tidak
  dirender; periode kosong → "Belum ada periode aktif" + 0; `block.error` → alert +
  Coba lagi; `residents.error` → angka disembunyikan (tanpa angka palsu).
- AC-1 build hijau (162 modul), grep host:port di `dist` = 0 baris.
- AC-2..6 diverifikasi dev; diverifikasi ulang Manager via browser dengan akun
  terbatas (`dashboard.read`+`sec.read`): kartu hsh hilang, kartu sec + resident
  tetap tampil, "Belum ada periode aktif" tampil tanpa error.
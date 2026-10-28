---
title: "WO-07: Dashboard"
description: "Dashboard ringkas: jumlah resident per status & cabang, total, daftar terbaru, dan tautan cepat — mengganti placeholder GET /api/ (GAP-3)."
status: draft
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
- Sumber data: `GET /api/admin/residents/stats` (WO-01 B-3) **atau** agregasi
  client-side dari `GET /api/admin/residents?limit=...&branch_key=...`.
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
- [AC-3] Bila `stats` tidak tersedia (B-3 tidak dikerjakan), dashboard tetap
  berfungsi memakai agregasi client-side dan mencatat batasannya di UI.
- [AC-4] Tautan cepat hanya muncul bila permission terkait dimiliki; unduh
  template menghasilkan file XLSX valid.
- [AC-5] `npm run build` lulus.

## Bukti
Screenshot dashboard berisi data nyata + perbandingan angka dengan list API.

## Catatan
Tanpa library chart pada iterasi ini; visual sederhana (bar CSS) agar bundle kecil.

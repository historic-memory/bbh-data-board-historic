---
title: "Work Order (Master Index): Dashboard Ringkasan Lintas Modul (resident + hsh + sec)"
description: "Memperluas dashboard admin dengan endpoint agregat baru dan kartu ringkasan Kebutuhan Rumah Tangga (hsh) & Sekunder (sec) di samping resident."
type: work-order
status: progress
created: "2026-11-03"
updated: "2026-11-03"
tags:
  - work-order
  - dashboard
  - admin-panel
---

# Work Order: Dashboard Ringkasan Lintas Modul

## Referensi
- SPEC: `.historic/1793687545424-.../spec.md` (D-1..D-9 terkunci; O-1..O-4 dijawab).

## Ringkasan
Endpoint agregat baru `GET /api/admin/dashboard/summary` di plugin baru
`plugins/dashboard` (tanpa tabel sendiri), lalu kartu hsh + sec di dashboard SPA.
Semua keputusan teknis lihat SPEC; WO di bawah hanya rincian langkah + AC.

| # | Berkas | Isi | Prasyarat | Status |
|---|---|---|---|---|
| 1 | `WO-01-endpoint-agregat.md` | Plugin `plugins/dashboard` + permission `dashboard.read` (migrasi plugin `auth`) + endpoint agregat 3 blok | — | pending |
| 2 | `WO-02-ui-kartu.md` | UI: blok "Kebutuhan Rumah Tangga" & "Kebutuhan Sekunder" + tautan cepat di `DashboardPage.tsx` | WO-01 | pending |
| 3 | `WO-03-verifikasi-dokumentasi.md` | Swagger, README, koleksi Bruno, uji RBAC end-to-end | WO-02 | pending |

## Aturan Umum
- Semua kode fitur di `plugins/dashboard/` — TIDAK menyentuh `internal/`.
- Permission baru di-seed lewat migrasi **plugin `auth`** (fail-closed).
- Tanpa library chart; bar CSS seperti WO-07.
- Commit kode per-WO; dokumen historic di-commit terpisah di repo `.historic/.database`.

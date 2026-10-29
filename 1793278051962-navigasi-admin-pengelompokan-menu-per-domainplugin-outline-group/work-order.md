---
title: "Work Order (Master Index): Navigasi Admin — outline group per domain/plugin"
description: "Rencana implementasi restrukturisasi sidebar admin: registry navigasi terpusat (data-driven), render outline group collapsible, filter izin per group/item, dan verifikasi."
status: draft
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - admin-panel
  - ui
  - navigation
  - frontend
---

# Work Order (Master Index): Navigasi Admin — Outline Group

## Referensi
- **SPEC topik ini:** `spec.md` (N-1..N-11, GAP-1..GAP-4, O-1..O-2).
- **Berkas utama (wajib):** `sub_app/webapp/src/layouts/AdminLayout.tsx`.
- **Berkas pendukung:** `sub_app/webapp/src/App.tsx` (rute — **tidak diubah**), `src/auth/AuthContext.tsx` (`hasPermission`).
- **Konteks owner (2026-10-29):** kelompokkan menu per **plugin core business**, buat **outline group** yang bisa di-expand.

## Prinsip Pelaksanaan
1. **Registry terpusat** = single source of truth menu + judul; layout hanya merender (N-2).
2. **Data-driven**: tambah domain/plugin = tambah entri registry, tanpa mengubah layout (N-11).
3. **URL/rute tidak diubah** (N-6); hanya struktur sidebar.
4. Filter izin **fail-safe**: group tampil bila ≥1 item lolos izin; item per izin; group kosong disembunyikan (N-5).
5. **Rahasia bukan di sini**: menyembunyikan menu bukan keamanan; otorisasi tetap di backend (N-10).
6. Tailwind v4 & komponen `src/components/ui/*` dipakai apa adanya; **tanpa dependensi baru**.
7. Setiap WO berakhir: `historic rebuild` → `historic save` → `historic git commit` (**tanpa push**).

## Daftar WO
| # | File | Scope | Prasyarat | Status |
|---|---|---|---|---|
| WO-01 | `WO-01-registry-navigasi-outline.md` | Registry navigasi + render outline group (collapsible) di `AdminLayout` | — | pending |
| WO-02 | `WO-02-verifikasi-dokumentasi.md` | Verifikasi build/perilaku (izin, expand, mobile) + dokumentasi | WO-01 | pending |

## Catatan Status
- Semua WO masih `pending`; belum ada implementasi.
- Topik ini **UI-only** (frontend `sub_app/webapp`); backend tidak disentuh.
- Perubahan ini **bukan** plugin baru; ia merapikan *shell* admin yang menampung semua plugin.


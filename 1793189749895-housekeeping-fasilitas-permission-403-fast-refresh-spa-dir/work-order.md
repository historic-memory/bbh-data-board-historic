---
title: "Work Order (Master Index): Housekeeping"
description: "Rencana implementasi H-9/H-10 (fasilitas DELETE + branch_key), H-2 (Fast Refresh useAuth), GAP-14 (SPA_DIR), dan H-8 (uji jalur 403)."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - housekeeping
---

# Work Order (Master Index): Housekeeping

## Referensi
- **SPEC topik ini:** `spec.md` (A-1..A-9, AC-1..AC-10, GAP-1..GAP-5).
- Sumber item: `.historic/1793174442984-setup-admin-panel-ui-kelola-resident-admin/work-order.md`
  Housekeeping **H-2, H-8, H-9, H-10**.
- `plugins/resident/plugin.go` (grup `facilities`), `plugins/resident/services/query.go:725`
  (`UpdateFacility`), `plugins/resident/services/facility.go`.
- `plugins/auth/middleware` (`RequirePermission`), `plugins/auth/services` (`EnsureCanGrant`).
- `sub_app/webapp/src/auth/AuthContext.tsx`, `src/pages/admin/facilities/*`, `src/api/facilities.ts`.
- `plugins/pages/plugin.go:20` (`spaDir`).

## Prinsip Pelaksanaan
1. Semua perubahan **aditif** — tidak memecah klien/konfigurasi existing.
2. Backend dulu (WO-01) sebelum UI fasilitas.
3. Verifikasi 403 memakai admin uji terpisah; **jangan** menurunkan hak root terakhir.
4. `internal/` tidak disentuh; `plugins/pages` adalah plugin sehingga aman diubah.
5. Swagger diregen di container `app`; Bruno memakai env `Local`.
6. Tidak ada push.

## Daftar WO
| # | File | Scope | Prasyarat | Status |
|---|---|---|---|---|
| 1 | `WO-01-fasilitas-delete-branch.md` | `DELETE /api/admin/facilities/:id` (guard `resident.delete`, hard delete) + `branch_key` pada `UpdateFacility`/`PUT` + test + UI (tombol hapus, field cabang aktif) | — | pending |
| 2 | `WO-02-fast-refresh-spa-dir.md` | Pisah `useAuth` dari `AuthContext.tsx` (H-2) + env `SPA_DIR` di `plugins/pages` dengan log error eksplisit (GAP-14) | — | pending |
| 3 | `WO-03-uji-403.md` | Bukti e2e tiga skenario 403 anti-escalation (H-8) dengan admin uji non-root; catat output | WO-01 | pending |

## Definisi Selesai
- Fasilitas bisa dihapus lewat API + UI; alias terhapus, `birth_records.facility_id` → `NULL` (dibuktikan).
- `PUT` fasilitas menerima `branch_key` (blank → NULL, tak dikenal → 400, absen → tak berubah).
- Tiga skenario 403 terbukti dengan output nyata.
- `AuthContext.tsx` hanya mengekspor komponen; `useAuth` di file sendiri; Fast Refresh bersih.
- `SPA_DIR` berfungsi; tanpa env perilaku identik; direktori hilang → log error jelas.
- Swagger + Bruno diperbarui; `go build`/`go test` lulus; `npm run build` lulus.
- `internal/` tidak berubah; tanpa push.
- Dokumen historic: `historic rebuild` → `historic save` → `historic git commit`.

## Catatan
- Topik ini **draft** — daftar WO dapat berubah setelah peninjauan.
- Urutan: **WO-01 → WO-02 → WO-03**.
- Konsekuensi hapus fasilitas (riwayat `facility_id` jadi `NULL`) adalah **keputusan A-1/A-2**, bukan bug.

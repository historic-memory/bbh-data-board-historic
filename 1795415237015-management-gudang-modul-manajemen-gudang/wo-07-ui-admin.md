---
title: WO-07 — UI admin /warehouse
description: "SPA: daftar item, batch, masuk/keluar, summary"
status: pending
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-07 — UI admin `/warehouse/*`

## Prasyarat

WO-03 lulus (bisa paralel dgn WO-04..06).

## Lingkup

- `src/api/wh.ts`, `src/lib/whFields.ts` (mirror fields.yaml + drift check), `src/pages/admin/warehouse/*`.
- Halaman: kategori & produk (CRUD + cari), gudang (CRUD), daftar item + stok terkini + badge near-expiry, detail item = daftar batch (umur hari, expiry, terlama dulu) + form masuk + modal keluar (pilih batch, reason/destination), summary/unduh template/impor.
- Rute `/warehouse/*` di `App.tsx`; grup nav "Gudang" di `src/navigation/menu.ts` filter `warehouse.read`.
- PermissionGate level halaman; tombol mutasi `hasPermission`; Pola komponen `src/components/ui/*`.

## Acceptance Criteria

- AC-1 `tsc -b && vite build` hijau; dist bebas URL backend (grep host:port kosong).
- AC-2 E2E Playwright via dev server :5173 (Tailscale): CRUD produk, batch masuk, keluar kurang dari sisa → pesan 409, near-expiry badge.
- AC-3 RBAC viewer: tombol mutasi tersembunyi; admin tanpa `warehouse.read` → guard 403.
- AC-4 Gotcha UI table: `TableSkeletonRows({colSpan,rows})`/`TableStateRow({colSpan,empty,...})` — tanpa prop `columns`/children.

## Catatan Implementasi (Dev)

(disi bertahap)


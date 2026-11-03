---
title: WO-07 — UI admin modul kunjungan
description: Halaman admin /visits (daftar booking, detail + riwayat status, master pengunjung) + api/visit.ts + nav grup "Kunjungan".
type: work-order
status: pending
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-07 — UI admin modul kunjungan

**Prasyarat:** WO-02, WO-03.

## Lingkup

1. `src/api/visit.ts` + `src/lib/visitFields.ts` (mirror enum/label — TERPISAH dari `lib/fields.ts` yang milik resident).
2. Halaman `src/pages/admin/visits/`:
   - `VisitListPage` — tabel booking (nama pengunjung, asal, jumlah, waktu (format UTC→WIB, pola `timeZone:'UTC'`), status badge), filter cabang/status/rentang tanggal, paginasi 25.
   - `VisitDetailPage` — detail booking + riwayat status + tombol ubah status (modal; note wajib utk CANCELLED) + edit catatan.
   - `VisitorListPage` — master pengunjung + jumlah kunjungan; detail pengunjung = daftar kunjungannya.
   - Form create manual visit (pilih/buat pengunjung).
3. Route `/visits/*` + `/visitors/*` di `App.tsx`; guard `hasPermission('visit.read')` level halaman.
4. Nav: grup baru **"Kunjungan"** di `src/navigation/menu.ts` (jangan ubah grup lain).
5. Uji Playwright :5173 — CRUD, ubah status, RBAC (viewer: tombol mutasi tersembunyi; admin tanpa `visit.read` → 403 guard).

## Acceptance Criteria

- AC-1: `npm run build` hijau; tidak ada URL absolut backend di `dist` (cek grep host:port).
- AC-2: Semua alur utama teruji di Playwright; data uji dibersihkan.

---
title: "WO-01: Registry navigasi & outline group di AdminLayout"
description: "Membuat registry navigasi terpusat (data-driven) dan merender sidebar admin sebagai outline group collapsible dengan filter izin per group/item serta judul dokumen dari sumber yang sama."
status: complete
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - admin-panel
  - ui
  - navigation
---

# WO-01: Registry Navigasi & Outline Group di `AdminLayout`

## Referensi
- SPEC `spec.md` N-2, N-3, N-5, N-7, N-8, N-9, N-11.
- `sub_app/webapp/src/layouts/AdminLayout.tsx` (berkas utama).
- `sub_app/webapp/src/auth/AuthContext.tsx` (`hasPermission`).

## Tujuan
Memindahkan definisi menu dari array datar di dalam `AdminLayout` ke **registry terpusat** yang
mengelompokkan item per **domain/plugin**, lalu merender sidebar sebagai **outline group**
collapsible yang siap di-expand.

## Deliverable
1. **Registry baru** `sub_app/webapp/src/navigation/menu.ts`:
   - Tipe `NavItem`/`NavGroup` (lihat SPEC §Rancangan Teknis).
   - `NAV_GROUPS` berisi 4 group: `Utama`, `Data Warga`, `Kebutuhan`, `Administrasi` (isi per N-4).
   - Helper `visibleGroups(hasPermission)` (atau sejenis) + `titleFor(pathname)`.
   - **Tanpa dependensi baru.**
2. **`AdminLayout.tsx`** direstrukturisasi:
   - Menghapus `NAV_ITEMS` dan `TITLES` (dipindah ke registry).
   - Merender group: heading label + daftar `<NavLink>` untuk setiap item.
   - **Collapsible** (tombol toggle per group) + persist state via `localStorage` (N-7).
   - Group yang memuat rute aktif **selalu terbuka**; menyorot group aktif.
   - Filter izin: group tampil bila ≥1 item lolos `hasPermission`; item per izin; item tanpa
     izin selalu tampil (N-5).
   - `document.title` memakai `titleFor(location.pathname)` dari registry (N-9).
   - Perilaku drawer mobile tetap berfungsi.

## Aturan Pelaksanaan
- Jangan mengubah `App.tsx` (rute tetap; N-6).
- Jangan mengubah backend/permission/RBAC.
- Jangan menambah dependensi; pakai Tailwind v4 & komponen `src/components/ui/*` yang ada.
- Tulis aksesibilitas dasar: tombol toggle memakai `aria-expanded`.

## Kriteria Penerimaan (AC)
- **AC-1** Sidebar terbagi dalam group bernama (Utama/Data Warga/Kebutuhan/Administrasi); "Kebutuhan RT" muncul **sekali** sebagai judul group, bukan diulang 5×.
- **AC-2** Menambah satu entri group baru di `menu.ts` **cukup** untuk menampilkan domain baru (tidak menyentuh `AdminLayout`/`App.tsx`).
- **AC-3** User tanpa izin suatu domain **tidak melihat** group/itu item; group yang semua itemnya tak berizin **tidak dirender**.
- **AC-4** Group bisa dibuka/ditutup; keadaan tersimpan setelah reload; group rute aktif terbuka otomatis.
- **AC-5** `document.title` mengikuti halaman (mis. "Resident · BBH Data Board — Admin"); tidak ada array judul duplikat tersisa.
- **AC-6** Drawer mobile tetap membuka/menutup dan tautan berfungsi.
- **AC-7** `npm run build` (di `sub_app/webapp`) sukses; tidak ada import mati/sisa `NAV_ITEMS`/`TITLES`.

## Bukti yang Diminta
- Diff `AdminLayout.tsx` + file registry baru.
- Output `npm run build` sukses.
- Catatan verifikasi manual ringkas (bisa dilanjutkan di WO-02).

## Bukti Pelaksanaan (2026-10-29)

**Kode commit:** `c458f82` (repo kode, tanpa push).

**Berkas:**
- BARU `sub_app/webapp/src/navigation/menu.ts` — tipe `NavItem`/`NavGroup`,
  `NAV_GROUPS` (4 group), `BASE_TITLE`, helper `isItemVisible`/`visibleGroups`/
  `groupContainsPath`/`isPathInItem`/`titleFor`/`shortTitleFor`. Tanpa dependensi baru.
- UBAH `sub_app/webapp/src/layouts/AdminLayout.tsx` — `NAV_ITEMS` & `TITLES`
  DIHAPUS; merender `visibleGroups(hasPermission)` sebagai `NavGroupSection`
  (heading tombol `aria-expanded` + daftar `NavLink`), `collapsed` dipersist ke
  `localStorage` (`admin.nav.collapsed`), group rute aktif selalu terbuka,
  `document.title` via `titleFor`. Handler baca aman terhadap JSON rusak.
- UBAH `sub_app/webapp/README.md` — bagian "Navigasi admin — registry terpusat".

**Build:** `cd sub_app/webapp && npm run build` → `tsc -b && vite build` SUKSES,
147 modul, `dist/assets/index-DuxNVFHM.js` 425.75 kB (gzip 126.66 kB).
`grep -rn "NAV_ITEMS\|TITLES" src` → kosong (AC-7).

**Verifikasi ringkas (lihat detail di WO-02):** AC-1..AC-6 terverifikasi di
browser dev (`http://localhost:5173/app/`) dengan akun root + akun terbatas.

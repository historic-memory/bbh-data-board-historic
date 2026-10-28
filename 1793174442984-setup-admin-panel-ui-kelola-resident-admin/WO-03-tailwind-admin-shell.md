---
title: "WO-03: Tailwind v4 + Shell Admin + Route Guard"
description: "Fondasi Tailwind CSS v4 (tanpa tailwind.config.js), layout shell admin bergaya TailAdmin (sidebar/header/konten), routing admin dengan guard, halaman 404, dan indikator user + logout."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - ui
  - tailwind
  - layout
---

# WO-03: Tailwind v4 + Shell Admin + Route Guard

## Referensi
- SPEC: A-2, A-7, A-9, A-13, GAP-2, GAP-8, AC-2, AC-9.
- `sub_app/webapp/vite.config.ts` (`base: '/app/'`), `src/main.tsx`
  (`BrowserRouter basename="/app"` sudah ada).
- `plugins/pages/plugin.go` — SPA disajikan dari `./sub_app/webapp/dist`.
- Pola visual acuan: TailAdmin (sidebar + header + card/table/form) —
  **implementasi sendiri**, cek lisensi sebelum menyalin kode.

## Langkah

### T-1 — Tailwind CSS v4
1. `npm i -D tailwindcss @tailwindcss/vite` (Tailwind v4, Node 24 OK).
2. `vite.config.ts`: tambah `tailwindcss()` ke `plugins`.
3. `src/styles/index.css`: `@import "tailwindcss";` + blok `@theme` (warna brand
   BBH, radius, font). **Tanpa `tailwind.config.js`.**
4. `src/main.tsx`: import `./styles/index.css`.
5. Hapus style inline bertahap pada halaman yang dikerjakan (tidak wajib di WO ini
   untuk `Portal.tsx`).

### T-2 — Komponen dasar (milik sendiri, `src/components/ui/`)
`Button`, `Input`, `Label`, `Select`, `Textarea`, `Card`, `Badge`, `Modal`,
`Table` (header/row/empty-state/skeleton), `Alert`, `Pagination`, `Spinner`,
`Toast`. Semua mendukung `disabled` dan `aria-*` dasar.

### T-3 — Shell admin (`src/layouts/AdminLayout.tsx`)
- Sidebar (collapsible di layar kecil): Dashboard, Resident, Fasilitas, Admin,
  Grup & Permission (item RBAC hanya bila punya izin terkait).
- Header: judul halaman, nama user + email, tombol logout.
- `<Outlet />` untuk konten.
- Route guard (`src/auth/RequireAuth.tsx`): status `loading` → spinner; `anonymous`
  → `<Navigate to="/login" state={{ from }} />`.
- Judul dokumen per halaman (`document.title`).

### T-4 — Routing (`src/App.tsx`)
Struktur baru: route publik (`/`, `/login`, `/portal`) + grup route ter-guard
dengan `AdminLayout`. Tambah rute placeholder `/residents`, `/facilities`,
`/admins`, `/groups` (diisi WO-04..WO-06) dan 404 di dalam shell.

## Kriteria Penerimaan
- [AC-1] `npm run build` lulus; aset ter-hash tersaji di `/app/` dan CSS Tailwind
  ikut ter-build (ukuran CSS > 0, utilitas terbukti aktif).
- [AC-2] Tidak ada `tailwind.config.js`; token warna didefinisikan di `@theme`.
- [AC-3] Akses `/app/dashboard` tanpa token → redirect `/app/login`, dan setelah
  login kembali ke halaman yang dituju.
- [AC-4] Shell responsif: sidebar jadi drawer < 1024px, konten tidak overflow.
- [AC-5] Logout dari header → token terhapus → kembali ke login.
- [AC-6] Halaman 404 tampil di dalam shell untuk path tak dikenal (bukan blank).
- [AC-7] Tidak ada `internal/`/`plugins/` yang berubah.

## Bukti
`npm run build` (log), screenshot shell desktop + mobile, bukti redirect guard.

## Catatan
`Portal.tsx` (186 baris) tetap dipertahankan; jangan dipindah ke shell admin
kecuali diputuskan terpisah.

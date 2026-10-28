---
title: "WO-03: Tailwind v4 + Shell Admin + Route Guard"
description: "Fondasi Tailwind CSS v4 (tanpa tailwind.config.js), layout shell admin bergaya TailAdmin (sidebar/header/konten), routing admin dengan guard, halaman 404, dan indikator user + logout."
status: complete
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

## Perbaikan pasca-complete (commit `f01f35d`)
Modal `<dialog>` tidak ter-center karena **Tailwind Preflight** mengeset `margin: 0` pada semua elemen,
termasuk `<dialog>` — penengahan bawaan browser (`inset: 0; margin: auto`) jadi hilang. Fix: kelas `m-auto`
pada elemen `dialog`. Wajib diingat untuk elemen lain yang mengandalkan user-agent margin.

## Bukti Pelaksanaan (2026-10-28)

### Tailwind v4
- `npm i -D tailwindcss @tailwindcss/vite` → **4.3.3**. Plugin ditambahkan ke
  `vite.config.ts` (`plugins: [react(), tailwindcss()]`).
- `src/styles/index.css` = `@import "tailwindcss";` + `@source "../"` + blok
  `@theme` (palet brand `brand-50..900`, warna status resident, `--radius-card`,
  `--font-sans`) + `@layer base` (`body`, `:focus-visible`).
- `main.tsx` mengimpor `./styles/index.css`. **Tanpa `tailwind.config.js`** (AC-2
  terbukti: berkas tidak ada).

### Komponen `src/components/ui/`
`Button` (+`Spinner` di dalamnya), `Field` (`Label`/`Input`/`Select`/`Textarea`),
`Card` (`CardHeader`/`CardTitle`/`CardBody`/`CardFooter`/`Section`), `Badge`
(+`StatusBadge` + `STATUS_TONES`), `Modal` (berbasis `<dialog>` native, ESC &
backdrop), `Table` (+`TableWrapper`/`THead`/`Th`/`TBody`/`Tr`/`Td`/
`TableStateRow`/`TableSkeletonRows`), `Alert`, `Pagination`, `Toast`
(`ToastProvider`/`useToast`, `aria-live`), `cn`. Semua punya `disabled` + `aria-*`
dasar; barrel `index.ts` untuk impor ringkas.

### Shell & guard
- `src/layouts/AdminLayout.tsx` — sidebar + header + `<Outlet/>`; drawer
  (`-translate-x-full` ⇄ `translate-x-0`) + backdrop pada < 1024px; menu difilter
  lewat `hasPermission`; `document.title` per rute; indikator user + tombol Keluar.
- `src/auth/RequireAuth.tsx` — `loading` → spinner; `anonymous` →
  `<Navigate to="/login" state={{from}}/>`.
- `src/App.tsx` — publik (`/`, `/login`, `/portal`) + grup ter-guard
  (`/dashboard` + placeholder `/residents`, `/facilities`, `/admins`, `/groups`)
  + 404 di dalam shell; dibungkus `ToastProvider`.
- Halaman baru `src/pages/admin/{DashboardPage,NotFoundPage,PlaceholderPage}.tsx`;
  `Home.tsx` & `Login.tsx` dirombak pakai Tailwind.
- **Dihapus**: `src/pages/Dashboard.tsx`, `src/pages/NotFound.tsx` (diganti halaman
  `admin/*`). **`Portal.tsx` tidak disentuh.**

### Gate (browser, dev `http://100.125.175.34:5173/app/`)
| # | Uji | Hasil |
|---|---|---|
| 1 | `/app/dashboard` tanpa token | redirect `/app/login`, `history.state.usr.from="/dashboard"` |
| 2 | deep-link `/app/residents` tanpa token → login | kembali ke **`/app/residents`** (bukan dashboard) |
| 3 | login root → shell | sidebar 256px, nav `Dashboard/Resident/Fasilitas/Admin/Grup & Permission`, header `Dashboard | root | root@admin.local | Keluar` |
| 4 | judul dokumen | `Dashboard · BBH Data Board — Admin` (berubah per halaman) |
| 5 | 404 `/app/tidak-ada` | **di dalam shell** (sidebar tetap ada), tombol kembali ke Dashboard |
| 6 | logout dari header | `localStorage` kosong → kembali `/app/login` |
| 7 | responsive 500px | drawer tertutup `--tw-translate-x: -100%`, hamburger tampil, `scrollWidth==clientWidth` (tidak overflow); dibuka → `--tw-translate-x: 0px` + backdrop |
| 8 | desktop 1280px | sidebar `translate-x-0`, hamburger `display:none`, konten `padding-left:256px` |
| 9 | permission-aware | admin tanpa grup (`--no-default-group`) → sidebar **hanya Dashboard** |
| 10 | `npm run build` | hijau; `dist/assets/*.css` **21.8 kB** (Tailwind terbukti ter-build) |
| 11 | stateless | `grep http://(tailscale|localhost):(8082\|5173) dist` → kosong |

### Catatan pengujian (jujur)
- Tab browser yang dipakai uji berada dalam keadaan **tidak terlihat**, sehingga
  transisi CSS dibekukan: `getBoundingClientRect()` bisa menunjukkan posisi
  lama walau class sudah berubah. Debugging di atas awalnya menyesatkan karena
  ini. Nilai **deklaratif** (`--tw-translate-x`, `aria-expanded`, keberadaan
  backdrop) yang dipakai sebagai bukti — semuanya benar. `transform` juga
  dilaporkan `none` karena Tailwind v4 memakai properti `translate`, bukan
  `transform`; ini bukan bug.
- `npm audit` melaporkan 10 advisory (2 moderate, 8 high) — **semuanya sudah ada
  sebelum WO ini** (axios, react-router(-dom), vite, nanoid, postcss,
  source-map-js, browserslist, follow-redirects, baseline-browser-mapping).
  Tailwind v4 tidak menambah advisory langsung. `npm audit fix --force` **tidak**
  dijalankan karena akan menurunkan versi paket inti (di luar lingkup WO).

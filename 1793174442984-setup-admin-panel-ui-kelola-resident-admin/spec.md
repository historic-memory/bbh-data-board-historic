---
title: "SPEC: Setup Admin Panel (UI) — kelola resident & admin"
description: "Spesifikasi panel admin SPA (React + Vite + Tailwind v4) di sub_app/webapp: auth flow (login/logout/sesi/guard), shell admin, CRUD data resident, manajemen admin/grup/RBAC, dan dashboard ringkas — memakai API yang sudah ada."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - admin-panel
  - ui
  - spa
  - react
  - resident
  - rbac
  - auth
---

# SPEC: Setup Admin Panel (UI)

## Ringkasan
Membangun **panel admin berbasis web (SPA)** di `sub_app/webapp` (React 19 +
Vite 6 + Tailwind CSS v4 + react-router-dom v7 + axios) untuk mengoperasikan data
BBH dari browser: kelola **resident** (CRUD + status history + dokumen),
**fasilitas**, serta **admin/grup/RBAC** — plus **auth flow** (login/logout/sesi/
guard route).

**Backend sudah siap** untuk sebagian besar kebutuhan; hanya **tiga perubahan
tipis** yang disetujui (B-1..B-3) untuk menutup gap kontrak UI:
- Plugin `auth` — admins/members/JWT/RBAC (topik `1792753542116`, complete).
- Plugin `resident` — 8 tabel canonical + admin API + validator + generator
  (topik `1793093835163`, complete).
- Plugin `endpoint`/`federation`/`portalsso` — multi-cabang & SSO.

Pekerjaan topik ini **hanya frontend** (`sub_app/webapp`) + **perubahan backend
tipis yang dibatasi** (lihat bagian "Perubahan Backend Tipis"). Tidak ada fitur
baru di `internal/` maupun `plugins/` di luar daftar itu.

## Kondisi Awal (baseline yang sudah ada)
- Halaman: `Home`, `Login`, `Dashboard`, `Portal`, `NotFound` (`src/pages/`).
- `src/api/client.ts`: dua jalur token (`main_token` via `Authorization`,
  `portal_token` via `X-Portal-Token` + `X-Branch-Endpoint`), tabel
  `ROUTE_OWNERS` meniru `plugins/federation/routes.go`, interceptor 401
  membersihkan sesi pusat.
- Login saat ini memanggil `POST /api/auth/login` — **SUDAH DIVERIFIKASI SALAH**:
  route nyata `POST /api/admin/login` (respon `{access_token, expires_in}`), dan
  token belum pernah disimpan (GAP-1).
- `Dashboard.tsx` memanggil `GET /api/` (pasti 404) — GAP-3.
- CORS: `internal/app/bootstrap.go` membaca `CORS_ALLOWED_ORIGINS`; `.env` sudah
  memuat `http://localhost:5173`, jadi dev server bisa memanggil API langsung (A-9).

## Keputusan Desain
| # | Keputusan | Nilai |
|---|---|---|
| A-1 | Lokasi kode | Hanya `sub_app/webapp` (SPA) + perubahan backend tipis B-1..B-3. Tidak menambah kode fitur di `internal/`. |
| A-2 | Stack | React 19 + Vite 6 + `react-router-dom` v7 + `axios` (existing). Tidak menambah state-manager berat; pakai Context + hook. |
| A-3 | Auth flow | Login → simpan `main_token` (localStorage via `setMainToken`). Guard route admin (redirect ke `/login` bila tanpa token). 401 → bersihkan sesi → redirect login. |
| A-4 | Sumber kebenaran API | Endpoint & field dari **swagger `docs/swagger.json`** (WO-07) — UI tidak mengarang kontrak. |
| A-5 | Permission-aware UI | Elemen mutasi (create/edit/delete) disembunyikan/disabled bila token tidak punya permission (`resident.write`/`resident.delete`) — fail-closed, server tetap sumber akhir. |
| A-6 | Cakupan cabang | Menu resident memakai `apiLocal` (route pusat `/api/admin/*`). Menu business/cabang memakai `apiPortal` (bila nanti ada). Tidak mencampur token. |
| A-7 | Styling | **Tailwind CSS v4** (`tailwindcss` + `@tailwindcss/vite`, tanpa `tailwind.config.js` — konfigurasi via `@theme` di CSS). Komponen mengacu pada pola **TailAdmin** (layout sidebar, tabel, form, badge, modal) tetapi **diimplementasikan sendiri** — bukan menyalin template. Ikonya inline SVG dulu. |
| A-8 | Tanpa mock | UI menembak API nyata (container `app`, port `8082`); tidak ada data palsu yang di-commit. |
| A-9 | Dev workflow | **Dev wajib jalan dengan `npm run dev` tanpa build.** `baseURL` axios saat ini `''` (relatif) sehingga request ke `/api/*` akan mendarat di dev server Vite, **bukan** backend — CORS saja tidak cukup. Solusi: **Vite `server.proxy`** meneruskan `/api` dan `/healthz` ke backend (`VITE_API_TARGET`, default `http://localhost:8082`) sehingga dev bersifat same-origin. `base: '/app/'` + `basename="/app"` (sudah ada) dipertahankan → dev diakses di `http://localhost:5173/app/`. `CORS_ALLOWED_ORIGINS` tetap dibiarkan memuat `5173` sebagai jalur cadangan (tanpa proxy). |
| A-10 | Sumber permission UI | Permission diambil dari `GET /api/admin/me` (field `permissions: string[]`, hasil perubahan backend tipis B-1). Tidak ada endpoint baru khusus UI. |
| A-11 | Paginasi | **Realisasi WO-01:** `GET /api/admin/residents` kini **selalu** mengembalikan `total` (additive, `items` tetap array) → UI memakai nomor halaman (page size 25) dihitung dari `total`. Filter `status` memfilter **status terakhir** saja dan nilai tak dikenal → **400** (bukan hasil kosong). |
| A-12 | Field form resident | Form di-generate dari **mirror statis** `plugins/resident/schema/fields.yaml` di TypeScript (`src/features/resident/fields.ts`), karena backend tidak mengekspos schema. Risiko drift dicatat: perubahan `fields.yaml` **wajib** disertai update file mirror (dicatat di README fitur). |
| A-13 | Bahasa UI | **Bahasa Indonesia** untuk seluruh label/teks; identifier kode tetap Inggris. Enum memakai label Indonesia (mengikuti `fields.yaml`), nilai API tetap istilah domain. |
| A-14 | Cakupan RBAC UI | **List + assign/unassign grup** untuk admin; list grup & permission **read-only**. Create/update/delete admin & grup **ditunda** (bukan bagian topik ini). |
| A-15 | Audit log | `/api/audit/endpoints` **tidak** masuk topik ini. |
| A-16 | Urutan kerja | **Dev workflow dulu (WO-00)** agar `npm run dev` settle sebelum fitur apa pun dikerjakan; panel UI didahulukan atas importer (`1793109556782`). |
| A-18 | **SPA stateless & base route sync** | Webapp adalah SPA murni tanpa SSR/state server: seluruh state di browser (React state + `localStorage`). `dist/` harus **self-contained** (semua aset di `/app/assets/`, nama ber-hash) dan dapat disajikan backend apa adanya. **Empat tempat base route WAJIB identik `/app`**: (1) `vite.config.ts` → `base: '/app/'`, (2) `main.tsx` → `BrowserRouter basename="/app"`, (3) `plugins/pages/plugin.go` → prefix `/app` + `spaDir="./sub_app/webapp/dist"`, (4) aset di `dist/index.html` → `/app/assets/*`. Bundle **tidak** memuat URL backend absolut dan **tidak** membaca env saat runtime. |
| A-19 | Deploy `dist` | Development: repo bind-mount → `dist` hasil build HOST langsung terlihat container. Produksi: `dist` **harus** ikut artefak (COPY ke image atau volume read-only) — belum ada image produksi, dicatat sebagai GAP-13 dan **di luar lingkup** topik ini (cukup didokumentasikan + diverifikasi manual). |
| A-20 | Akses dev via Tailscale (commit `9466e75`) | Dev server bind semua interface (`VITE_DEV_HOST`, default `true`) + `server.allowedHosts` (localhost/127.0.0.1 + `VITE_ALLOWED_HOSTS`). Panel dibuka di `http://<ip-tailscale>:5173/app/`. **Catatan:** `VITE_API_TARGET` default dev kini menunjuk IP Tailscale; nilai per-mesin **seharusnya** di `.env.development.local`, bukan di `.env.development` yang di-commit (dicatat sebagai GAP-16, tidak memblokir). |

## Kontrak SPA ↔ Backend (stateless, base route `/app`)

### Aturan stateless
1. Tidak ada SSR, server-side session, atau cookie sesi. Auth = Bearer token dari
   `localStorage`; logout murni client-side (endpoint logout stateless).
2. `dist/` tidak boleh bergantung pada env runtime: **tidak ada** panggilan
   `import.meta.env` untuk URL API absolut di bundle produksi. Semua request
   memakai path relatif (`/api/...`).
3. Aset di `dist/index.html` harus memakai prefix `/app/` (dijamin `base`),
   sehingga backend bisa menyajikan langsung dari `spaDir` tanpa rewrite.
4. Semua rute client-side dilayani `index.html` oleh `pages` (`NoRoute` →
   `serveSPA`); SPA tidak boleh mengandalkan server menyediakan rute per-halaman.

### Tabel sinkronisasi base route (harus selalu sama)
| Komponen | Nilai | Berkas |
|---|---|---|
| Vite `base` | `/app/` | `sub_app/webapp/vite.config.ts` |
| Router `basename` | `/app` | `sub_app/webapp/src/main.tsx` |
| Backend prefix | `/app` | `plugins/pages/plugin.go` (`wildcardHandler`) |
| Direktori statis | `./sub_app/webapp/dist` | `plugins/pages/plugin.go` (`spaDir`) |
| Aset hasil build | `/app/assets/*` | `dist/index.html` |

**Aturan perubahan:** mengubah salah satu dari empat nilai di atas = perubahan
kontrak; wajib memperbarui semuanya sekaligus + menjalankan verifikasi WO-00 T-6.

### Konsekuensi
- `spaDir` bersifat **relatif terhadap CWD** proses server. Server wajib
  dijalankan dari root repo (container: `/src`). Bila tidak, SPA 404 — dicatat di
  GAP-14.
- Karena `dist/` di-gitignore (`.gitignore:11`), build produksi wajib menjalankan
  `npm ci && npm run build` di HOST sebelum image dibuat.

## Perubahan Backend Tipis (disetujui, batas AC-8)
Hanya tiga perubahan berikut yang boleh menyentuh `plugins/`; **tidak ada
penambahan endpoint/fitur lain**. Semuanya additive & backward-compatible.

| # | Perubahan | Alasan | Berkas terdampak |
|---|---|---|---|
| B-1 | `GET /api/admin/me` mengembalikan `permissions: []string` dan `groups` yang benar-benar terisi | A-5 permission-aware UI; saat ini `AdminView.Groups` di-hard-code `[]` | `plugins/auth/handlers/{handler.go,auth.go}` |
| B-2 | `GET /api/admin/residents` menerima query `status` (kode enum) dan mengembalikan `total` di samping `items` | Filter status + paginasi di WO-04 | `plugins/resident/{handlers,services,model}` | ✅ **Selesai WO-01** (`eeb7b74`) |
| B-3 | `/api/admin/residents/stats?branch_key=` → `{total, by_status, by_branch}` (guard `resident.read`) | Agregasi status/cabang tanpa menarik seluruh data | `plugins/resident/handlers` | ✅ **Selesai WO-01** (dipilih dikerjakan) |

**Aturan B-1..B-3:** kontrak lama tidak berubah (field baru additive), migrasi DB
baru tidak diperlukan, dan scope RBAC tetap. Semua perubahan wajib disertai
update `docs/swagger` + entry Bruno.

Bila B-2 ditolak saat implementasi, UI jatuh ke A-11 (prev/next tanpa `total`)
dan filter status dilakukan client-side pada halaman berjalan (dicatat sebagai
keterbatasan).

## Cakupan Fungsional
1. **Auth**: halaman Login (form + error state), Logout, `me` (nama/email),
   proteksi route, penanganan 401 global.
2. **Shell Admin**: layout (sidebar + header + area konten), navigasi antar
   halaman, indikator user aktif + tombol logout, halaman 404.
3. **Resident**: tabel list (search/filter status & branch, paginasi), halaman
   detail, form create/edit, aksi ubah status (append `resident_status_history`),
   daftar & tautan dokumen (Google Drive).
4. **Fasilitas**: list `facilities` untuk dipakai form resident (endpoint alias belum tersedia di API).
5. **Admin & RBAC**: list admin, assign/unassign grup, list grup + permission
   (read-only, sesuai A-14).
6. **Dashboard**: ringkasan (jumlah resident per status/branch, tautan cepat).

## Non-Goals
- Tidak mengubah backend di luar **B-1..B-3**; `internal/` tidak disentuh sama sekali.
- Tidak membangun modul importer (itu topik `1793109556782`).
- Tidak membangun desain visual final/branding (fase polish terpisah); TailAdmin
  dipakai sebagai acuan pola, bukan tema yang harus ditiru 1:1.
- Tidak menyentuh portal publik/federasi kecuali perbaikan bug yang ditemukan.
- Tidak menambah halaman create/edit/delete admin & grup (A-14).
- Tidak menambah audit log viewer (A-15).
- Tidak membangun UI importer (topik `1793109556782`).

## Gap yang Diketahui
| # | Gap | Catatan |
|---|---|---|
| GAP-1 | `Login.tsx` memanggil `/api/auth/login`; route auth resmi `/api/admin/login` | Diperbaiki di WO-02 |
| GAP-2 | Belum ada guard route (semua route publik) | Ditangani WO-03 |
| GAP-3 | `Dashboard.tsx` memanggil `/api/` (placeholder) | Diganti dashboard nyata di WO-07 |
| GAP-4 | Belum ada halaman kelola resident/fasilitas/admin | WO-04/WO-05/WO-06 |
| GAP-5 | `AdminView.Groups` selalu `[]string{}` (hard-code) walau admin punya grup | Diselesaikan **B-1** |
| GAP-6 | `/api/admin/me` tidak memuat `permissions` → A-5 tidak mungkin | Diselesaikan **B-1** |
| GAP-7 | List resident tanpa filter `status` dan tanpa `total` | Diselesaikan **B-2**; fallback A-11 |
| GAP-8 | `vite.config.ts` tanpa `server.proxy`; `axios.baseURL = ''` sehingga dev tidak bisa memanggil API | Diselesaikan **WO-00** (A-9) |
| GAP-9 | Tidak ada export schema field (`fields.yaml` hanya dibaca backend) | Diterima sebagai risiko drift, dimitigasi A-12 |
| GAP-10 | `plugins/pages` hanya menyajikan `./sub_app/webapp/dist`; build harus dijalankan di HOST sebelum verifikasi UI | Prosedur verifikasi: `npm run build` lalu buka `http://localhost:8082/app/` |
| GAP-11 | Tidak ada `.env`/konfigurasi target API untuk mode dev maupun produksi | Diselesaikan **WO-00** (`VITE_API_TARGET`, proxy) |
| GAP-12 | Dev SPA memakai `base: '/app/'` sehingga `http://localhost:5173/` kosong (harus `/app/`) | Diterima; didokumentasikan di WO-00 (`npm run dev` → buka `/app/`) |
| GAP-13 | Tidak ada image/Dockerfile produksi; `dist/` di-gitignore & tidak di-COPY, padahal `pages` menyajikan `./sub_app/webapp/dist` | **Di luar lingkup** (A-19): didokumentasikan + verifikasi manual di WO-00 T-7 |
| GAP-14 | `spaDir` relatif terhadap CWD → SPA 404 bila server dijalankan dari direktori lain (gagal senyap, bukan error jelas) | Didokumentasikan (WO-00 T-6/T-7); perbaikan kode (env `SPA_DIR`) di luar lingkup |
| GAP-15 | `index.html` masih `lang="en"` + judul `App` (bukan bahasa/identitas BBH) | Diperbaiki di WO-00 T-8 (metadata statis, bukan runtime env) |
| GAP-16 | `vite.config.ts` memakai fallback IP absolut (`http://100.125.175.34:8082`) dan `.env.development` (di-commit) berisi IP mesin → tidak portabel untuk dev lain | Housekeeping: fallback ke `http://localhost:8082`, IP pindah ke `.env.development.local` |
| GAP-17 | `AuthContext.tsx` mengekspor provider + hook → peringatan Fast Refresh (full reload saat diedit) | Housekeeping: pisah `useAuth` ke file sendiri |
| GAP-18 | `npm audit` webapp: 10 advisory (2 moderate, 8 high) pada axios/react-router-dom/vite — **sudah ada sebelum topik ini** | Housekeeping terpisah; JANGAN `npm audit fix --force` (menurunkan versi paket inti) |

## Kriteria Penerimaan (SPEC-level)
- [AC-1] Login ke `POST /api/admin/login` berhasil → `main_token` tersimpan & user diarahkan ke `/app/dashboard`.
- [AC-2] Route admin tanpa token → redirect `/app/login`; token invalid/expired (401) → sesi dibersihkan + redirect.
- [AC-3] Halaman Resident: list (search `q`, filter `branch_key` & `status`, paginasi limit/offset) + detail (termasuk status history & dokumen) + create + edit + ubah status berfungsi terhadap API nyata.
- [AC-4] Mutasi mengikuti permission dari `/api/admin/me` (tombol tersembunyi/disabled bila tak berhak; 403 server ditangani rapi).
- [AC-5] Halaman Admin: daftar admin + assign/unassign grup berfungsi (bila berhak); daftar grup & permission tampil read-only.
- [AC-6] Error API (400/401/403/404/409/422/500) ditampilkan sebagai pesan ramah, bukan crash.
- [AC-7] `npm run build` (tsc + vite) lulus tanpa error TS; bundle tersaji oleh plugin `pages` di `http://localhost:<APP_PORT>/app/`.
- [AC-8] Kode di `internal/` tidak berubah; perubahan `plugins/` **hanya** B-1/B-2(/B-3) yang tercatat, masing-masing dengan bukti uji.
- [AC-9] Styling Tailwind v4 terpasang dan dipakai konsisten (token `@theme`), tanpa `tailwind.config.js`.
- [AC-11] **`npm run dev` berfungsi penuh**: login + list resident sukses melawan backend nyata tanpa `npm run build`, di `http://localhost:5173/app/` (proxy `/api` & `/healthz`), dengan HMR aktif.
- [AC-12] **Base route sinkron**: `dist/index.html` hanya mereferensikan `/app/*`; `GET http://localhost:8082/app/`, `/app/residents`, dan `/app/assets/<hash>.js` semua merespons 200 dengan konten yang benar (deep link tidak 404).
- [AC-13] **SPA stateless**: bundle produksi tidak memuat URL absolut backend (`grep` artefak: tidak ada `http://localhost:8082` / `http://localhost:5173`), tidak memakai cookie sesi, dan berfungsi setelah `dist/` disalin ke lokasi lain selama backend dijalankan dari root repo.
- [AC-14] Auth berjalan tanpa state server: token hanya di `localStorage`; menghapus `dist/` tidak memengaruhi data apa pun; membuka dua tab tidak merusak sesi.
- [AC-10] Tidak ada data mock, token, atau PII yang ter-commit; `package.json` hanya menambah dependensi Tailwind.


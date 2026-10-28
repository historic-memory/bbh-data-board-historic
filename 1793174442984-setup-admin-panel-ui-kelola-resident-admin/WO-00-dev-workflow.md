---
title: "WO-00: Dev Workflow Settle (npm run dev)"
description: "Memastikan pengembangan harian berjalan penuh dengan `npm run dev` tanpa build (proxy Vite ke /api dan /healthz), sekaligus mengunci kontrak base route /app agar dist stateless dapat disajikan backend tanpa rewrite."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - dev-experience
  - vite
  - tooling
---

# WO-00: Dev Workflow Settle

> **WO ini dikerjakan PERTAMA** (SPEC A-16). Semua WO fitur mengasumsikan
> `npm run dev` sudah berfungsi; tanpa ini setiap iterasi UI harus `npm run build`
> (mahal, 10-20 detik) dan tidak ada HMR.

## Referensi
- SPEC: A-9, A-17, A-18, A-19, GAP-8, GAP-11..GAP-15, AC-11..AC-14.
- `sub_app/webapp/vite.config.ts` — `base: '/app/'`, `build.outDir: 'dist'`, tanpa `server.proxy`.
- `sub_app/webapp/src/api/client.ts` — `axios.create({ baseURL: '' })` (relatif!) di
  tiga klien (`apiLocal`, `apiPortal`, `apiClient`).
- `sub_app/webapp/src/main.tsx` — `BrowserRouter basename="/app"` (sudah benar).
- `internal/app/bootstrap.go` — CORS dari `CORS_ALLOWED_ORIGINS` (cadangan).
- `.env` root — `APP_PORT=8082`, `CORS_ALLOWED_ORIGINS` memuat `http://localhost:5173`.

## Diagnosa (kenapa dev belum jalan)
1. `baseURL: ''` → request `/api/...` dikirim ke origin dev server
   (`http://localhost:5173`), yang **tidak punya** route API → 404 / HTML.
   `CORS_ALLOWED_ORIGINS` tidak menolong karena request tidak pernah keluar
   dari origin Vite (bukan cross-origin).
2. `base: '/app/'` → membuka `http://localhost:5173/` menghasilkan halaman kosong;
   dev harus dibuka di `/app/`.
3. Tidak ada konfigurasi target backend, sehingga port `8082` (bukan 8080)
   harus di-hard-code.

## Langkah

### T-1 — Proxy dev di `vite.config.ts`
```ts
const apiTarget = process.env.VITE_API_TARGET ?? 'http://localhost:8082'

export default defineConfig(({ mode }) => ({
  plugins: [react(), tailwindcss()],
  base: '/app/',
  server: {
    port: 5173,
    strictPort: true,
    proxy: {
      '/api':    { target: apiTarget, changeOrigin: true },
      '/healthz':{ target: apiTarget, changeOrigin: true },
    },
  },
  build: { outDir: 'dist', emptyOutDir: true },
}))
```
- Catatan: konfigurasi fungsi (`({ mode })`) dipakai hanya bila perlu; bentuk objek
  statis juga boleh — pilih yang paling sederhana dan tetap terbaca.
- Jangan tambahkan `rewrite`; path `/api/*` diteruskan apa adanya.

### T-2 — Konfigurasi env
**Status: env files sudah dibuat** (2026-10-28):

| File | Status | Isi |
|---|---|---|
| `sub_app/webapp/.env.example` | di-commit | template + komentar |
| `sub_app/webapp/.env.development` | di-commit | `VITE_API_TARGET=http://localhost:8082` |
| `sub_app/webapp/.env.local`, `.env.development.local` | gitignored | override lokal |
| `.gitignore` | diubah | tambah `sub_app/webapp/.env.local`, `.env.*.local` (pola `.env` lama **tidak** mencakup keduanya) |

Catatan prioritas Vite: `.env.[mode].local` > `.env.[mode]` > `.env.local` > `.env`.
Karena itu **jangan** menaruh `VITE_API_TARGET` di `.env` — nilai di
`.env.development` akan menimpanya dan membingungkan. Override selalu lewat
`.env.development.local`.

> **JEBAKAN PENTING (blocker WO-00):** `process.env.VITE_API_TARGET` **tidak**
> terisi otomatis di `vite.config.ts`, karena Vite memuat berkas `.env` setelah
> konfigurasi dievaluasi. Wajib pakai `loadEnv`:
> ```ts
> import { defineConfig, loadEnv } from 'vite'
>
> export default defineConfig(({ mode }) => {
>   const env = loadEnv(mode, process.cwd(), '')
>   const apiTarget = env.VITE_API_TARGET || 'http://localhost:8082'
>   ...
> })
> ```
> Tanpa `loadEnv`, proxy akan jatuh ke fallback dan `.env` seolah tidak
> berpengaruh.

4. **Tidak** menambah `VITE_API_BASE_URL` — klien tetap relatif (A-17). Bila
   suatu saat dipakai, nilainya harus dioper ke klien lewat `loadEnv` juga,
   bukan `process.env` langsung.

### T-3 — Script `package.json`
- `dev`: `vite` (default, port 5173).
- `dev:lan`: `vite --host` untuk uji dari perangkat lain (opsional, dokumentasikan).
- `build`: tetap `tsc -b && vite build`.
- `preview`: `vite preview` — **catat** bahwa preview memakai `base: /app/`
  dan tetap butuh proxy; untuk verifikasi akhir pakai backend (`/app/` di 8082).

### T-4 — Dokumentasi singkat
`sub_app/webapp/README.md` (baru, atau bagian di README root):
1. `docker compose up -d app` (backend di `http://localhost:8082`).
2. `cd sub_app/webapp && npm ci && npm run dev`.
3. Buka `http://localhost:5173/app/` — **bukan** `/`.
4. Jika backend di port lain → ubah `VITE_API_TARGET` di `.env.local`.
5. Untuk mode produksi: `npm run build` lalu buka `http://localhost:8082/app/`.
6. Catat: `node_modules`/`dist` tidak di-commit; `app` container Go-only (tanpa Node).

### T-5 — Verifikasi (wajib, sebelum WO lain)
1. `npm run dev` → buka `/app/` → Home tampil, tidak ada error console.
2. `curl` dari dev server: `curl -i http://localhost:5173/healthz` → 200 (dari backend).
3. Request API lewat SPA (mis. `fetch('/api/admin/residents')` di console) →
   respons JSON backend (401 tanpa token = **benar**, bukan 404 HTML).
4. HMR: ubah teks di sebuah komponen → browser ter-update tanpa reload penuh.
5. Regresi: `npm run build` tetap lulus dan `http://localhost:8082/app/` menyajikan
   bundle hasil build.

### T-6 — Verifikasi sinkronisasi base route (SPEC A-18)
Empat nilai berikut **wajib** `/app`; jalankan checklist ini setiap kali salah
satunya disentuh:

| # | Cek | Perintah / lokasi |
|---|---|---|
| 1 | Vite `base` | `grep "base:" sub_app/webapp/vite.config.ts` → `'/app/'` |
| 2 | Router `basename` | `grep basename sub_app/webapp/src/main.tsx` → `"/app"` |
| 3 | Prefix backend | `grep -n '"/app"' plugins/pages/plugin.go` |
| 4 | Aset hasil build | `grep -o '/app/[^"]*' sub_app/webapp/dist/index.html` → semua berawalan `/app/` |

Lalu uji end-to-end terhadap backend (server dijalankan dari **root repo**):
```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8082/app/            # 200
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8082/app/residents    # 200 (deep link → index.html)
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8082/app/assets/<hash>.js  # 200 (application/javascript)
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8082/api/tidak-ada     # 404 JSON
```

### T-7 — Dokumentasi deploy `dist` (stateless, A-19)
1. Dokumentasikan di README: urutan rilis = `npm ci && npm run build` di **HOST**
   → pastikan `sub_app/webapp/dist/` ada → jalankan server dari **root repo**
   (atau `/src` di container) karena `spaDir` relatif terhadap CWD.
2. Catat GAP-13 (belum ada image produksi; `dist` tidak di-COPY, `.gitignore:11`)
   sebagai risiko operasional, **tanpa** mengubah `plugins/pages`.
3. Catat GAP-14: bila SPA 404 padahal `dist/` ada, periksa CWD proses server.
4. Verifikasi stateless: `grep -rE "http://localhost:(8082|5173)" sub_app/webapp/dist`
   → **tidak ada hasil**. Bundle tidak boleh menyimpan host backend.

### T-8 — Metadata statis `index.html`
`lang="en"` → `lang="id"`, `<title>App</title>` → `<title>BBH Data Board — Admin</title>`.
Bukan runtime env; murni statis agar bundle tetap deterministik.

## Kriteria Penerimaan
- [AC-1] `npm run dev` lalu buka `http://localhost:5173/app/` → SPA termuat tanpa
  error console; HMR aktif.
- [AC-2] `GET http://localhost:5173/healthz` → 200 dari backend (proxy bekerja).
- [AC-3] `GET http://localhost:5173/api/admin/residents` tanpa token → **401 JSON**
  dari backend (bukan 404/HTML) → membuktikan proxy benar.
- [AC-4] Ganti `VITE_API_TARGET` (mis. lewat `.env.local`) mengubah target tanpa
  mengedit kode.
- [AC-5] `npm run build` tetap lulus; bundle tersaji di `http://localhost:8082/app/`.
- [AC-6] `.env.local` tidak ter-commit; `.env.development` tidak berisi secret.
- [AC-7] `internal/` dan `plugins/` tidak berubah (hanya `vite.config.ts`,
  `package.json`, `.env.development`, `.env.example`, README, `.gitignore`,
  `index.html`).
- [AC-8] **Base route sinkron**: keempat nilai `/app` (Vite base, router basename,
  prefix `pages`, aset `dist/index.html`) sesuai; `GET /app/`, `/app/residents`,
  `/app/assets/<hash>.js` → 200 levat backend; `/api/<unknown>` → 404 JSON.
- [AC-9] **Stateless**: bundle `dist` tidak memuat URL absolut backend
  (`grep -rE "http://localhost:(8082|5173)" dist` kosong), tidak memakai cookie
  sesi, dan tidak membaca env saat runtime.
- [AC-10] Prosedur rilis terdokumentasi: build di HOST → server dijalankan dari
  root repo (karena `spaDir` relatif CWD); SPA 404 dicatat sebagai gejala CWD salah.

## Bukti
Log `npm run dev`, output `curl` (healthz, /api 401 JSON, /app/ 200, deep link 200,
ases 200), hasil `grep` bundle (tanpa host absolut), dan log `npm run build`.

## Catatan
- CORS `CORS_ALLOWED_ORIGINS` tetap dibiarkan memuat `5173` sebagai cadangan bila
  seseorang memakai `VITE_API_BASE_URL` absolut; tetapi jalur resmi = proxy.
- Jangan mengubah `base: '/app/'` menjadi `/` — plugin `pages` menyajikan SPA di
  `/app` dan `basename` router sudah mengikuti.
- SPA ini **murni statis**: tidak ada SSR, tidak ada penyimpanan state di server.
  Bila di masa depan butuh SSR, itu topik terpisah (bukan penambahan diam-diam).

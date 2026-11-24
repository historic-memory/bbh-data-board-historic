---
title: konteks
description: ""
status: create
created: "2026-11-24"
updated: "2026-11-24"
---
# konteks

## Kebutuhan
- Saat akses `https://bbh.rolldev.my.id/` (path `/`) harus menampilkan
  homepage sederhana yang berisi link tombol menuju SPA `/app/`.
- Saat ini `/` jatuh ke fallback SPA index (plugin `pages`), namun index
  SPA dibangun dengan `base: /app/` sehingga tampil blank/tanpa asset.

## Keputusan user (24 Nov 2026)
- Cukup **static HTML biasa disajikan dari Go** (bukan nginx static,
  bukan proxy `HOMEPAGE_URL`).
- Pendekatan `deploy/nginx/` (html + conf) DIBATALKAN dan sudah dihapus.

## Fakta kode relevan
- Plugin `plugins/pages` memegang wildcard semua path tak dikenal via
  `router.NoRoute` di `plugins/pages/plugin.go`.
  - `case strings.HasPrefix(path, "/app")` → `serveSPA` (dist Vite).
  - fallback default: `c.File(p.spaIndex)` (2 lokasi: error handler proxy
    homepage dan default wildcard).
- `p.spaDir = "./sub_app/webapp/dist"`, `p.spaIndex = <dist>/index.html`.
- Register di `cmd/server/main.go`, plugin ini harus terdaftar PALING
  AKHIR (memiliki wildcard).
- `internal/app/bootstrap.go` `registerStoreRoutes()` sengaja kosong —
  semua tampilan dipegang plugin `pages`.
- Health `/healthz` didaftarkan sebelum plugin agar tidak ditelan
  wildcard.

## Konteks deploy (sudah jalan)
- Domain `https://bbh.rolldev.my.id` sudah di-DNS ke VPS `51.79.173.32`;
  nginx di VPS → `proxy_pass` ke `http://100.125.175.34:8082`
  (Tailscale mesin dev).
- `.env` sudah diubah: `APP_URL`/`ADMIN_URL` → domain publik, CORS +
  domain publik, `STORAGE_PUBLIC_URL` → `https://bbh.rolldev.my.id/assets`.
- Backend Go jalan di container `bbb-data-board-app-1` (host :8082),
  SPA dist ada di `sub_app/webapp/dist`.
- SSH laptop → VPS ditolak (Permission denied) — pasang nginx via akses
  user sendiri.

## Rancangan implementasi (draft)
- Tambah static homepage di plugin `pages`, mis. `plugins/pages/homepage.html`
  (di-embed via `go:embed` agar bekerja di container tanpa volume tambahan).
- Di `wildcardHandler`, path exact `/` → serve homepage
  (`text/html; charset=utf-8`); tidak lagi jatuh ke fallback SPA index.
- Homepage minimal: judul "BBH Data Board", tombol `href="/app/"`.
- Tidak ada perubahan kontrak API lain.

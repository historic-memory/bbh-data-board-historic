---
title: "WO-01: Image produksi (SPA dist + biner)"
description: "Dockerfile.prod multi-stage + .dockerignore + entrypoint yang memverifikasi dist/index.html + docker-compose.prod.yml. Menutup GAP-13/H-4."
status: pending
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - deploy
  - docker
---

# WO-01: Image produksi (SPA dist + biner)

## Referensi
- SPEC: A-1..A-11, A-13, AC-1..AC-7, AC-9..AC-10, GAP-1, GAP-2.
- `Dockerfile` (dev — **jangan ubah perilaku**), `docker-compose.yml` (dev).
- `plugins/pages/plugin.go:20` — `spaDir = "./sub_app/webapp/dist"` (relatif CWD).
- `sub_app/webapp/package.json` — `build: "tsc -b && vite build"`; `vite.config.ts` `base: '/app/'`.

## Tugas
1. **`.dockerignore`** (repot baru): buang `node_modules`, `dist`, `.git`, `bin`, `docs`,
   `.historic`, `.credentials`, `*.log` agar context build ramping & tidak bocor rahasia.
2. **`Dockerfile.prod`** multi-stage:
   - **Stage `spa`** (`node:22-alpine`): `npm ci` → `npm run build` di `sub_app/webapp`
     (hasil `dist`).
   - **Stage `build`** (`golang:1.27-alpine`): copy modul → `go build -o /out/server ./cmd/server`
     + `-o /out/console ./cmd/console`. (Swagger tidak diperlukan di runtime.)
   - **Stage runtime** (`alpine` atau `gcr.io/distroless` bila memungkinkan):
     - `WORKDIR /app` (penting: `spaDir` relatif valid → A-5).
     - copy `/out/server`, `/out/console`, dan `dist` → `/app/sub_app/webapp/dist`.
     - `ca-certificates`, `tzdata`.
     - user **non-root** (UID tetap).
     - `EXPOSE 8080`, `HEALTHCHECK` → `/healthz`.
     - Entrypoint: **cek** `/app/sub_app/webapp/dist/index.html` ada; jika tidak → cetak error +
       `exit 1` (A-6).
3. **`docker-compose.prod.yml`**: service `app` (build `Dockerfile.prod`), `postgres`, `keydb`;
   `env_file`, `CORS_ALLOWED_ORIGINS`, `INSTANCE_KEY`, `APP_ENV=production`; healthcheck; **tanpa**
   bind-mount source. Beri komentar bahwa migrasi dijalankan manual.
4. Pastikan runtime **tidak** memuat Node/npm/`node_modules`.

## Bukti yang harus dilampirkan
- `docker build -f Dockerfile.prod -t bbh-data-board:test .` output sukses.
- `docker image inspect` → ukuran (target < 80 MB).
- `docker run` + curl: `/app/` 200, `/app/residents` 200, `/app/assets/<file>` 200,
  `/api/admin/residents` 401 JSON, `/api/xxx` 404 JSON.
- `docker run ... sh -c 'ls /app; command -v node || echo NO_NODE'` → NO_NODE.
- `id` di dalam container → non-root.
- Uji negatif: hapus `dist/index.html` → entrypoint exit ≠ 0.

## Kriteria Selesai
- AC-1..AC-7, AC-9, AC-10.
- `Dockerfile` & `docker-compose.yml` dev **tidak berubah**.

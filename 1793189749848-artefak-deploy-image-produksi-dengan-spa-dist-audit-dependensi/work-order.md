---
title: "Work Order (Master Index): Artefak Deploy"
description: "Rencana implementasi image produksi berisi SPA dist (H-4/GAP-13), klasifikasi audit dependensi (H-3), dan verifikasi + dokumentasi deploy."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - deploy
  - docker
---

# Work Order (Master Index): Artefak Deploy

## Referensi
- **SPEC topik ini:** `spec.md` (A-1..A-14, AC-1..AC-12, GAP-1..GAP-5).
- Sumber gap: `.historic/1793174442984-setup-admin-panel-ui-kelola-resident-admin/work-order.md`
  Housekeeping **H-3** & **H-4**; SPEC A-19 (GAP-13).
- `Dockerfile`, `docker-compose.yml`, `build.sh`, `.gitignore` (root).
- `plugins/pages/plugin.go` — `const spaDir = "./sub_app/webapp/dist"`.
- `sub_app/webapp/package.json`, `vite.config.ts` (`base: '/app/'`).

## Prinsip Pelaksanaan
1. **Jangan merusak dev**: `Dockerfile` & `docker-compose.yml` existing **tidak diubah perilakunya**.
   Artefak baru = file terpisah (`Dockerfile.prod`, `docker-compose.prod.yml`).
2. Build SPA di stage Node; build biner di stage Go; runtime **tanpa Node**.
3. `WORKDIR` runtime harus membuat `spaDir` relatif (`./sub_app/webapp/dist`) tetap valid.
4. Base route `/app` **tidak berubah** (4 titik sinkron).
5. Gagal keras (bukan senyap) bila `dist/index.html` tidak ada.
6. **Dilarang** `npm audit fix --force`.
7. Tanpa push registry / CI.

## Daftar WO
| # | File | Scope | Prasyarat | Status |
|---|---|---|---|---|
| 1 | `WO-01-image-produksi.md` | `Dockerfile.prod` multi-stage (node build SPA → go build → runtime slim non-root), `.dockerignore`, entrypoint yang memverifikasi `dist/index.html`, `docker-compose.prod.yml` | — | pending |
| 2 | `WO-02-audit-dependensi.md` | Klasifikasi 10 advisory `npm audit`; perbaikan aman (minor/patch) tanpa `--force`; rekomendasi major bump sebagai keputusan terpisah | — | pending |
| 3 | `WO-03-verifikasi-dokumentasi.md` | Verifikasi AC-1..AC-7 (SPA + API + non-root + ukuran), dokumentasi deploy (env, migrasi, rollback, CWD) | WO-01, WO-02 | pending |

## Definisi Selesai
- `docker build -f Dockerfile.prod` sukses dari repo bersih (tanpa `dist/`, tanpa `node_modules/`).
- Container prod: `/app/` 200, `/app/residents` 200, `/app/assets/*` 200, `/api/admin/*` 401 JSON,
  `/api/<unknown>` 404 JSON.
- Runtime tanpa Node/npm/`node_modules`; non-root; ukuran tercatat.
- Entrypoint gagal keras bila `dist/index.html` hilang.
- Laporan audit dependensi lengkap + tindakan aman diterapkan (tanpa `--force`).
- Dokumentasi deploy tersedia.
- Dev workflow (`Dockerfile`, `docker-compose.yml`, `npm run dev`) tidak berubah.
- Dokumen historic: `historic rebuild` → `historic save` → `historic git commit`.

## Catatan
- Topik ini **draft** — daftar WO dapat berubah setelah peninjauan.
- Urutan: **WO-01 → WO-02 → WO-03**.
- Override `spaDir` via env (GAP-14) dikerjakan di topik housekeeping, **bukan** di sini;
  di sini cukup dimitigasi dengan `WORKDIR` + verifikasi entrypoint.

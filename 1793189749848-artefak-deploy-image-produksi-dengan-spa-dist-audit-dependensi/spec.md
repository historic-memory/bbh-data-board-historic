---
title: "SPEC: Artefak Deploy — image produksi dengan SPA dist & audit dependensi"
description: "Menutup GAP-13/H-4 (tidak ada image produksi yang menyertakan sub_app/webapp/dist) dan H-3 (10 advisory npm pada webapp) agar rilis dapat di-deploy sebagai satu artefak mandiri."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - deploy
  - docker
  - frontend
  - supply-chain
---

# SPEC: Artefak Deploy (image produksi + audit dependensi)

## Ringkasan
Sampai saat ini aplikasi **hanya bisa dijalankan untuk development**: image `app` adalah
container Go-only yang *idle*, repo di-bind-mount, dan `sub_app/webapp/dist` dibangun manual
di HOST. `plugins/pages` menyajikan `./sub_app/webapp/dist` dari CWD, sehingga **tanpa bind-mount
produksi tidak ada SPA**. Inilah GAP-13 / H-4.

Topik ini menghasilkan **artefak rilis tunggal**: satu image yang berisi `bin/server` **dan**
`dist` SPA hasil build, dengan base route `/app` tetap sinkron, tanpa Node.js di runtime.
Sekaligus mengevaluasi/menutup advisory `npm audit` yang sudah ada sejak lama (H-3).

## Kondisi Awal (terverifikasi)
- `Dockerfile` — header eksplisit: *"Development image ... Go toolchain only (+ swag CLI).
  NO Node.js runtime"*. `CMD ["sh","-c","umask 002 && exec sleep infinity"]` → **idle**.
- `docker-compose.yml` — `app` bind-mount repo, `APP_PORT` default 8080, komentar tegas
  *"This compose file is for LOCAL DEVELOPMENT ONLY. Do not use in production."*
- `.gitignore` memuat `sub_app/webapp/dist/`, `docs/`, `/bin/`, `.credentials/`.
- `plugins/pages/plugin.go` — `const spaDir = "./sub_app/webapp/dist"` **hard-code**, relatif
  terhadap CWD (`plugin.go:20`). Tidak ada env override → GAP-14.
- `build.sh` — lingkungan kanonik container `app`; memanggil `go build` + `swag init`;
  opsi `--verify-docs`.
- `sub_app/webapp/package.json` — script `build: "tsc -b && vite build"`; deps
  `axios ^1.7`, `react ^19`, `react-router-dom ^7`, `vite ^6`, `tailwindcss ^4.3.3`.
- `npm audit` webapp: **10 advisory (2 moderate, 8 high)** — semua **pre-existing**, bukan
  akibat perubahan kita. `npm audit fix --force` **dilarang** (menurunkan versi paket inti).

## Keputusan Desain
| # | Keputusan | Nilai |
|---|---|---|
| A-1 | Artefak | **Satu image produksi** berisi biner Go + `dist` SPA. Bukan dua container, bukan volume eksternal wajib. |
| A-2 | Multi-stage | Stage 1 `node:*` (build SPA) → stage 2 `golang:*` (build biner) → stage 3 runtime slim. Setiap stage punya alasan tertulis. |
| A-3 | Nama file | `Dockerfile` **tidak diubah** (dipakai dev). Tambah **`Dockerfile.prod`** + `docker-compose.prod.yml` agar dev workflow tidak rusak. |
| A-4 | Node hanya di stage build | Runtime image **tanpa** Node/npm. `node_modules` tidak ikut ke image akhir. |
| A-5 | Lokasi `dist` di runtime | `/app/sub_app/webapp/dist` dan `WORKDIR /app` → `spaDir` relatif tetap valid **tanpa ubah kode**. |
| A-6 | Sensitivitas CWD | Karena `spaDir` relatif (GAP-14), runtime **wajib** `WORKDIR` = root layout yang memuat `sub_app/webapp/dist`; didokumentasikan; entrypoint **memverifikasi** keberadaan `index.html` dan gagal keras (exit ≠ 0) bila hilang. |
| A-7 | Konfigurasi runtime | Tetap via env (`DB_*`, `KEYDB_*`, `PORT`, `INSTANCE_KEY`, `APP_KEY`, `CORS_ALLOWED_ORIGINS`, `HOMEPAGE_URL`). **Tanpa** env khusus frontend di runtime (SPA stateless, A-18 topik sebelumnya). |
| A-8 | Base route | `/app` **tidak berubah** (4 titik sinkron tetap). Image produksi tidak boleh memperkenalkan base baru. |
| A-9 | Non-root | Runtime jalan sebagai user non-root (UID tetap), tidak menulis ke filesystem aplikasi. |
| A-10 | Ukuran | Target wajar < 80 MB (biner Go ~ statis + aset SPA ~ kecil). Diukur & dicatat, bukan janji kosong. |
| A-11 | Migrasi DB | **Tidak** dijalankan otomatis dari entrypoint. Tetap perintah eksplisit (`console migrate`) agar deploy tidak surprio-mutasi skema. |
| A-12 | Audit dependensi (H-3) | Evaluasi per-advisory: klasifikasi **dapat diperbaiki tanpa breaking** vs **butuh major bump**. Major bump (vite/router) = keputusan terpisah, tidak dipaksa di topik ini. `--force` dilarang. |
| A-13 | Healthcheck | Image menyediakan `GET /healthz` yang sudah ada; compose prod memakai healthcheck itu. |
| A-14 | Tanpa push registry | Topik ini berhenti di **build image lokal + verifikasi jalan**. Publikasi registry di luar lingkup. |

## Cakupan Fungsional
- Build image produksi satu perintah; container hasil build menyajikan SPA di `/app/`.
- Deep link SPA (`/app/residents`) → 200 `index.html`; aset `/app/assets/*` → 200.
- `/api/<unknown>` → 404 JSON (bukan SPA).
- Entrypoint gagal jelas bila `dist/index.html` tidak ada.
- Laporan audit dependensi dengan klasifikasi tindakan.

## Non-Goals
- Tidak mendaftarkan image ke registry / CI pipeline (di luar lingkup).
- Tidak mengubah mekanisme migrasi atau menambah auto-migrate.
- Tidak melakukan upgrade major framework frontend (vite 7 / router 8) — hanya dievaluasi.
- Tidak mengubah `Dockerfile` dev atau alur `npm run dev`.
- Tidak menambah reverse proxy / TLS (di luar lingkup).

## Gap yang Diketahui
| GAP | Deskripsi | Rencana |
|---|---|---|
| GAP-1 | Tidak ada image produksi; `dist` gitignored & tidak di-COPY (H-4/GAP-13) | **WO-01** |
| GAP-2 | `spaDir` hard-code relatif CWD → gagal senyap bila CWD salah (GAP-14) | Dimitigasi A-6 di **WO-01**; override env kode di topik housekeeping |
| GAP-3 | 10 advisory npm belum diklasifikasi (H-3) | **WO-02** |
| GAP-4 | Belum ada verifikasi bahwa image prod benar-benar bisa melayani API + SPA tanpa bind-mount | **WO-03** |
| GAP-5 | Dokumen deploy (env wajib, urutan migrasi, rollback) belum ada | **WO-03** |

## Kriteria Penerimaan (SPEC-level)
- **AC-1** — `docker build -f Dockerfile.prod -t bbh-data-board:test .` sukses dari repo bersih
  (tanpa `dist/` di host, tanpa `node_modules/`).
- **AC-2** — Container hasil build menyajikan `GET /app/` → 200, `GET /app/residents` → 200
  (index.html), `GET /app/assets/<file>` → 200 dengan `Content-Type` benar.
- **AC-3** — `GET /api/admin/residents` tanpa token → **401 JSON** (bukan HTML/404) dari container
  produksi; `GET /api/tidak-ada` → **404 JSON**.
- **AC-4** — Runtime image **tidak** memuat Node/npm dan **tidak** memuat `node_modules`.
- **AC-5** — Container jalan sebagai **non-root**; `WORKDIR` membuat `spaDir` relatif valid.
- **AC-6** — Entrypoint **exit ≠ 0** dengan pesan jelas bila `sub_app/webapp/dist/index.html` hilang.
- **AC-7** — `docker image inspect` ukuran tercatat; target < 80 MB (atau justifikasi bila lebih).
- **AC-8** — Laporan `npm audit`: setiap advisory punya klasifikasi (fix tersedia / butuh major /
  tidak relevan runtime) + rekomendasi; **tidak ada** `npm audit fix --force`.
- **AC-9** — `docker-compose.prod.yml` dapat menyalakan stack (app + postgres + keydb) dengan
  healthcheck; dev `docker-compose.yml` **tidak berubah perilaku**.
- **AC-10** — `Dockerfile` dev tetap identik (dev workflow tidak terdampak).
- **AC-11** — Dokumentasi deploy: env wajib, urutan `console migrate`, cara rollback, catatan CWD.
- **AC-12** — Tanpa push; tanpa menyentuh `internal/` kecuali benar-benar perlu (dan dicatat).

## Risiko
- **Drift base route**: bila stage SPA build lupa `base: '/app/'` → aset 404. Dimitigasi AC-2.
- **Ukuran image membengkak**: modul Go & cache ikut. Dimingasi multi-stage + `.dockerignore`.
- **`spaDir` relatif**: salah `WORKDIR` → SPA 404 senyap. Dimitigasi A-6 (gagal keras).
- **Upgrade audit**: memperbaiki advisory bisa memicu breaking change. Dimitigasi A-12 (tanpa `--force`).

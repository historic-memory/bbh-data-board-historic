---
title: "Work Order (Master Index): Setup Admin Panel (UI)"
description: "Rencana implementasi panel admin SPA di sub_app/webapp: auth flow, shell admin, CRUD resident, fasilitas, admin/grup/RBAC, dan dashboard — memakai API yang sudah ada (tanpa ubah backend)."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - admin-panel
  - ui
  - spa
---

# Work Order (Master Index): Setup Admin Panel (UI)

## Referensi
- **SPEC topik ini:** `spec.md` (A-1..A-8, AC-1..AC-8, GAP-1..GAP-4).
- Backend auth/RBAC: `.historic/1792753542116-bbh-data-board-auth-plugin-admin-schema/`
- Backend resident (API + validator + generator): `.historic/1793093835163-analisa-data-mentah-business/`
- Kontrak API: `docs/swagger/swagger.json` (topik `1793093835163` WO-07) + koleksi `api-tests/` (Bruno).
- Kode SPA: `sub_app/webapp/src/` (client.ts, App.tsx, pages/).

## Prinsip Pelaksanaan
1. **Frontend-first**: kode di `sub_app/webapp`. `internal/` **tidak disentuh**; `plugins/` hanya untuk B-1..B-3 (WO-01).
2. Sumber kebenaran endpoint/field = **swagger** + koleksi Bruno, bukan asumsi.
3. Token & pemilik route mengikuti `client.ts` (`ROUTE_OWNERS` ↔ `federation/routes.go`); jangan campur `main_token`/`portal_token`.
4. Build frontend dijalankan di **HOST** (`npm ci`/`npm run build`); `app` container Go-only. Build di container **tidak mungkin** (tidak ada Node).
5. Verifikasi UI dengan menembak API nyata (server container `app`, `http://localhost:8082`).
6. **Bahasa UI Indonesia**, identifier kode Inggris (SPEC A-13).
7. **SPA stateless** (SPEC A-18): tanpa SSR/state server; semua state di browser. Bundle tidak memuat URL backend absolut.
8. **Base route `/app` sinkron di 4 tempat** (A-18): Vite `base`, router `basename`, prefix `plugins/pages`, prefix aset `dist/index.html`. Mengubah satu = ubah semua.
9. Tidak ada push ke remote.

## Daftar WO
| # | File | Scope | Prasyarat | Status |
|---|---|---|---|---|
| 0 | `WO-00-dev-workflow.md` | **Dev workflow settle**: `npm run dev` jalan penuh — proxy Vite `/api`+`/healthz` ke `VITE_API_TARGET`, `.env.development`, script `dev`/`dev:lan`, verifikasi login tanpa build | — | complete |
| 1 | `WO-01-backend-kontrak-tipis.md` | B-1 `permissions[]` + `groups` terisi di `/api/admin/me`; B-2 filter `status` + `total` di list resident; (B-3 opsional) endpoint stats. Sertakan swagger + Bruno | — | complete |
| 2 | `WO-02-auth-flow.md` | Auth flow SPA: perbaiki endpoint login (GAP-1), simpan `main_token`, `me` + permissions, logout, penanganan 401 global, halaman Login proper | WO-00, WO-01 | complete |
| 3 | `WO-03-tailwind-admin-shell.md` | Fondasi Tailwind v4 + shell admin (sidebar/header/konten, pola TailAdmin), routing admin, **route guard**, 404, indikator user + logout | WO-00, WO-02 | complete |
| 4 | `WO-04-resident-crud-ui.md` | Resident: list (`q`, `branch_key`, `status`, limit/offset) + detail (status history & dokumen) + form create/edit + ubah status + tambah dokumen | WO-03, WO-01 | **complete** (Lapis A `d05ceb7`, B `00a918b`, C `a6406ca`) |
| 5 | `WO-05-facilities-ui.md` | Fasilitas: list `facilities` + alias; integrasi dropdown di form resident | WO-03 | draft |
| 6 | `WO-06-admin-rbac-ui.md` | Admin & RBAC: list admin, assign/unassign grup, list grup + permission (read-only), permission-aware | WO-03, WO-01 | draft |
| 7 | `WO-07-dashboard.md` | Dashboard: ringkasan resident per status/cabang + tautan cepat (ganti placeholder `/api/`, GAP-3) | WO-04 | draft |

## Definisi Selesai
- **`npm run dev` berfungsi penuh** (login + data nyata) tanpa `npm run build`, HMR aktif.
- **Base route `/app` sinkron**: `/app/`, deep link (`/app/residents`), dan aset `/app/assets/*` → 200 lewat backend; `/api/<unknown>` → 404 JSON.
- **Bundle stateless**: `dist` tidak memuat `http://localhost:*`; tidak ada cookie sesi; tidak membaca env runtime.
- `npm run build` (`tsc -b && vite build`) lulus tanpa error TS; bundle tersaji oleh plugin `pages` di `http://localhost:8082/app/`.
- Login → token tersimpan → redirect `/app/dashboard`; route admin ter-guard; 401 bersihkan sesi.
- Alur CRUD resident berfungsi end-to-end terhadap API nyata (dibuktikan manual/curl).
- Error API ditampilkan ramah (tidak crash) dengan pesan Indonesia.
- `internal/` tidak berubah; `plugins/` hanya B-1..B-3 dan tercatat.
- Tailwind v4 tanpa `tailwind.config.js`; tidak ada data mock / token / PII ter-commit.
- Dokumen historic: `historic rebuild` → `historic save` → `historic git commit`.

## Housekeeping (bukan WO, dikerjakan kapan saja)
| # | Item | Sumber |
|---|---|---|
| H-1 | `vite.config.ts` fallback `VITE_API_TARGET` → `http://localhost:8082`; IP Tailscale pindah ke `.env.development.local` (GAP-16) | WO-03 (commit `9466e75`) |
| H-2 | Pisahkan `useAuth` dari `AuthContext.tsx` agar Fast Refresh bersih (GAP-17) | WO-03 |
| H-3 | Evaluasi `npm audit` (10 advisory) — tanpa `--force` (GAP-18) | WO-03 |
| H-4 | Image produksi yang menyertakan `sub_app/webapp/dist` (GAP-13) | SPEC A-19 |
| H-5 | `bin/` **tidak** di-gitignore → risiko terulang insiden commit binari ~80 MB (WO-04 Lapis B) — **selesai**: `/bin/` ditambahkan ke `.gitignore` (`904d99c`) | WO-04 |
| H-6 | `UpdateResidentInput` belum mencakup `mother`/`birth`/`admission` → data tersebut tidak bisa diubah setelah create; perlu WO backend terpisah | WO-04 Lapis C |
| H-7 | Dokumen resident belum bisa dihapus/diedit (API hanya `GET`/`POST /:id/documents`) | WO-04 Lapis C |

## Catatan
- Progress: WO-00, WO-01, WO-02, WO-03, **WO-04 complete**. Berikutnya **WO-05** (fasilitas) atau **WO-06** (RBAC).
- **Cara cek stateless yang benar:** cari **host:port** (`http://localhost:8082`, `:5173`, IP tailscale), bukan string `http://localhost` polos. Bundle library (react-router) memang memuat `"http://localhost"` sebagai fallback internal — bukan URL backend.
- Topik ini **draft** — daftar WO dapat berubah setelah peninjauan SPEC.
- Importer data = topik `1793109556782` (di luar lingkup). Urutan: **WO-00 (dev workflow) → WO-01 → WO-02 → WO-03 → ...** (A-16).
- TailAdmin dipakai sebagai **acuan pola visual**; implementasi komponen milik sendiri.
  Periksa lisensi sebelum menyalin kode apa pun dari template.


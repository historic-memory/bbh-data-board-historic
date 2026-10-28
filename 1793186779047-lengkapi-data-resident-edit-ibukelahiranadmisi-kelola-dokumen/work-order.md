---
title: "Work Order (Master Index): Lengkapi Data Resident"
description: "Rencana implementasi H-6 (update mother/birth/admission pada PATCH resident) dan H-7 (DELETE/PATCH dokumen resident), beserta penyesuaian UI dan kontrak (swagger + Bruno)."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - resident
  - backend
  - documents
  - ui
---

# Work Order (Master Index): Lengkapi Data Resident

## Referensi
- **SPEC topik ini:** `spec.md` (A-1..A-14, AC-1..AC-11, GAP-1..GAP-6).
- Topik sebelumnya (frontend-only, sumber GAP): `.historic/1793174442984-setup-admin-panel-ui-kelola-resident-admin/`
  — lihat Housekeeping **H-6** & **H-7** di `work-order.md` topik itu.
- Backend resident: `plugins/resident/` (`services/service.go:159`, `services/query.go:188`,
  `handlers/handlers.go`, `plugin.go`).
- Skema DB: `plugins/resident/migrations/postgres/2026_10_27_120100_create_resident_core_tables.up.sql`.
- Kontrak: `docs/swagger/swagger.json` + koleksi `api-tests/`.
- UI: `sub_app/webapp/src/api/residents.ts`, `src/pages/admin/residents/*`.

## Prinsip Pelaksanaan
1. **Backend dulu, UI kemudian**: WO-01 & WO-02 harus `complete` (swagger ter-regen) sebelum WO-03.
2. Semua perubahan parsial mengikuti konvensi existing: **`nil` = tidak diubah**, **blank teks = NULL**.
3. **Satu transaksi** untuk update agregat (A-6) — jangan ada partial-write.
4. `internal/` **tidak disentuh**; semua perubahan fitur di `plugins/resident/`.
5. Build/perintah Go dijalankan di container `app`; build frontend di **HOST**
   (`npm run build` — container tidak punya Node).
6. Regen swagger dengan `swag init` **di container** `app`; jangan menulis ulang JSON manual.
7. Bruno request baru wajib memakai environment `api-tests/environments/Local.bru`.
8. Tidak ada push ke remote.

## Daftar WO
| # | File | Scope | Prasyarat | Status |
|---|---|---|---|---|
| 1 | `WO-01-update-sub-entitas.md` | H-6: `UpdateResidentInput` + `mother`/`birth`/`admission`; upsert dalam transaksi; validasi enum & `admitted_at`; unit/DB test | — | pending |
| 2 | `WO-02-dokumen-delete-patch.md` | H-7: service `DeleteDocument`/`UpdateDocument`, handler + route `DELETE`/`PATCH /:id/documents/:docId` (guard `resident.write`), test | — | pending |
| 3 | `WO-03-ui-edit-lengkap.md` | UI: perluas `PATCHResidentPayload`, form edit menampilkan Ibu/Kelahiran/Admisi (prefill + delta), aksi hapus/edit dokumen di detail, permission-aware, toast | WO-01, WO-02 | pending |
| 4 | `WO-04-kontrak-verifikasi.md` | Regen swagger, Bruno baru (patch sub-entitas, delete/patch dokumen), dokumentasi README, verifikasi end-to-end | WO-01, WO-02, WO-03 | pending |

## Definisi Selesai
- `PATCH /api/admin/residents/:id` mengubah `mother`/`birth`/`admission` sesuai A-2..A-5, atomik.
- `DELETE`/`PATCH /api/admin/residents/:id/documents/:docId` berfungsi, ter-guard `resident.write`.
- `go build ./...` + `go test ./plugins/resident/...` lulus di container `app`.
- `npm run build` (HOST) lulus tanpa error TS; `tsc -b` bersih.
- Swagger memuat field & endpoint baru; koleksi Bruno memuat request baru dan lolos manual.
- UI: form edit menampilkan seksi Ibu/Kelahiran/Admisi terisi; dokumen bisa dihapus/diedit.
- `internal/` tidak berubah; tidak ada PII/token ter-commit.
- Dokumen historic: `historic rebuild` → `historic save` → `historic git commit`.

## Catatan
- Topik ini **draft** — daftar WO dapat berubah setelah peninjauan.
- Urutan eksekusi: **WO-01 → WO-02 → WO-03 → WO-04**.
- `status` & daftar `documents` **tidak** disatukan ke `PATCH /:id` (A-14).
- Konvensi "admission terbaru" (A-4) didokumentasikan di kode & swagger description.

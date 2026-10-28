---
title: 'WO-04: Kontrak & verifikasi'
description: Regen swagger, tambah koleksi Bruno (patch sub-entitas + delete/patch dokumen), dokumentasi, dan verifikasi end-to-end.
status: planned
created: "2026-10-28"
updated: "2026-10-28"
tags:
    - work-order
    - contract
    - swagger
    - bruno
    - verification
---
# WO-04: Kontrak & verifikasi

## Referensi
- SPEC: A-10, A-13, AC-9..AC-11, GAP-4.
- `docs/swagger/` (dihasilkan `swag init`), `api-tests/residents/` + `api-tests/environments/Local.bru`.

## Tugas
1. **Regen swagger** di container `app` (`swag init`) → perbarui
   `docs/swagger/{docs.go,swagger.json,swagger.yaml}`. Pastikan field baru
   `mother`/`birth`/`admission` pada `UpdateResidentInput` & endpoint `DELETE`/`PATCH`
   `documents/{docId}` muncul.
2. **Bruno** request baru di `api-tests/residents/`:
   - `10-Patch-mother.bru`, `11-Patch-birth.bru`, `12-Patch-admission.bru`.
   - `13-Patch-document.bru`, `14-Delete-document.bru`.
   - Gunakan env `Local`; simpan `residentId`/`docId` dari request sebelumnya (chaining) bila ada.
3. **Dokumentasi**: perbarui `api-tests/README.md` (daftar request) & catatan kontrak bila perlu.
4. **Verifikasi end-to-end**:
   - Jalankan koleksi Bruno berturut-turut terhadap server `app` (port `8082`).
   - `go build ./...` + `go test ./plugins/resident/...`.
   - `npm run build` di HOST → buka `http://localhost:8082/app/` dan uji form edit + aksi dokumen.
   - Cek stateless: `grep -rn "http://localhost:(8082|5173)" sub_app/webapp/dist` → kosong.

## Kriteria Selesai
- AC-9, AC-10, AC-11 terpenuhi.
- Swagger & Bruno sinkron dengan implementasi.
- Verifikasi manual tercatat di laporan WO.

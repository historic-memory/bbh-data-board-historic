---
title: 'WO-04: Kontrak & verifikasi'
description: Regen swagger, tambah koleksi Bruno (patch sub-entitas + delete/patch dokumen), dokumentasi, dan verifikasi end-to-end.
status: complete
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

## Laporan Verifikasi (2026-10-28)

### Swagger (regen di container `app`)
`swag init -g cmd/server/main.go -o docs/swagger --parseDependency --parseInternal` → sukses.
- `paths_total` = **38** (sebelumnya 36), `ops_total` = **60**.
- Path baru: `/api/admin/residents/{id}/documents/{docId}` dengan operasi **delete** + **patch**.
- `UpdateResidentInput` kini memuat `mother`, `birth`, `admission` (aditif; tidak ada field lama hilang).
- Definisi baru: `UpdateMotherInput` (full_name/nik/phone/address), `UpdateBirthInput` (7 field),
  `UpdateAdmissionInput` (admitted_at/exited_at/note), `UpdateDocumentInput` (doc_type/url/label).
- `docs/` tetap **gitignored** (`.gitignore:25`) → artefak tidak di-commit, hanya dihasilkan lokal.

### Bruno (`api-tests/residents/`)
Request baru: `09-Patch-document`, `10-Delete-document`, `11-Patch-mother`, `12-Patch-birth`,
`13-Patch-admission`. `09-Delete-resident` → **`14-Delete-resident`** (seq 14) agar alur folder
berjalan dari create sampai delete; `07-Add-document` kini menyimpan `docId` untuk chaining.
`api-tests/README.md` diperbarui (tabel urutan request + cara menjalankan satu folder).

### Hasil eksekusi
- `bru run --env Local residents` → **14/14 request PASS, 28/28 test** (termasuk patch dokumen,
  delete dokumen 204, patch mother/birth/admission).
- `bru run --env Local --tags=smoke` → 4/4 request, 9/9 test PASS (tanpa regresi).
- `go build ./...` + `go test ./plugins/resident/...` (container `app`) lulus.
- `npm run build` (HOST) lulus; grep `http://<host>:(8082|5173)` pada `dist` **kosong** (stateless).
- `internal/` tidak diubah; tidak ada push.

### Catatan
- Token di `api-tests/.env` (gitignored) harus di-refresh bila server di-restart; nilai lama
  menghasilkan 401 massal.
- Jalur **403** (`resident.write` tanpa wildcard) tetap belum teruji end-to-end (H-8) — aktor uji root
  memegang `*`.
- Kode: `56f78f8`.

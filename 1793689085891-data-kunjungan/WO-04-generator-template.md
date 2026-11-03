---
title: WO-04 — Generator template impor kunjungan
description: Endpoint GET /api/admin/visit-template (xlsx/csv) digenerate dari fields.yaml; header = kolom sumber Notion agar round-trip mulus.
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-04 — Generator template impor kunjungan

**Prasyarat:** WO-02.

## Lingkup

1. `schema/fields.yaml` → blok `template:` dengan kolom **header Indonesia sesuai CSV sumber**: `Nama`, `Alamat`, `Jumlah Pengunjung`, `Nomor HP`, `Status Kunjungan`, `Waktu Kunjungan` (D-9: round-trip Notion→sistem mulus).
2. `services/template.go` (pola hsh/sec): `GenerateTemplate` xlsx (sheet PETUNJUK/DATA/REFERENSI(hidden)/CONTOH) + csv (BOM+header). Dropdown REFERENSI: Status Kunjungan (5 enum).
3. Route flat `GET /api/admin/visit-template?format=xlsx|csv` (guard `visit.read`) — didaftarkan SEBELUM wildcard `/:id`.
4. `Jumlah Pengunjung` bebas teks (tidak ada dropdown) — hanya instruksi contoh format `2-4` di sheet PETUNJUK.

## Acceptance Criteria

- AC-1: Unit test — xlsx valid (4 sheet, dropdown dari REFERENSI), csv BOM+header, `format=pdf` → 400, tanpa token 401.
- AC-2: curl — xlsx 200 ukuran wajar & bisa dibuka; csv 200.
- AC-3: `template_version` tercantum di PETUNJUK (dipakai validator WO-05).

## Realisasi

- `schema/fields.yaml`: blok `template:` v1 dengan 6 kolom (header Notion apa adanya). `Nama` satu-satunya `required: true`.
- `services/template.go`: `GenerateTemplate(TemplateOptions{Format})` → `GeneratedTemplate{Filename, ContentType, Data}`. xlsx: sheet `PETUNJUK`, `REFERENSI` (hidden, A1 label + 5 kode enum), `DATA`, `CONTOH`. Dropdown hanya di `E2:E501` (`REFERENSI!$A$2:$A$6`). CSV: BOM + satu baris header.
- `handlers/template.go`: `DownloadTemplate` — `?format=` kosong → xlsx, `pdf`/lain → `400 visit.invalid`.
- `plugin.go`: route flat `GET /api/admin/visit-template` guard `visit.read`.

## Bukti

**Commit:** `f6af156` — `feat(visit): generator template impor xlsx/csv dari fields.yaml (WO-04)` (5 file, +592).

**AC-1 (unit test):** `go test ./plugins/visit/...` → `ok services 0.157s`. `template_test.go`:
- `TestGenerateTemplateXLSX`: 4 sheet ada, `REFERENSI` tersembunyi, header DATA = `Nama * / Alamat / Jumlah Pengunjung / Nomor HP / Status Kunjungan / Waktu Kunjungan`, data validation di `E2`+ berformula `REFERENSI!`, `template_version = 1` di PETUNJUK.
- `TestGenerateTemplateCSV`: prefix BOM `\ufeff`, tepat 1 baris header.
- `TestGenerateTemplateRejectsUnknownFormat`: `pdf` → `ErrInvalid`.
- `TestGenerateTemplateDefaultIsXLSX`: format kosong → `.xlsx`.

**AC-2 (curl, server lokal port 8082, token root):**
```
xlsx: 200 8971 application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
csv : 200 77   text/csv; charset=utf-8
pdf : 400
no token: 401
```
Isi xlsx (openpyxl): `sheets ['PETUNJUK','REFERENSI','DATA','CONTOH']`, `REFERENSI visible=hidden`, `dvs [(E2:E501, REFERENSI!$A$2:$A$6)]`, CONTOH berisi 3 baris (Miki Phoon/Singapore, Ni Luh Ayu/Denpasar, Anna Müller/Germany).

**AC-3:** PETUNJUK baris `template_version | 1` terverifikasi di test dan via pembacaan file.

**Suite penuh:** `go test ./...` hijau (grep kosong), `gofmt -l plugins/visit` kosong.


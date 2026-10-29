---
title: "WO-04: Generator template Excel/CSV"
description: "Endpoint GET /admin/sec-template?format=xlsx|csv yang digenerate dari schema/fields.yaml, lengkap dropdown REFERENSI."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - template
---

# WO-04: Generator template Excel/CSV

## Referensi
- SPEC: S-11, S-16, S-17, §Kontrak Template Excel/CSV.
- Pola acuan: generator template `plugins/hsh` WO-04 (`services/template.go`), `github.com/xuri/excelize/v2`.

## Tugas
1. **Service** `services/template.go` (`GenerateTemplate(format)`):
   - XLSX: 4 sheet — **PETUNJUK / DATA / REFERENSI (hidden) / CONTOH**.
   - Kolom sheet DATA (header Bahasa Indonesia): `Periode`, `Kategori`, `Nama Kebutuhan`, `Merk/Brand`,
     `Spesifikasi`, `Satuan`, `Harga Satuan`, `Status`, `Jumlah`, `Catatan`.
   - Kolom diturunkan **dari `fields.yaml`** (`Template.Columns`), tanpa daftar hard-code.
   - Dropdown (Excel data validation) untuk `Kategori`/`Satuan`/`Status` via rumus
     `REFERENSI!$X$2:$X$n` (pola `hsh`/`resident`, N-14). `Kategori` = data DB aktif → query kategori.
   - `template_version` ditulis di sheet PETUNJUK.
   - CSV: BOM + header (tanpa dropdown).
2. **Handler** `handlers/template.go` + route **flat** `GET /api/admin/sec-template?format=xlsx|csv`
   (guard `sec.read`). Path flat agar tidak bentrok wildcard `/:id`.
3. **Test**: xlsx valid (bisa dibuka `excelize`), 4 sheet, jumlah kolom DATA, ada data validation;
   `format=pdf` → 400.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/sec/...` hijau.
- curl `GET /api/admin/sec-template?format=xlsx` → 200, file valid; `format=csv` → BOM+header;
  `format=pdf` → 400; tanpa token → 401.

## Kriteria Selesai
- Template 100% digenerate dari `fields.yaml`; ubah kolom cukup edit `fields.yaml`.
- Dropdown kategori/satuan/status berfungsi; versi template tercatat.

## Bukti Pelaksanaan
_(diisi saat WO dikerjakan)_

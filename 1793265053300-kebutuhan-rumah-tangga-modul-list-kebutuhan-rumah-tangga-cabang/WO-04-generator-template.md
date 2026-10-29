---
title: "WO-04: Generator template (XLSX/CSV)"
description: "Endpoint unduh template hsh digenerate dari schema/fields.yaml, dengan dropdown REFERENSI untuk kategori/satuan/status."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - template
  - excel
---

# WO-04: Generator template (XLSX/CSV)

## Referensi
- SPEC: N-11, N-13, N-14, §Kontrak Template Excel/CSV.
- Pola acuan: `plugins/resident/services/template.go` + `handlers/template.go`
  (4 sheet PETUNJUK/DATA/REFERENSI/CONTOH, dropdown via `REFERENSI!$X$2:$X$n`, `template_version`);
  dependensi `github.com/xuri/excelize/v2` (sudah ada).

## Tugas
1. **`services/template.go`** — `GenerateTemplate(format, noPII)` menurunkan kolom dari
   `schema/fields.yaml` `template.columns` (**tanpa** daftar kolom hard-code).
2. **XLSX** — 4 sheet:
   - **PETUNJUK**: cara pakai + `template_version`.
   - **DATA**: kolom (header Indonesia) → `Periode`, `Kategori`, `Nama Kebutuhan`, `Satuan`,
     `Harga Satuan`, `Status`, `Jumlah`, `Catatan`. Baris 2 dibiarkan kosong untuk entri.
   - **REFERENSI**: daftar **Kategori** (7), **Satuan** (statis), **Status** (`URGENT/ENOUGH/UNKNOWN`
     dengan label Indonesia). Sheet disembunyikan.
   - **CONTOH**: 2–3 baris contoh valid.
   - **Dropdown**: data validation pada kolom Kategori, Satuan, Status (rumus ke `REFERENSI`).
3. **CSV** — BOM + header (tanpa dropdown; dokumentasikan agar operator memakai XLSX bila butuh dropdown).
4. **Handler + route**:
   | Method | Path | Guard |
   |---|---|---|
   | GET | `/admin/hsh-template?format=xlsx\|csv` | read |
   Path **flat** agar tidak bentrok wildcard `/:id` (pola WO-08 `resident`).
   `Content-Disposition` + `Content-Type` benar; format tak dikenal → 400.
5. **Test**: xlsx valid (dibuka `excelize`), jumlah kolom benar, dropdown ada, csv BOM+header, bad format 400.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/hsh/...` hijau.
- curl `format=xlsx` → 200, file valid (byte + dibuka ulang), `format=csv` → 200; `format=pdf` → 400.
- Tunjukkan jumlah kolom DATA = 8 dan keberadaan dropdown (mis. via `excelize` `GetDataValidations`).
- `template_version` tercatat di PETUNJUK.

## Kriteria Selesai
- Template 100% diturunkan dari `fields.yaml` (menambah kolom = edit yaml, bukan kode).
- Dropdown Kategori/Satuan/Status berfungsi di XLSX.
- Route ter-guard `hsh.read`.

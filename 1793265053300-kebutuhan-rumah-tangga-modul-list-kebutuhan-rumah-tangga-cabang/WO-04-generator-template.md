---
title: "WO-04: Generator template (XLSX/CSV)"
description: "Endpoint unduh template hsh digenerate dari schema/fields.yaml, dengan dropdown REFERENSI untuk kategori/satuan/status."
status: complete
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

## Bukti Pelaksanaan
- **Commit kode:** `18eaf8f` — `feat(hsh): generator template XLSX/CSV dengan dropdown REFERENSI (WO-04)`.
  File: `plugins/hsh/services/template.go`, `plugins/hsh/services/template_test.go`,
  `plugins/hsh/services/template_db_test.go`, `plugins/hsh/handlers/template.go`, `plugins/hsh/plugin.go`.
- **Kualitas:** `gofmt -l` bersih, `go build ./...` hijau, `go vet ./plugins/hsh/...` bersih,
  `go test ./plugins/hsh/...` hijau. Semua test template benar-benar RUN (bukan skip),
  termasuk `TestTemplateCategoryDropdownFromDB`.
- **Implementasi kunci:**
  - Tidak ada daftar kolom hard-code: `resolveTemplateColumns()` membaca
    `schema.Template.Columns` + `LookupField` → `Ref/Header/Required/Type/Enum/RefKey`.
    Menambah kolom = edit `fields.yaml` saja.
  - `referenceKeyFor()` memetakan sumber dropdown: `hsh_categories.code` → daftar **kategori aktif
    dari DB** (data, bukan skema), `hsh_items.unit` → `units` (N-13), `need_status` → enum (N-8).
    Kolom lain (Periode, Nama Kebutuhan, harga, jumlah, catatan) tetap teks bebas (N-16).
  - Sheet `REFERENSI` disembunyikan, header ramah: `Kategori | Satuan | Status`.
  - Nama berkas: `template_hsh_v<version>.xlsx|csv` (`Content-Disposition` + `Content-Type` benar).
  - Route **flat** `GET /api/admin/hsh-template` guard `hsh.read` (tidak bisa tertelan wildcard `/:id`).
- **curl (server :8082):**

  | Permintaan | Hasil |
  |---|---|
  | `?format=xlsx` | **200**, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `attachment; filename="template_hsh_v1.xlsx"`, 10.908 byte, `file` → `Microsoft Excel 2007+` |
  | `?format=csv` | **200**, `text/csv; charset=utf-8`, byte awal `EF BB BF` (BOM) lalu `Periode *,Kategori *,…` |
  | `?format=pdf` | **400** `{"error":"hsh: data tidak valid: format template \"pdf\" tidak dikenal (pakai xlsx|csv)"}` |
  | tanpa token | **401** |

- **Isi XLSX (dibuka ulang dengan `excelize`):**
  - Sheet: `[PETUNJUK REFERENSI DATA CONTOH]`, `REFERENSI` berstatus **hidden**.
  - Header DATA = **8 kolom**: `Periode *`, `Kategori *`, `Nama Kebutuhan *`, `Satuan`,
    `Harga Satuan`, `Status *`, `Jumlah`, `Catatan`.
  - **3 dropdown** (`GetDataValidations` → 3):
    `B2:B201 → REFERENSI!$A$2:$A$8` (7 kategori aktif dari DB),
    `D2:D201 → REFERENSI!$B$2:$B$13` (12 satuan),
    `F2:F201 → REFERENSI!$C$2:$C$4` (URGENT/ENOUGH/UNKNOWN).
  - `PETUNJUK!B3` = `template_version` → **1**; sheet CONTOH berisi 3 baris fiktif.
- **Dampak di luar plugin:** tidak ada (semua perubahan di `plugins/hsh/`).
- Konsumen berikutnya: WO-05 (validator) memakai `resolveTemplateColumns()`/`schema.Template`
  yang sama, sehingga header template dan pembaca unggahan tidak bisa melenceng.

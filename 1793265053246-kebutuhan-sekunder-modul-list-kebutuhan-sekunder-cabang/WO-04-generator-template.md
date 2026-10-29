---
title: "WO-04: Generator template Excel/CSV"
description: "Endpoint GET /admin/sec-template?format=xlsx|csv yang digenerate dari schema/fields.yaml, lengkap dropdown REFERENSI."
status: complete
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

**File yang dibuat/diubah**
- `plugins/sec/services/template.go` (baru) — kembar `hsh` dengan prefix `sec`:
  `GenerateTemplate(TemplateOptions{Format})` → `GeneratedTemplate{Filename, ContentType, Data}`;
  `resolveTemplateColumns()` memetakan `schema.Template.Columns` → `TemplateColumn{Ref,Header,Required,Type,Enum,RefKey}`
  (**tanpa daftar kolom hard-code**; `field` yang tidak ada di `fields.yaml` → `ErrInvalid`);
  `templateReferenceKeyFor` memetakan `sec_categories.code`→kategori, `sec_items.unit`→units,
  `need_status`→status; `referenceCategories()` membaca `sec_categories WHERE is_active` (data DB,
  urut `sort_order, code`), `referenceLists` memakai `schema.UnitCodes()` dan `schema.EnumCodes("need_status")`.
  XLSX: sheet `PETUNJUK` / `REFERENSI` (**hidden**) / `DATA` (header + 200 baris terformat) / `CONTOH`
  (3 baris contoh domain `sec`: kipas angin, Sepeda BMX, setrika); dropdown via
  `dv.SetSqrefDropList("REFERENSI!$X$2:$X$n")`; kolom bertipe `numeric*` dapat format `#,##0.00`;
  `template_version` ditulis di PETUNJUK. CSV: BOM UTF-8 + header saja (tanpa dropdown).
- `plugins/sec/handlers/template.go` (baru) — `DownloadTemplate` + anotasi swagger `@Tags sec`,
  `Content-Disposition: attachment; filename="template_sec_v<N>.xlsx|csv"`.
- `plugins/sec/plugin.go` (diubah) — route **flat** `GET /api/admin/sec-template` (guard `sec.read`),
  didaftarkan setelah blok entries agar tidak bentrok dengan wildcard `/:id`.
- `plugins/sec/services/template_test.go` (baru) — 6 test.

**Gate statis**
```
go build ./...            -> OK
go vet ./plugins/sec/...  -> OK
gofmt -l plugins/sec      -> (kosong)
go test ./plugins/sec/... -> ok (services 0.138s)
```

**Test yang dijalankan**
```
=== RUN   TestGenerateTemplateXLSX        --- PASS (0.02s)
=== RUN   TestGenerateTemplateCSV         --- PASS
=== RUN   TestGenerateTemplateDefaultsToXLSX --- PASS
=== RUN   TestGenerateTemplateBadFormat   --- PASS
=== RUN   TestTemplateReferenceKeyFor     --- PASS
=== RUN   TestTemplateColumnsCoverSchema  --- PASS
```
Isi yang diperiksa test: 4 sheet ada, `REFERENSI` hidden, header DATA = `fields.yaml` (10 kolom,
termasuk tanda `*` untuk wajib), dropdown `Satuan`/`Status` menunjuk `REFERENSI!$`, `Periode` dan
`Nama Kebutuhan` **tanpa** dropdown, `template_version` ada di PETUNJUK, CONTOH 4 baris, REFERENSI
memuat label Kategori/Satuan/Status + URGENT/ENOUGH/UNKNOWN.

**Verifikasi HTTP (server di-restart dari binary baru)**
| Permintaan | Hasil |
|---|---|
| `GET /api/admin/sec-template?format=xlsx` | **200**, 10.953 byte, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |
| `GET /api/admin/sec-template?format=csv` | **200**, 106 byte (BOM + header) |
| `GET ...?format=pdf` | **400** |
| tanpa token | **401** |

Header CSV yang dihasilkan:
`Periode *,Kategori *,Nama Kebutuhan *,Merk/Brand,Spesifikasi,Satuan,Harga Satuan,Status,Jumlah,Catatan`

Inspeksi artefak XLSX (zip/XML):
- sheet: `PETUNJUK`, `REFERENSI` (state=`hidden`), `DATA`, `CONTOH`
- `DATA` dropdown: `B2:B201 → REFERENSI!$A$2:$A$3` (Kategori = 2 kategori seed),
  `F2:F201 → REFERENSI!$B$2:$B$9` (8 satuan), `H2:H201 → REFERENSI!$C$2:$C$4` (3 status)
- `sharedStrings` memuat `elektronik-peralatan`, `mainan-rekreasi`, pcs/unit/set/box/pak/roll/lusin/meter

**Catatan temuan**
- Dropdown **Kategori** berasal dari data DB (bukan `fields.yaml`), jadi tanpa DB (unit test) hanya 2
dropdown yang muncul — perilaku ini memang diharapkan dan diuji pada `New(nil)`.
- Helper lokal diberi nama `templateStrPtr`/`columnIndex`/`flattenRows` karena `strPtr`/`boolPtr` sudah
  dipakai test lain dalam paket yang sama.
- Token uji di `/tmp/root.token` kedaluwarsa setelah server restart → ambil ulang via
  `POST /api/admin/login` sebelum verifikasi curl.

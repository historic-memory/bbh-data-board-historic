---
title: WO-04 — Generator template Excel/CSV
description: "GET /api/admin/wh-template dari fields.yaml"
status: complete
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-04 — Generator template Excel/CSV

## Prasyarat

WO-03 lulus.

## Lingkup

- `services/template.go` + `handlers/template.go` (pola hsh/sec WO-04): `GET /api/admin/wh-template?format=xlsx|csv` guard `warehouse.read`; path flat agar tak bentrok `/wh/*`.
- Template dari `schema/fields.yaml` — TANPA hard-code daftar kolom. Sheet PETUNJUK/DATA/REFERENSI(hidden)/CONTOH. Kolom DATA: Kategori, Produk, Gudang, Qty, Satuan, Tanggal Masuk, Kadaluarsa, Catatan (final di WO-06 impor). Dropdown REFERENSI untuk Kategori (data DB), Satuan, Gudang (data DB).
- CSV = BOM + header. `template_version` di sheet PETUNJUK.
- G-4 terjawab via EX-1 (owner "sertakan excel") — masuk kolom Kadaluarsa.

## Acceptance Criteria

- AC-1 xlsx 200 valid (inspeksi dropdown `REFERENSI!$X$2:$X$n`), csv 200 BOM; `format=pdf` → 400; tanpa token 401; viewer 200.
- AC-2 Unit test + Bruno template.
- AC-3 Gotcha: excelize `DataValidation.Sqref` bertipe string (split `:`).

## Catatan Implementasi (Dev)

### A. Generator dari `fields.yaml` (D-9)

- `plugins/warehouse/services/template.go:65` — `GenerateTemplate(opts)` adalah satu-satunya pintu keluar artefak: XLSX default, CSV fallback, format lain → `fmt.Errorf("%w: format template %q tidak dikenal (pakai xlsx|csv)", ErrInvalid, ...)` → 400. Tidak ada cabang "tebak format dari nama berkas" supaya kontraknya eksplisit dan bisa diuji.
- `plugins/warehouse/services/template.go:101` — `resolveTemplateColumns()` membaca `s.schema.Template.Columns` dan memvalidasi tiap `field` lewat `schema.LookupField`; `field` yang tidak ada → `ErrInvalid`, bukan kolom yang diam-diam hilang. Ini yang membuat "TANPA hard-code daftar kolom" (lingkup WO-04) benar-benar berlaku: menambah kolom = edit `fields.yaml`, bukan edit Go.
- `plugins/warehouse/services/template.go:41` — `TemplateColumn{Ref, Header, Required, Type, Enum, RefKey}` memisahkan **nama kolom** (dari YAML) dari **cara render** (dropdown/format angka): header boleh berubah tanpa mengubah perilaku dropdown.
- `plugins/warehouse/services/template.go:153` — `headerCell` menambahkan `" *"` pada kolom `required`: penanda wajib ikut ke header XLSX **dan** baris header CSV, sehingga validator WO-05 dan mata manusia membaca aturan yang sama.
- `plugins/warehouse/services/template.go:388` — `isNumericType` memakai prefix kanonik `"numeric"` (bukan daftar tipe literal): tipe baru seperti `numeric12_2`/`numeric14_3` otomatis dapat format ribuan tanpa perubahan kode.
- `plugins/warehouse/services/template.go:438` — `buildSampleSheet` menulis 3 baris contoh fiktif dengan header yang sama persis: pengguna melihat "1 baris = 1 batch" tanpa perlu membaca PETUNJUK dulu.

### B. Dropdown REFERENSI (D-8, O-1, O-2)

- `plugins/warehouse/services/template.go:124` — `referenceKeyFor` menentukan sumber dropdown: `enum` di `fields.yaml` menang (Satuan, D-8), lalu `wh_categories.name` → Kategori, `wh_warehouses.name` → Gudang; sisanya teks bebas. Kolom tipe teks tidak pernah dipaksa jadi dropdown.
- `plugins/warehouse/services/template.go:193` — `referenceCategoryNames` = `SELECT name FROM wh_categories WHERE is_active = TRUE ORDER BY LOWER(name)`: hanya kategori aktif, urut huruf kecil sehingga `ABC` dan `abc` tidak terpisah jauh; konsisten dengan `is_active` yang dipakai CRUD WO-02.
- `plugins/warehouse/services/template.go:209` — `referenceWarehouseNames` = `SELECT name FROM wh_warehouses ORDER BY LOWER(name)` (tanpa filter aktif karena gudang tidak punya `is_active`, SPEC §2). Komentar di kode menyebut O-2: importer fail-closed 400 untuk gudang tak dikenal, jadi dropdown ini **adalah** daftar yang akan diterima — pengguna tidak bisa menyiapkan berkas yang pasti ditolak.
- `plugins/warehouse/services/template.go:162` — `referenceLists` mengumpulkan hanya key yang **benar-benar dipakai** kolom DATA, urut tetap `kategori, satuan, gudang` lalu enum lain terurut: output deterministik, jadi tes bisa membandingkan byte/ranges tanpa flaky.
- `plugins/warehouse/services/template.go:193` — tanpa handle DB (`s.db == nil`, unit test tanpa Postgres) atau tabel kosong, kategori/gudang → `nil`; **bukan** error. Template tetap bisa di-generate (dan itu yang diuji di `TestGenerateTemplateXLSXWithoutDB`).
- `plugins/warehouse/services/template.go:361` — dropdown hanya dibuat bila `rc.Count > 0`. Tanpa penjaga ini, REFERENSI kosong menghasilkan `SetSqrefDropList("REFERENSI!$A$2:$A$1")` — rentang terbalik yang ditolak Excel; jadi "tidak ada data" berarti "tidak ada dropdown", bukan berkas rusak.
- `plugins/warehouse/services/template.go:242` — REFERENSI ditulis **sebelum** DATA supaya `buildDataSheet` bisa menunjuk rentang nyata (`$A$2:$A$n`), lalu `SetSheetVisible(templateRefSheet, false)` di akhir: daftar baku tersembunyi tapi tetap hidup sebagai sumber validasi.

### C. CSV & struktur berkas

- `plugins/warehouse/services/template.go:222` — `buildCSV` menulis BOM `\ufeff` lalu satu baris header lewat `encoding/csv` (quoting otomatis). BOM dipakai supaya Excel Windows membaca UTF-8 (nama kategori Indonesia) tanpa mojibake.
- `plugins/warehouse/services/template.go:222` — CSV **hanya header**, tanpa baris kosong dan tanpa dropdown: format CSV tidak mendukung validasi daftar, jadi template ini jujur soal batasnya (dinyatakan juga di sheet PETUNJUK baris `CSV`).
- `plugins/warehouse/services/template.go:77` — nama berkas dari `s.schema.Template.Version` (`template_wh_v1.xlsx` / `.csv`): versi template melekat di nama berkas sehingga berkas lama yang diunggah balik bisa dikenali versionya.
- `plugins/warehouse/handlers/template.go:27` — `DownloadTemplate` hanya mengambil `format` via `c.DefaultQuery` (default `xlsx`), meneruskan ke layanan, lalu `Content-Disposition: attachment; filename="..."` + `c.Data(...)`. Tidak ada logika template di handler, jadi tidak ada aturan yang bisa berbeda antara HTTP dan unit test.
- `plugins/warehouse/handlers/template.go:17` — godoc `@Tags warehouse`, `@Security BearerAuth`, `@Produce` xlsx+csv, `@Failure 400/401/403`: entry swagger dibuat sekarang meski file swagger dibangun ulang di WO-08.
- `plugins/warehouse/plugin.go:110` — route didaftarkan **flat** `/admin/wh-template` (bukan `/admin/wh/template`) dengan guard `warehouse.read`, meniru `plugins/hsh/plugin.go:107`. Alasannya ditulis sebagai komentar di kode: path bersarang akan masuk ruang wildcard `/admin/wh/*` dan bentrok dengan rute stok.
- `plugins/warehouse/plugin.go:110` — guard `read` (bukan `write`): template adalah dokumen, bukan mutasi data, jadi `viewer` boleh mengunduh — dan itu diuji.

### D. Test

- `plugins/warehouse/services/template_test.go:88` — `TestGenerateTemplateXLSXWithoutDB` mengunci struktur berkas tanpa Postgres: 4 sheet, `REFERENSI` `state="hidden"`, header DATA dari `fields.yaml`, `Tanggal Masuk` teks bebas (bukan dropdown), PETUNJUK memuat `template_version` + seluruh kolom, CONTOH = header + 3 baris.
- `plugins/warehouse/services/template_test.go:67` — `dropdownFirstCells` membaca nilai `Sqref` sebagai **string** lalu `strings.Split(dv.Sqref, ":")[0]`: gotcha AC-3. `Sqref` bukan slice, jadi pemakaian `structs`/`[]string` di sini akan salah tipe.
- `plugins/warehouse/services/template_test.go:260` — `TestGenerateTemplateDropdownsFollowCatalogue` (DB-backed) menambah 1 kategori + 1 gudang, generate, lalu memastikan dropdown menunjuk nama yang baru dibuat dan `REFERENSI` memuatnya; fixture dibersihkan berurutan (categories → warehouses) lewat `t.Cleanup`.
- `plugins/warehouse/services/stock_db_test.go:89` — helper `warehouseName(t)` ditambahkan pada fixture WO-03 (dibaca langsung dari DB) supaya tes template tidak perlu menebak nama gudang atau menyalin string fixture.
- `plugins/warehouse/services/template_test.go:236` — `TestReferenceKeyForMapsDropdownSources` menguji pemetaan `enum`/`wh_categories.name`/`wh_warehouses.name` → key, dan `wh_batches.note` → `""` (teks bebas): perubahan pemetaan dropdown tertangkap tanpa harus generate XLSX.
- `plugins/warehouse/services/template_test.go:228` — `TestGenerateTemplateRejectsUnknownFormat` memastikan `pdf` → error `ErrInvalid` (bukan template kosong). `TestGenerateTemplateCSVHeaderOnlyWithBOM` (`:183`) memeriksa 3 byte pertama `ef bb bf` dan jumlah kolom header. `TestGenerateTemplateDefaultsToXLSX` (`:217`) mengunci default format kosong → xlsx.
- `api-tests/warehouse/01-Download-template-xlsx.bru` — request Bruno: 200, content-type xlsx, `content-disposition` memuat `template_wh_v` + `.xlsx`, body non-kosong.
- `api-tests/warehouse/02-Download-template-csv.bru` — request Bruno: 200, `text/csv`, body memuat seluruh header termasuk penanda `*` dan kolom opsional `Kadaluarsa`/`Catatan`.
- `api-tests/warehouse/03-Template-unsupported-format.bru` — request Bruno negatif: `?format=pdf` → **400** + pesan memuat `xlsx|csv`.

### Bukti

Unit test (container `bbb-data-board-app-1`, DB nyata):

```
go build ./...                                  # OK
go test -count=1 -run "TestGenerateTemplate|TestReferenceKeyFor" ./plugins/warehouse/services/
--- PASS: TestGenerateTemplateXLSXWithoutDB (0.01s)
--- PASS: TestGenerateTemplateCSVHeaderOnlyWithBOM (0.00s)
--- PASS: TestGenerateTemplateDefaultsToXLSX (0.00s)
--- PASS: TestGenerateTemplateRejectsUnknownFormat (0.00s)
--- PASS: TestReferenceKeyForMapsDropdownSources (0.00s)
--- PASS: TestGenerateTemplateDropdownsFollowCatalogue (0.05s)
ok  github.com/rolldone/donisgo/plugins/warehouse/services  0.074s
gofmt -l plugins/warehouse/                     # (kosong)
```

Bruno (`cd api-tests && bru run auth/00-Login.bru warehouse -r --env Local`):

```
warehouse/01-Download-template-xlsx (200 OK)          3/3 tests ✓
warehouse/02-Download-template-csv (200 OK)           3/3 tests ✓
warehouse/03-Template-unsupported-format (400)        2/2 tests ✓
Requests 4 (4 Passed) · Tests 10/10 · Status PASS
```

Matriks curl (katalog berisi 1 kategori + 1 gudang):

| Kasus | Hasil |
|---|---|
| `GET /wh-template` (admin) | **200**, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `filename="template_wh_v1.xlsx"`, 10.169 byte, `unzip` valid |
| `GET /wh-template?format=xlsx` | **200** (default sama, AC-1) |
| `GET /wh-template?format=csv` | **200**, `text/csv; charset=utf-8`, 89 byte, 3 byte pertama `ef bb bf` (BOM), 1 baris header |
| `GET /wh-template?format=pdf` | **400** `format template "pdf" tidak dikenal (pakai xlsx\|csv)` |
| `GET /wh-template` tanpa token | **401** |
| `GET /wh-template` (viewer) | **200** xlsx · **200** csv · **400** pdf (guard `warehouse.read`) |

Inspeksi XML XLSX (`unzip` + parser host):

```
workbook.xml : PETUNJUK · REFERENSI(state="hidden") · DATA(activeTab) · CONTOH
DATA!r1      : Kategori * | Produk * | Gudang * | Jumlah Masuk * | Satuan | Tanggal Masuk * | Kadaluarsa | Catatan
dataValidation: A2:A201 => REFERENSI!$A$2:$A$2   (Kategori, 1 nilai)
                C2:C201 => REFERENSI!$C$2:$C$2   (Gudang, 1 nilai)
                E2:E201 => REFERENSI!$B$2:$B$10  (Satuan, 9 nilai dari fields.yaml)
REFERENSI!r1 : Kategori | Satuan | Gudang
CONTOH       : 4 baris (header + 3 contoh), tanpa dropdown
PETUNJUK     : 25 baris (template_version=1 + aturan + daftar kolom)
```

Catatan bukti: tanpa data DB, dropdown Kategori & Gudang **tidak dibuat** (bukan rentang kosong) — hanya Satuan yang tersisa; ini perilaku yang dikunci `TestGenerateTemplateXLSXWithoutDB`.

Kebersihan: data uji dihapus setelah verifikasi → `cats=0 whs=0 items=0 batches=0 movements=0`, admin uji `0`. Tanpa `push`. `sub_app/webapp/src/layouts/AdminLayout.tsx` dan `sub_app/webapp/src/styles/index.css` tidak disentuh (milik topik lain, tetap uncommitted).


## Review Manager (2026-11-23) — LULUS ✅

Verifikasi independen Manager:

| Cek | Hasil |
|---|---|
| `git show --stat 739c9b0` | Scope bersih: `plugins/warehouse/**` + `api-tests/warehouse/**` saja ✅ |
| `go test -count=1` (container) | `services` + `schema` ok ✅ |
| Gotcha Sqref | Ditegakkan (`strings.Split(dv.Sqref, ":")[0]`) + test ✅ |
| Data uji | `wh_*` 0/0/0/0/0 bersih ✅ |
| Proses | 4 commit lokal tanpa push; AdminLayout/index.css tak disentuh ✅ |
| Dokumen | wo-04 complete (open+closed), master index terisi ✅ |

**Catatan review:**
1. Path FLAT `/admin/wh-template` benar (terhindar dari wildcard `/wh/*`).
2. Sumber kosong → dropdown dilewati (bukan rentang `$A$2:$A$1`) — sesuai pola `referenceIndex()`; pastikan WO-05 validator juga mengikuti pola ini.
3. Kategori dropdown dari DB, Satuan dari fields.yaml, Gudang dari DB — konsisten D-8/O-2.

**WO-04 DITERIMA. WO-05 DIBUKA — mulai sekarang.**

Jabatan: Manager Agent — Nama Tugas: Review WO-04 warehouse. Sayonara.

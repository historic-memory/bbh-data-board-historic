---
title: "WO-05: Validator umpan-balik"
description: "Endpoint POST /admin/sec-template/validate read-only: memeriksa berkas unggahan per baris dan mengembalikan umpan-balik."
status: complete
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - validation
---

# WO-05: Validator umpan-balik

## Referensi
- SPEC: S-11, S-17, S-18, §Alur Impor, GAP-1.
- Pola acuan: validator `plugins/hsh` WO-05 / `plugins/resident` WO-09 — aturan 100% dari `fields.yaml`.

## Tugas
1. **`schema`**: tambah dukungan blok `validation:` di `fields.yaml` (kinds: `regex`, `in_reference`,
   `warn_when_in`). `RulesFor`/`EnumCodes` helper.
   - `Periode` → `regex` `^\d{4}-(0[1-9]|1[0-2])$`.
   - `Satuan` → `in_reference` (units).
   - `Kategori` → `in_reference` (categories, dari DB aktif).
   - `Status` → label ID → code.
2. **Service** `services/validate.go`:
   - parse `.xlsx`/`.csv`; **tolak `.xls`** via ekstensi/magic OLE2 (S-18).
   - `template_version` mismatch → error file-level.
   - `parseNumeric` toleran pemisah ribuan; baris kosong diabaikan.
   - umpan-balik: xlsx → sheet HASIL+RINGKASAN; csv → + kolom `hasil_validasi`/`alasan_ditolak`.
   - `referenceIndex()`: sumber `categories` hanya didaftarkan bila tidak kosong (kalau tidak, semua baris ditolak).
3. **Handler** `handlers/validate.go` + route `POST /api/admin/sec-template/validate`
   (multipart `file`, `?format=`, `?strict=1`→422; guard `sec.read`). **Read-only** (tak menulis DB).
   Header ringkasan `X-Validate-Total/Valid/Rejected` + log stdout.
4. **Fixture** `api-tests/fixtures/sec-broken-template.csv` (beberapa baris error).

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/sec/...` hijau.
- curl: 9 baris → total/valid/ditolak; 7 kategori error; versi template usang → file-level reject;
  non-strict 200 / strict 422 / no-token 401 / viewer 200.

## Kriteria Selesai
- 100% aturan dari `fields.yaml`; tidak menulis DB saat validasi.
- `.xls` ditolak jelas; feedback memakai header asli.

## Bukti Pelaksanaan

**Catatan awal:** blok `validation:` di `plugins/sec/schema/fields.yaml` sudah dibuat sejak WO-01
(`regex` untuk `sec_periods.code`, `in_reference` untuk `sec_items.unit` → `units` dan
`sec_categories.code` → `categories`) bersama tipe `ValidationRule`/`Validation` + helper
`RulesFor`/`EnumCodes` di `schema/schema.go`. WO-05 hanya menambah service + handler + route.

**File yang dibuat/diubah**
- `plugins/sec/services/validate.go` (baru) — kembar `hsh` dengan prefix/istilah `sec`:
  `ValidateTemplate(name, data, ValidateOptions)` → `ValidateResult{Report, Feedback}`;
  `isLegacyXLS` menolak `.xls` via ekstensi **atau** magic OLE2 (`D0 CF 11 E0 A1 B1 1A E1`, S-18);
  `parseInput` mendeteksi XLSX (magic `PK`) vs CSV (BOM dibuang); `readTemplateVersion` membaca
  `PETUNJUK`; `buildColSpecs` mencocokkan header ter-`normHeader` (trim + buang `*` + lowercase),
  kolom wajib yang hilang → error file-level; `referenceIndex()` **tidak** mendaftarkan sumber
  `categories` bila daftar kosong (tanpa DB), supaya tidak menolak semua baris;
  `validateRow` memeriksa required / enum (terima code **atau** label ID via `statusLabelToCode` +
  `normalizeLabel`) / `numeric*` (≥ 0, toleran pemisah ribuan) / rule deklaratif dari `fields.yaml`;
  umpan-balik xlsx = sheet `HASIL` + `RINGKASAN` (baris ditolak diberi fill `F8CBAD`), csv = header
  asli + 2 kolom `hasil_validasi`/`alasan_ditolak`. Baris kosong diabaikan.
- `plugins/sec/handlers/validate.go` (baru) — `ValidateTemplate` (multipart `file` atau body mentah,
  limit 32 MiB), header `X-Validate-Total/Valid/Rejected`, log `[sec.validate] ...` ke stdout,
  `strict=1` → **422** saat ada baris ditolak. Anotasi swagger `@Tags sec`.
- `plugins/sec/plugin.go` (diubah) — `POST /api/admin/sec-template/validate` (guard `sec.read`,
  **read-only**: tidak menulis DB).
- `plugins/sec/services/validate_test.go` (baru) — 12 test.
- `api-tests/fixtures/sec-broken-template.csv` (baru) — 9 baris (2 valid, 7 error).

**Gate statis**
```
go build ./...            -> OK
go vet ./plugins/sec/...  -> OK
gofmt -l plugins/sec      -> (kosong)
go test ./plugins/sec/... -> ok (services 0.238s)
```

**Test yang dijalankan (12, semua PASS)**
`TestValidateTemplateValidRows`, `TestValidateTemplateRejections`,
`TestValidateTemplateStatusLabelAccepted`, `TestValidateTemplateVersionMismatch`,
`TestValidateTemplateMissingRequiredColumn`, `TestValidateTemplateRejectsLegacyXLS`,
`TestValidateTemplateEmptyUpload`, `TestValidateTemplateCSV`,
`TestValidateFeedbackFormatOverride`, `TestRuleFlags`, `TestReferenceIndexUnits`,
`TestNormalizeLabel`.

**Verifikasi HTTP**
| Skenario | Hasil |
|---|---|
| CSV fixture 9 baris | **200**; `X-Validate-Total: 9`, `X-Validate-Valid: 2`, `X-Validate-Rejected: 7` |
| XLSX terisi 2 baris valid | **200**; total 2 / valid 2 / ditolak 0; feedback `HASIL`+`RINGKASAN` |
| XLSX `template_version=99` | **200** (file-level); total 2 / valid 0 / ditolak 2; RINGKASAN memuat pesan file error |
|`.xls` (magic OLE2) | **400** `berkas .xls (Excel lama) tidak didukung; buka di Excel/LibreOffice lalu simpan sebagai .xlsx atau .csv` |
| `?strict=1` + 7 error | **422** |
| tanpa token | **401** |
| viewer (`sec.viewer5@test.local`) | validate **200**, download template **200** |

Tujuh kategori error yang terbukti dari fixture (satu per baris): Periode regex, Nama Kebutuhan wajib,
Satuan `karung` di luar REFERENSI, Status `BANYAK` di luar enum, Harga negatif, Jumlah `dua` bukan angka,
Kategori `perabot-dapur` tidak dikenal (via `in_reference [categories]`).

Umpan-balik CSV (baris ke-4 sebagai contoh):
`...,periode ditolak,ditolak,Periode: harus berformat YYYY-MM (mis. 2026-09) atau YYYY-MM-DD`

Log server (stdout) juga menulis ringkasan + `alasan_terbanyak`, mis.:
`[sec.validate] file=... total=9 valid=2 ditolak=7 template_version=-1 file_errors=[]`

**Read-only terbukti**: sesudah semua panggilan, `sec_periods/sec_items/sec_entries` = `0/0/0`
(tidak ada baris dibuat). Data uji (admin viewer) dihapus kembali.

**Catatan temuan**
- Fixture awal memakai label status `PERLU SEGERA` yang tidak cocok dengan label enum
  (`Urgent / Perlu segera`) → diganti dengan kasus kategori tidak dikenal agar 7 error terpenuhi.
- Untuk membuat fixture XLSX terisi, template diunduh dari API lalu diisi dengan `openpyxl`
  (bukan menyusun workbook manual) sehingga header/versi selalu sinkron dengan generator.

---
title: WO-05 — Validator umpan balik kunjungan
description: POST /api/admin/visit-template/validate — parse & laporan baris bermasalah tanpa menulis DB.
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-05 — Validator umpan balik kunjungan

**Prasyarat:** WO-04.

## Lingkup

1. `services/validate.go` (pola hsh/sec): tolak `.xls` (ekstensi/magic OLE2); parse xlsx/csv; `template_version` mismatch → file-level error.
2. Aturan validasi dari `fields.yaml` blok `validation:`:
   - `Nama` required.
   - `Nomor HP` — regex longgar (karakter `+0-9 ()-`); pola tak dikenal → **warning**, bukan penolakan (data sumber memang kacau).
   - `Jumlah Pengunjung` — bebas; tak terparse → **warning**.
   - `Status Kunjungan` — label `Not started`/`Done` (dan ID: `Belum dimulai`/`Selesai` bila ada) → map ke enum; tak dikenal → error.
   - `Waktu Kunjungan` — parse `DD Month YYYY HH:MM` bulan ID+EN; gagal → error; kosong → warning.
3. Feedback: xlsx (sheet HASIL+RINGKASAN) / csv (+`hasil_validasi`, `alasan_ditolak`); header `X-Validate-*`; `?strict=1` → 422.
4. Route `POST /api/admin/visit-template/validate` (guard `visit.read`), TIDAK menulis DB.
5. Fixture `api-tests/fixtures/visit-broken-template.csv`.

## Acceptance Criteria

- AC-1: Test — baris valid lulus; tanggal bulan `Mei`/`Agustus`/`Desember` terparse; format tak dikenal ditolak; HP aneh hanya warning.
- AC-2: curl — mixed file → total/valid/rejected benar; strict 422; tanpa token 401.

## Realisasi

- `schema/fields.yaml`: blok `validation:` (`date_format: "DD Month YYYY HH:MM"`) + 4 aturan — `phone_raw` regex (warning), `party_raw` regex (warning), `scheduled_at` rule `date` (error), `scheduled_at` rule `warn_when_empty` (warning).
- `services/parse.go` (dipakai ulang WO-06): `ParseVisitSchedule` (bulan ID+EN penuh/singkat, bentuk `DD Month YYYY [HH:MM]` & ISO, suffix `(GMT+8)` ditoleransi, validasi kalender, hasil UTC) dan `ParsePartySize` (hanya angka tunggal → `party_size`).
- `services/validate.go`: `ValidateTemplate(name, data, opts)` — tolak `.xls` (ekstensi + magic OLE2), parse XLSX (sheet DATA, versi dari PETUNJUK) / CSV, resolusi kolom dari `template.columns`, verifikasi `template_version`, validasi baris (required, enum + sinonim label `Not started`/`Done`/`Belum dimulai`/`Selesai`/…, rule YAML), umpan balik XLSX (`HASIL` + `RINGKASAN`) atau CSV (BOM + `hasil_validasi`, `alasan_ditolak`). Tidak menulis DB.
- `handlers/validate.go`: `POST /api/admin/visit-template/validate` — multipart `file` atau body mentah, `?format=xlsx|csv`, `?strict=1` → 422, header `X-Validate-Total/Valid/Rejected`, ringkasan dicatat ke log server.
- `plugin.go`: route didaftarkan setelah generator template, guard `visit.read`.
- `api-tests/fixtures/visit-broken-template.csv`: 9 baris campuran (3 valid bersih, 3 valid dengan warning, 3 ditolak).

## Bukti

**Commit:** `adb20bb` — `feat(visit): validator umpan balik template kunjungan (WO-05)` (8 file, +1211).

**AC-1 (unit test)** — `go test ./plugins/visit/...` hijau, seluruh `go test ./...` hijau, `gofmt -l plugins/visit` kosong:
- `TestParseVisitSchedule`: `5 Mei 2026 15:00` → `2026-05-05T07:00:00Z`, `17 Agustus 2026 10:00` → `2026-08-17T02:00:00Z`, `25 Desember 2026 08:30` → `2026-12-25T00:30:00Z`, `1 Januari 2027 00:00` → `2026-12-31T16:00:00Z`, ISO `2026-04-23 09:00` diterima; `23/04/2026 09:00`, bulan asing, dan `31 Februari 2026` ditolak.
- `TestValidateBrokenFixtureCSV`: total 9 / valid 6 / ditolak 3; baris `Nama` kosong ditolak; HP `(WA) 0812 7777 8888` **warning** dan baris tetap valid; status `Maybe` ditolak; tanggal `23/04/2026` ditolak; waktu kosong **warning** dan baris tetap valid.
- `TestValidateTemplateXLSXRoundTrip`: template hasil WO-04 valid terhadap validator sendiri, `template_version=1`, tanpa file error.
- `TestValidateTemplateRejectsLegacyXLS`: ekstensi `.xls` dan magic OLE2 ditolak.
- `TestValidateTemplateRejectsWrongVersion`: `template_version=2` → file-level error.
- `TestParsePartySize`: `2-4`, `Just myself`, `3 adult, 1 children` → bukan angka (D-12).
- `schema_test.go` → `TestValidationRulesResolve`: semua `field` pada blok `validation:` benar-benar ada dan punya pesan.

**AC-2 (curl, port 8082, token root, fixture campuran):**
```
mixed csv : 200 text/csv | X-Validate-Total: 9, X-Validate-Valid: 6, X-Validate-Rejected: 3
strict=1  : 422
format=xlsx: 200 HASIL + RINGKASAN
no token  : 401
```
Umpan balik CSV: BOM + header asli + `hasil_validasi`/`alasan_ditolak`; baris `2-4` → `valid` dengan `PERINGATAN Jumlah Pengunjung: ...`. RINGKASAN menampilkan `total 9 / valid 6 / ditolak 3` dan 3 alasan terbanyak. Log server mencatat ringkasan tiap run (`[visit.validate] ...`).

**Catatan WO-06:** importer wajib memakai `ParseVisitSchedule` + `ParsePartySize` yang sama, dan baris awal riwayat status harus memakai `effective_at` dari sumber (bukan waktu impor).


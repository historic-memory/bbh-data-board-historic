---
title: WO-05 — Validator template
description: "POST /api/admin/wh-template/validate"
status: complete
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-05 — Validator template

## Prasyarat

WO-04 lulus.

## Lingkup

- `services/validate.go` + `handlers/validate.go` (pola hsh/sec WO-05): `POST /api/admin/wh-template/validate` (multipart file; `?format=xlsx|csv`; `?strict=1` → 422) guard `warehouse.read`; TIDAK menulis DB.
- Aturan dari blok `validation:` di `fields.yaml`; tolak `.xls` via ekstensi/magic OLE2; `template_version` mismatch → file-level; header feedback pakai header ASLI; baris kosong diabaikan; tanggal `DD/MM/YYYY`.
- Feedback: xlsx sheet HASIL+RINGKASAN / csv + kolom `hasil_validasi`/`alasan_ditolak`. Header `X-Validate-Total/Valid/Rejected`.
- Fixture `api-tests/fixtures/wh-broken-template.csv`.

## Acceptance Criteria

- AC-1 Feedback baris benar (total/valid/rejected di header & body); strict 422.
- AC-2 `.xls` → 400; version mismatch → file-level reject; read-only terbukti (0 tulisan DB).

## Catatan Implementasi (Dev)

### A. Aturan (single source of truth)

- `plugins/warehouse/schema/fields.yaml:143` — blok `validation:` diperluas, bukan kode: validator TIDAK punya daftar aturan sendiri. Menambah aturan = edit YAML (pola D-9/WO-04). Header komentar blok menjelaskan jenis rule + arti `severity` supaya junior tidak perlu membaca Go untuk mengubah kebijakan.
- `plugins/warehouse/schema/fields.yaml:157` — rule baru `wh_batches.qty_in` → `min_value` (`min: 0`, `exclusive: true`): DB sudah punya CHECK `qty_in > 0` (WO-01), jadi angka 0/negatif harus ketahuan di validator, bukan baru muncul sebagai 400 saat impor WO-06.
- `plugins/warehouse/schema/fields.yaml:168` — rule baru `wh_categories.name` → `in_reference [categories]`, **`severity: warning`**: O-1 bilang kategori asing DIBUAT otomatis saat impor, jadi ketidakcocokan adalah informasi ("akan dibuat"), bukan alasan menolak baris.
- `plugins/warehouse/schema/fields.yaml:174` — `wh_warehouses.name` diubah dari error (default) → **`severity: warning`**: keputusan owner 2026-11-23 (dikonfirmasi saat eksekusi) — penegakan gudang fail-closed ada di importer WO-06/O-2, validator cukup memberi peringatan agar tidak "menghukum" berkas yang masih bisa diperbaiki.
- `plugins/warehouse/schema/schema.go:70` — `ValidationRule` dapat `Min float64` + `Exclusive bool` (kind `min_value`), plus doc daftar kind diperbarui. Perubahan aditif: YAML lama tetap valid.
- `plugins/warehouse/schema/schema_test.go:215` — `min_value` didaftarkan sebagai kind yang dikenal, sehingga salah tulis kind di YAML langsung gagal di test schema (bukan diam-diam diabaikan).

### B. Validator (`services/validate.go`)

- `plugins/warehouse/services/validate.go:120` — `ValidateTemplate(name, data, opts)` satu pintu: file-level (kosong / `.xls` / tidak terbaca) → `error`; masalah baris → `ValidationReport`. Handler jadi tipis dan perilaku HTTP = perilaku unit test.
- `plugins/warehouse/services/validate.go:187` — `isLegacyXLS` menolak `.xls` lewat **ekstensi DAN magic bytes OLE2** (`xlsMagic`, baris 50): pengguna bisa mengganti nama `.xls` → `.csv`, dan tanpa cek magic file itu lolos sebagai CSV sampah. Pesannya menyuruh simpan ulang sendiri (image tetap Go-only, tanpa konversi server-side).
- `plugins/warehouse/services/validate.go:197` — deteksi format dari **isi** (`PK` = XLSX, sisanya CSV) sehingga endpoint tidak butuh parameter format untuk membaca; `?format=` tetap ada hanya untuk memilih format umpan-balik.
- `plugins/warehouse/services/validate.go:228` — `readTemplateVersion` membaca baris `template_version` di sheet PETUNJUK; CSV mengembalikan `-1` (tidak ada versi) → **tidak** dianggap mismatch. Tanpa aturan ini setiap berkas CSV akan ditolak seluruhnya.
- `plugins/warehouse/services/validate.go:130` — mismatch versi menulis satu `FileErrors` lalu **semua baris** ditandai ditolak (`res.Valid = false` di baris ~139). Versi template salah = salah berkas, bukan salah isi; user harus tahu itu di tingkat file, bukan membaca 200 baris merah.
- `plugins/warehouse/services/validate.go:265` — `trimBlankRows` membuang baris kosong: template WO-04 menyiapkan 200 baris, jadi tanpa ini `total` selalu 200 dan laporan tidak berarti.
- `plugins/warehouse/services/validate.go:289` — `normHeader` hanya untuk PENCARIAN kolom (trim + buang tanda `*`), sementara umpan-balik memakai `in.RawHeader` (baris 123): gotcha hsh/sec — kalau memakai header normalisasi, kolom `Kategori *` berubah jadi `kategori` di berkas hasil dan user bingung mencari kolomnya.
- `plugins/warehouse/services/validate.go:301` — `buildColSpecs` memetakan kolom template ke posisi header; kolom `required` yang hilang → error file-level, kolom opsional (mis. `Satuan`) boleh absen tanpa suara.
- `plugins/warehouse/services/validate.go:334` — `referenceIndex` menolak mendaftarkan sumber kosong (`setOf(...)` hanya dipakai bila `len > 0`): DB baru belum punya kategori/gudang, dan bila sumber kosong didaftarkan maka SEMUA baris tertolak hanya karena katalog kosong (gotcha #1 handoff). Dikunci test `TestReferenceIndexSkipsEmptySources`.
- `plugins/warehouse/services/validate.go:368` — `validateRow` menjalankan YAML rules **lebih dulu** (kata-katanya yang dikurasi), lalu enum & cek tipe sebagai jaring pengaman; cek tipe menangkap hal yang regex tidak bisa (mis. `2026-02-31` lolos pola tanggal tapi bukan tanggal nyata).
- `plugins/warehouse/services/validate.go:371` — helper `add` melaporkan **maksimal satu pesan per kolom** (error menang, tidak ada error = boleh peringatan): tanpa ini satu sel `Jumlah Masuk` yang salah bisa memunculkan dua pesan serupa, dan tabel umpan-balik jadi sulit dibaca.
- `plugins/warehouse/services/validate.go:426` — `isDateType` + `time.Parse(DateLayout, ...)`: validasi tanggal memakai layout kanonik server (`2006-01-02`) sehingga tidak ada format kedua yang diam-diam "hampir benar".
- `plugins/warehouse/services/validate.go:469` — `ruleFlags` menambah kind `min_value` (toleran pemisah ribuan via `parseNumeric`) dan tetap memakai pola lama: sumber referensi tidak dikenal → **jangan** memblokir; kind tak dikenal → diabaikan (forward-compatible).
- `plugins/warehouse/services/validate.go:515` — `severityOf` memetakan `warning` dari YAML; baris dengan hanya warning tetap `valid` dan pesannya diberi awalan `PERINGATAN` di `reasonString` (baris 554) supaya beda dengan alasan penolakan.
- `plugins/warehouse/services/validate.go:592` — `buildFeedback` mengembalikan `GeneratedTemplate` (tipe yang sama dengan WO-04) → tidak ada tipe artefak baru; format default = mengikuti berkas masukan, `?format=` hanya menimpanya.
- `plugins/warehouse/services/validate.go:648` — XLSX umpan-balik: sheet `HASIL` (semua baris asli + 2 kolom verdict, baris ditolak diberi fill merah) + `RINGKASAN` (versi, total/valid/ditolak, kesalahan file, `alasan_terbanyak`) sehingga penyebab dominan terlihat sekali lihat.
- `plugins/warehouse/services/validate.go:629` — CSV umpan-balik = BOM + header asli + 2 kolom tambahan, ditulis lewat `encoding/csv` (quoting otomatis untuk alasan yang memuat koma/kutip).

### C. HTTP (`handlers/validate.go`, `plugin.go`)

- `plugins/warehouse/handlers/validate.go:48` — `ValidateTemplate` hanya mengambil berkas + query, memanggil layanan, lalu menyetel `Content-Disposition` + tiga header `X-Validate-Total/Valid/Rejected`. Tidak ada logika aturan di handler.
- `plugins/warehouse/handlers/validate.go:84` — `readUpload` menerima multipart `file` **atau** body mentah, dibatasi `32 MiB` (`maxValidateUpload`, baris 21): template 200 baris jauh di bawah batas, tapi unggahan raksasa tidak boleh masuk memori tanpa rem.
- `plugins/warehouse/handlers/validate.go:110` — `strictOn` menerima `1/true/yes`; `strict=1` hanya mengubah status menjadi **422 dengan body yang sama** (`c.Data(status, ...)` dipakai untuk kedua jalur) — bukan error berbeda, kontrak handoff.
- `plugins/warehouse/handlers/validate.go:65` — ringkasan run ditulis ke log server (`[wh.validate] ...`) + daftar `top_reasons`: hasil validasi selalu meninggalkan jejak di log, tidak hanya di berkas yang diunduh.
- `plugins/warehouse/plugin.go:110` — route didaftarkan di jalur **flat** `/admin/wh-template/validate` (sama seperti WO-04) supaya tidak masuk ruang wildcard `/wh/*`, guard `warehouse.read` karena validator tidak menulis apa pun.

### D. Test & fixture

- `plugins/warehouse/services/validate_test.go:94` — `TestValidateTemplateAcceptsCleanFile` membuktikan tanpa DB berkas wajar tetap 100% valid (sumber referensi kosong dilewati) dan `top_reasons` kosong.
- `plugins/warehouse/services/validate_test.go:126` — `TestValidateTemplateRejectsBadRows`: 9 baris → 1 valid / 8 ditolak dengan lokasi kolom yang diperiksa satu per satu (kosong wajib, qty 0/negatif, satuan asing, tanggal salah format, tanggal mustahil).
- `plugins/warehouse/services/validate_test.go:185` — `TestValidateTemplateSingleErrorPerColumn` mengunci aturan "satu pesan per kolom" (nilai `dua belas` untuk qty hanya memunculkan 1 pesan).
- `plugins/warehouse/services/validate_test.go:211` — `TestValidateTemplateCategoryWarehouseAreWarnings` (DB nyata) mengunci keputusan owner: kategori/gudang asing → baris tetap `valid` + warning, dan warning TIDAK masuk `top_reasons`. `TestValidateTemplateKnownCategoryIsSilent` (baris 263) memastikan kebalikannya: katalog yang cocok tidak memunculkan pesan apa pun.
- `plugins/warehouse/services/validate_test.go:292` — version mismatch diuji di XLSX (upgrade versi di PETUNJUK): 0 valid, semua baris membawa pesan file-level.
- `plugins/warehouse/services/validate_test.go:354` — `.xls` ditolak baik lewat ekstensi maupun magic bytes (dua sub-test), karena nama berkas bisa dipalsukan tapi magic bytes tidak.
- `plugins/warehouse/services/validate_test.go:383` — jalur CSV: `template_version` = -1 (tanpa mismatch), baris kosong diabaikan, header umpan-balik memakai header ASLI (`Kategori *`), dan dua kolom verdict ditambahkan di akhir.
- `api-tests/fixtures/wh-broken-template.csv` — 9 baris (2 valid / 7 ditolak) memakai header template WO-04 apa adanya; sengaja mencampur kesalahan format (tanggal), referensi (satuan), dan wajib-isi supaya satu kali jalan menguji seluruh jenis aturan.
- `api-tests/fixtures/wh-legacy.xls` — 79 byte, dibuka dengan 8 byte magic OLE2 + teks penanda: fixture kecil untuk membuktikan tolakan `.xls` lewat jalur HTTP nyata (bukan hanya unit test).
- `api-tests/warehouse/04-Validate-template-broken.bru` / `05-Validate-template-strict.bru` / `06-Validate-rejects-legacy-xls.bru` — Bruno: jalur normal (200 + total/valid/ditolak 9/2/7 di header + kolom verdict di body), `strict=1` → 422 dengan berkas tetap terlampir, dan `.xls` → 400. `@file(./fixtures/...)` relatif ke ROOT koleksi `api-tests` (gotcha #3 handoff).

### Bukti

Unit test (container `bbb-data-board-app-1`, DB nyata):

```
go build ./... && go vet ./...                       # OK
go test -count=1 -run "TestValidate|TestRuleFlags|TestReferenceIndex" ./plugins/warehouse/services/
--- PASS: TestValidateTemplateAcceptsCleanFile
--- PASS: TestValidateTemplateRejectsBadRows
--- PASS: TestValidateTemplateSingleErrorPerColumn
--- PASS: TestValidateTemplateCategoryWarehouseAreWarnings   (DB, 0.05s)
--- PASS: TestValidateTemplateKnownCategoryIsSilent          (DB, 0.03s)
--- PASS: TestValidateTemplateVersionMismatch
--- PASS: TestValidateTemplateMissingRequiredColumn
--- PASS: TestValidateTemplateRejectsLegacyXLS (by extension / by magic bytes)
--- PASS: TestValidateTemplateEmptyUpload
--- PASS: TestValidateTemplateCSV
--- PASS: TestValidateFeedbackFormatOverride
--- PASS: TestValidateFeedbackRejectsUnknownFormat
--- PASS: TestRuleFlagsMinValue
--- PASS: TestReferenceIndexSkipsEmptySources
ok  github.com/rolldone/donisgo/plugins/warehouse/services  0.149s
go test ./...                                        # 0 FAIL
gofmt -l plugins/warehouse/                          # (kosong)
```

Curl (server container, port 8082):

| Kasus | Hasil |
|---|---|
| XLSX terisi 5 baris (2 valid, 3 ditolak) | **200**, feedback `wh_filled_hasil.xlsx`, sheet `HASIL` + `RINGKASAN`; header `X-Validate-Total: 5 / Valid: 2 / Rejected: 3` |
| Versi 99 pada XLSX terisi | **200**, `5/0/5` — semua baris `ditolak` dengan alasan `template_version: 99 ditemukan, versi yang didukung: 1` |
| CSV `wh-broken-template.csv` | **200**, `text/csv`, `wh-broken-template_hasil.csv`, header `9/2/7`, body memuat tiap baris + `hasil_validasi`/`alasan_ditolak` |
| `?strict=1` berkas rusak | **422** (body = berkas umpan-balik yang sama) |
| `.xls` (magic OLE2) | **400** `berkas .xls (Excel lama) tidak didukung; ... simpan sebagai .xlsx atau .csv` |
| berkas kosong | **400** `berkas kosong` |
| tanpa token | **401** |
| viewer (`warehouse.read`) | **200** `9/2/7`; GET `/wh/items` 200; POST `/wh/categories` 403 |

Aturan yang terbukti dari body umpan-balik CSV:

```
Kebutuhan Bayi,,,, → "Produk: wajib diisi"
,Produk Tanpa Kategori,, → "Kategori: wajib diisi"
...,Gudang kosong → "Gudang: wajib diisi"
...,0,... → "Jumlah Masuk: harus angka lebih dari 0 (boleh pemisah ribuan)"
...,karung,... → "Satuan: tidak ada di daftar satuan REFERENSI (lihat sheet REFERENSI)"
...,23-11-2026,... → "Tanggal Masuk: harus berformat YYYY-MM-DD (mis. 2026-11-23)"
...,2026-02-31,... → "Tanggal Masuk: "2026-02-31" bukan tanggal yang valid (YYYY-MM-DD, mis. 2026-11-23)"
```

Warning-only dengan katalog terisi (1 kategori + 1 gudang terdaftar):

```
Kategori Terdaftar,Produk A,Gudang Terdaftar,5,pcs,2026-11-23,,katalog dikenal,valid,
Kategori Belum Ada,Produk B,Gudang Belum Ada,5,pcs,2026-11-23,,katalog asing,valid,PERINGATAN Kategori: kategori belum terdaftar — akan dibuat otomatis saat impor (O-1); PERINGATAN Gudang: gudang belum terdaftar (daftarkan lewat menu Gudang sebelum impor)
→ header 2/2/0 (tidak ada baris yang ditolak karena katalog)
```

Bruno (`cd api-tests && bru run auth/00-Login.bru warehouse -r --env Local`):

```
warehouse/04-Validate-template-broken (200 OK)      4/4 test ✓
warehouse/05-Validate-template-strict (422)         2/2 test ✓
warehouse/06-Validate-rejects-legacy-xls (400)      2/2 test ✓
Requests 7 (7 Passed) · Tests 18/18 · Status PASS
```

Read-only terbukti: `cats/whs/items/batches/movements` **0/0/0/0/0 sebelum dan sesudah** seluruh matriks validate (termasuk warning-test), dan tidak ada satu pun endpoint validate yang menulis tabel. Data uji dinolkan lagi setelah verifikasi (kategori/gudang uji + admin uji `whviewer@test.local` dihapus → `testadmins=0`). Tanpa `push`; `AdminLayout.tsx`/`index.css` tidak disentuh.

Catatan selisih dokumen: kontrak handoff menyebut tanggal `DD/MM/YYYY` (pola hsh/sec), tetapi `fields.yaml` (blok `validation:`) dan template WO-04 yang sudah LULUS memakai `YYYY-MM-DD`; dikonfirmasi ke Manager saat eksekusi → **YYYY-MM-DD** dipertahankan (satu format saja, layout server `2006-01-02`).


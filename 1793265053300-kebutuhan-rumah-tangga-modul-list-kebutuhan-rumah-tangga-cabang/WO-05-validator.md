---
title: "WO-05: Validator umpan-balik template"
description: "Endpoint validasi template hsh (read-only) yang mengembalikan error per baris tanpa menyentuh DB."
status: complete
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - template
  - validation
---

# WO-05: Validator umpan-balik template

## Referensi
- SPEC: N-14, N-16, N-17, §Kontrak Template, §Alur Impor.
- Pola acuan: `plugins/resident/services/validate.go` + `handlers/validate.go`
  (multipart `file`, `?format=`, `?strict=1`→422, header ringkasan `X-Validate-*`, aturan dari `fields.yaml`).

## Tugas
1. **`services/validate.go`** — aturan diturunkan dari `fields.yaml` (required/enum/type + blok
   `validation:` baru bila perlu):
   - `Periode`: wajib; format `YYYY-MM` (atau `YYYY-MM-DD`).
   - `Kategori`: wajib; **harus ada** di daftar REFERENSI (kategori **tidak** auto-create).
   - `Nama Kebutuhan`: wajib; non-kosong setelah trim.
   - `Satuan`: opsional; bila diisi harus ada di daftar REFERENSI.
   - `Status`: wajib; map label Indonesia → code (`URGENT/ENOUGH/UNKNOWN`).
   - `Harga Satuan`/`Jumlah`: opsional; bila diisi harus angka ≥ 0.
   - **`template_version`** dicek; versi tak dikenal → file-level reject.
   - **Deteksi `.xls`** (BIFF8): tolak dengan pesan **pra-konversi** (N-17).
2. **`handlers/validate.go`** — `POST /admin/hsh-template/validate`:
   - multipart `file`; `?format=xlsx|csv`; `?strict=1` → status 422 bila ada penolakan.
   - **read-only**: tidak menulis DB apa pun.
   - Ringkasan → stdout + header `X-Validate-Total/Valid/Rejected`.
   - Feedback memakai **header asli** dari berkas (pola `RawHeader`).
3. **Test**: berkas valid, berkas rusak (fixture) dengan 7 kategori error berbeda, `.xls` ditolak,
   versi usang ditolak, non-strict 200 vs strict 422.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/hsh/...` hijau.
- curl: template valid → total=N valid=M rejected=0; fixture rusak → rejected>0 dengan rincian baris+kolom.
- Uji `.xls` → 400/422 dengan pesan konversi; `?strict=1` → 422.
- Bukti tidak ada penulisan DB (bandingkan `SELECT count(*)` sebelum/sesudah).

## Kriteria Selesai
- 100% aturan dari `fields.yaml` (menambah aturan = edit yaml, bukan kode).
- Validator read-only; pesan error jelas & berbahasa Indonesia.

## Bukti Pelaksanaan
- **Commit kode:** `7c85b71` — `feat(hsh): validator umpan-balik template read-only (WO-05)`.
  File: `plugins/hsh/services/validate.go`, `plugins/hsh/services/validate_test.go`,
  `plugins/hsh/services/validate_db_test.go`, `plugins/hsh/handlers/validate.go`,
  `plugins/hsh/schema/{fields.yaml,schema.go,schema_test.go}`, `plugins/hsh/plugin.go`,
  `api-tests/fixtures/hsh-broken-template.csv` (fixture rusak, 8 baris).
- **Aturan 100% dari `fields.yaml`** — blok `validation:` baru:
  - `hsh_periods.code` → `regex '^\d{4}-(0[1-9]|1[0-2])(-\d{2})?$'`.
  - `hsh_items.unit` → `in_reference: [units]` (daftar `units` di yaml).
  - `hsh_categories.code` → `in_reference: [categories]` (tabel `hsh_categories` aktif; kategori **tidak** auto-create).
  - `required`/`enum`/`type` datang dari `tables`; `numeric*` otomatis dicek angka ≥ 0.
  - `schema_test.go` menambah `TestValidationReferencesResolve` (field harus ada, jenis rule dikenal, message wajib).
- **Kualitas:** `gofmt -l` bersih, `go build ./...` hijau, `go vet ./plugins/hsh/...` bersih,
  `go test ./plugins/hsh/...` hijau. 11 test validator RUN (tidak skip), termasuk
  `TestValidateTemplateCategoryFromDB` dan `TestValidateTemplateRejections`.
- **curl (server :8082, `POST /api/admin/hsh-template/validate`):**

  | Berkas | Hasil |
  |---|---|
  | CSV valid (2 baris, status `Stok cukup` = label ID) | **200**, `X-Validate-Total: 2`, `Valid: 2`, `Rejected: 0`; feedback `hsh-valid_hasil.csv` + dua kolom verdict |
  | `api-tests/fixtures/hsh-broken-template.csv` | **200** (non-strict), `Total: 8`, `Valid: 1`, `Rejected: 7` — 7 alasan berbeda (lihat bawah) |
  | fixture sama + `?strict=1` | **422** (artefak tetap dikirim) |
  | `.xls` legacy (magic `D0 CF 11 E0 …`) | **400** `berkas .xls (Excel lama) tidak didukung; buka di Excel/LibreOffice lalu simpan sebagai .xlsx atau .csv` |
  | XLSX terisi 2 baris (feedback default = xlsx) | **200**, `Total: 2`, `Valid: 1`, `Rejected: 1`, `hsh-filled_hasil.xlsx` (sheet `HASIL` + `RINGKASAN`) |
  | tanpa token | **401** |

- **7 alasan berbeda pada fixture rusak:** `Periode: harus berformat YYYY-MM…`; `Nama Kebutuhan: wajib diisi`;
  `Satuan: tidak ada di daftar satuan REFERENSI…`; `Status: "BANYAK" tidak dikenal (pilihan: URGENT, ENOUGH, UNKNOWN)`;
  `Harga Satuan: "-5" harus angka >= 0…`; `Jumlah: "dua" harus angka >= 0…`;
  `Kategori: kategori tidak dikenal (pakai kode dari sheet REFERENSI)`.
- **Read-only terbukti:** `hsh_entries/hsh_items/hsh_periods/hsh_categories` = `0/0/0/7`
  **sebelum dan sesudah** seluruh validasi; kategori `kategori_hantu` tidak tercipta
  (diuji juga di `TestValidateTemplateCategoryFromDB`).
- **Detail implementasi:**
  - Deteksi `.xls` dari ekstensi **atau** magic OLE2 (N-17), tanpa konversi sisi server.
  - `template_version` dibaca dari sheet PETUNJUK; versi tak cocok → file-level error,
    semua baris ditolak (tetap 200/422, bukan 500).
  - Label status Indonesia diterima dan dipetakan ke kode (`Urgent / Stok kosong` → `URGENT`).
  - Angka berformat Excel (`95,000.00`) diterima; pemisah ribuan dinormalkan.
  - Baris kosong (200 baris pra-format template) diabaikan dari `total`.
  - Umpan balik mempertahankan **header asli** berkas + `hasil_validasi`/`alasan_ditolak`;
    XLSX menambah sheet `RINGKASAN` (versi, total, valid, ditolak, alasan terbanyak);
    baris ditolak diberi latar `F8CBAD`.
  - Ringkasan juga dicetak ke stdout server (`[hsh.validate] …`).
  - Route: `POST /api/admin/hsh-template/validate` (flat, guard `hsh.read`) — read-only.
- **Dampak di luar plugin:** hanya fixture baru di `api-tests/fixtures/`; tidak menyentuh `internal/`.
  Fixture ini akan dipakai koleksi Bruno di WO-09.

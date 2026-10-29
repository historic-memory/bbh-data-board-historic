---
title: "WO-05: Validator umpan-balik"
description: "Endpoint POST /admin/sec-template/validate read-only: memeriksa berkas unggahan per baris dan mengembalikan umpan-balik."
status: pending
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
_(diisi saat WO dikerjakan)_

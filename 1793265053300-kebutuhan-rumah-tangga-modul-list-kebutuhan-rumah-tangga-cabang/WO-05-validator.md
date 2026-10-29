---
title: "WO-05: Validator umpan-balik template"
description: "Endpoint validasi template hsh (read-only) yang mengembalikan error per baris tanpa menyentuh DB."
status: pending
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

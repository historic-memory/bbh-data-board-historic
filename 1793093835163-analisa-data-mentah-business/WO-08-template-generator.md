---
title: "WO-08: Generator Template Impor (XLSX + CSV) via HTTP API"
description: "Endpoint `GET /api/admin/resident-template` yang menghasilkan template impor XLSX (sheet PETUNJUK/DATA/REFERENSI/CONTOH, dropdown enum, format tanggal, template_version) dan varian CSV, seluruhnya di-generate dari `schema/fields.yaml` agar tidak drift dengan migrasi & validator."
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 2
---

# WO-08: Generator Template Impor

**SPEC acuan:** `SPEC.md` §15 (Template Impor), P-13, P-14, D-14..D-18
**Prasyarat:** WO-02 (butuh `schema/fields.yaml`)

## Tujuan
Menghasilkan **template XLSX/CSV** yang selalu konsisten dengan skema canonical, sehingga cabang bisa **mengisi data sesuai standar** tanpa transformasi.

## Keputusan (2026-10-27)
| # | Keputusan | Nilai |
|---|---|---|
| T-1 | **Bentuk** | **HTTP API** (bukan CLI): `GET /api/admin/resident-template`. Owner memilih API konsisten dengan WO-05. |
| T-2 | **Library XLSX** | `github.com/xuri/excelize/v2`. |
| T-3 | **Sumber enum** | **`schema/fields.yaml`** (D-3/D-12: tidak ada tabel `ref_*`). Bagian WO lama yang menyebut `ref_*` **USANG**. |
| T-4 | **Path route** | `/api/admin/resident-template` (flat) agar tidak bentrok dengan wildcard `/api/admin/residents/:id` di router gin. |

## Cakupan

Endpoint:
```
GET /api/admin/resident-template[?format=xlsx|csv][&no_pii=1]
```
- `format` default `xlsx`; `csv` mengembalikan CSV satu file (header + BOM).
- `no_pii=1` menghilangkan kolom PII (D-18).
- Guard permission: `resident.read`. Respons: file unduhan (Content-Disposition attachment).

### 1. Sumber kebenaran
- `schema/fields.yaml`: daftar kolom sheet `DATA` (`template.columns`), label, field wajib, dan **enum** (dropdown). **Tidak ada daftar kolom hard-code** di generator (AC-10).

### 2. Struktur XLSX
| Sheet | Isi |
|---|---|
| `PETUNJUK` | Aturan isi, field wajib, format tanggal `DD/MM/YYYY`, aturan kode RM `BBH-<YY>-<P\|L>-<NNN>`, kontak, **`template_version`** |
| `DATA` | Header + 200 baris kosong siap isi (kolom = `template.columns`) |
| `REFERENSI` | Daftar nilai enum (sumber dropdown); sheet disembunyikan |
| `CONTOH` | 3 baris data fiktif terisi lengkap |

Kolom `DATA` (23, urutan sesuai `fields.yaml`): `resident_code, full_name, gender, birth_date, birth_place, religion_code, blood_type_code, nik, bpjs_no, mothers.full_name, mothers.nik, mothers.phone, birth_weight_gram, birth_length_cm, child_order, delivery_method_code, attendant_type_code, facility_id, attendant_detail, admitted_at, status_code, alias, documents.url` (header Indonesia).

### 3. Kontrol kualitas di dalam file
- **Data Validation (dropdown)** untuk 6 kolom enum: `gender`, `religion`, `blood_type`, `delivery_method`, `attendant_type`, `status` → bersumber `REFERENSI`.
- Format sel: `birth_date`/`admitted_at` = tanggal `DD/MM/YYYY`; `birth_weight_gram`/`birth_length_cm`/`child_order` = numerik.
- Header field wajib diberi penanda `*`.
- **`template_version`** ditulis di sheet `PETUNJUK`.

### 4. Varian CSV
- Mengembalikan `template_resident_v<N>.csv` (BOM + header = kolom `DATA`).
- CSV **tidak** mendukung dropdown — catatan keterbatasan ada di `PETUNJUK`/README.

### 5. Opsi `no_pii`
- Menghasilkan template **tanpa** kolom PII (`residents.nik`, `residents.bpjs_no`, `mothers.nik`, `mothers.phone`) untuk versi pihak luar (D-18). 23 → 19 kolom.

## Non-Cakupan
- Validator & umpan-balik error → WO-09.
- Impor file terisi → WO-06 (ditunda).

## Bukti yang Harus Dilaporkan
- `GET /api/admin/resident-template` → file XLSX (Content-Type/Disposition benar).
- XLSX: 4 sheet, `DATA` = 23 kolom, 6 dropdown, `template_version`.
- Bukti **tidak ada hard-code kolom**: ubah satu label di `fields.yaml` → regenerasi → header ikut berubah.
- `no_pii=1` → kolom PII hilang.
- `go build`/`vet`/`test`/`gofmt` lulus.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | Endpoint menghasilkan XLSX & CSV. |
| AC-2 | XLSX punya 4 sheet: `PETUNJUK`, `DATA`, `REFERENSI`, `CONTOH`. |
| AC-3 | Kolom `DATA` = 23 kolom, urutan benar. |
| AC-4 | 6 kolom enum punya **dropdown** (nilai dari `fields.yaml`). |
| AC-5 | Header field wajib bertanda `*`. |
| AC-6 | `template_version` tercatat di file. |
| AC-7 | Kolom & label **di-generate** dari `schema/fields.yaml`. |
| AC-8 | `no_pii` menghasilkan template tanpa kolom PII. |
| AC-9 | Sheet `CONTOH` berisi 3 baris data fiktif valid. |
| AC-10 | Tidak ada daftar kolom hard-code di generator. |
| AC-11 | `go build`/`vet`/`test`/`gofmt` lulus. |

## Status Eksekusi (complete)
**Status: complete.** Tidak ada push.

### Bukti
- **Endpoint:** `GET /api/admin/resident-template` -> `200`, `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `Content-Disposition: attachment; filename="template_resident_v1.xlsx"` (12943 byte, `file` = "Microsoft Excel 2007+").
- **Struktur XLSX** (dibedah dari arsip): sheet `PETUNJUK`, `REFERENSI`, `DATA`, `CONTOH`; **6 dataValidation** di sheet DATA (formula `REFERENSI!$X$2:$X$n`); `REFERENSI` `state="hidden"`; `template_version` ada.
- **CSV:** `?format=csv` -> header (BOM) `Kode RM,Nama Bayi *,Jenis Kelamin *,...`.
- **no_pii:** `?no_pii=1` -> 12825 byte (lebih kecil); unit test memastikan 19 kolom (23 - 4 PII).
- **AC-7 (bukan hard-code):** label `Nama Bayi` -> `Nama Bayi UJI` di `fields.yaml` -> rebuild -> CSV header jadi `Nama Bayi UJI *`; di-revert kembali.
- **Error handling:** `?format=pdf` -> `400`; tanpa token -> `401`.
- **Kualitas:** `go build ./... && go vet ./... && gofmt -l plugins/resident && go test ./plugins/resident/... -count=1` -> **lulus** di container `app` (4 test generator: xlsx/nopii/csv/bad-format).
- **Bruno:** `bru run --env Local smoke templates` -> **PASS** (6/6 request, 13/13 test).

| # | AC | Hasil |
|---|---|---|
| AC-1 | XLSX & CSV | OK |
| AC-2 | 4 sheet | OK |
| AC-3 | 23 kolom DATA | OK `len(rows[0])==len(schema.Template.Columns)` |
| AC-4 | 6 dropdown | OK 6 dataValidation |
| AC-5 | Header wajib `*` | OK |
| AC-6 | template_version | OK PETUNJUK B3=1 |
| AC-7 | Di-generate dari fields.yaml | OK label-change proof |
| AC-8 | no_pii | OK 19 kolom |
| AC-9 | CONTOH 3 baris | OK header+3 |
| AC-10 | Nol hard-code kolom | OK |
| AC-11 | build/vet/test/gofmt | OK |

### Catatan
- Menambah `excelize` meng-upgrade beberapa `golang.org/x/*` (MVS) — dicatat sebagai konsekuensi dependensi.
- Impor file terisi (WO-06) **ditunda** (lihat WO-06).

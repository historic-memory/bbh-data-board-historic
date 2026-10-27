---
title: "WO-08: Generator Template Impor (XLSX + CSV)"
description: "Perintah `resident template generate` yang menghasilkan template XLSX (sheet PETUNJUK/DATA/REFERENSI/CONTOH, dropdown enum, format tanggal, template_version) dan varian CSV, seluruhnya di-generate dari `schema/fields.yaml` agar tidak drift dengan migrasi & validator."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-08: Generator Template Impor

**SPEC acuan:** `SPEC.md` §15 (Template Impor), P-13, P-14, D-14..D-18
**Prasyarat:** WO-02 (butuh `schema/fields.yaml` + seed referensi)

## Tujuan
Menghasilkan **template XLSX/CSV** yang selalu konsisten dengan skema canonical, sehingga cabang bisa **mengisi data sesuai standar** tanpa transformasi.

## Cakupan

Perintah:
```
resident template generate [--out DIR] [--format xlsx|csv|both] [--version N] [--include-pii|--no-pii]
```

Default: `--out ./dist/template`, `--format both`, versi dari konstanta kode, `--include-pii` (internal).

### 1. Sumber kebenaran
- Baca `plugins/resident/schema/fields.yaml` + tabel `ref_*` (dari DB) sebagai sumber:
  - daftar kolom sheet `DATA`
  - label kolom
  - field wajib (`*` di header)
  - nilai dropdown per enum
- **Tidak boleh** ada daftar kolom hard-code di kode generator.

### 2. Struktur XLSX
| Sheet | Isi |
|---|---|
| `PETUNJUK` | Aturan isi, field wajib, format tanggal `DD/MM/YYYY`, aturan kode RM `BBH-<YY>-<P\|L>-<NNN>`, kontak bantuan, **`template_version`** |
| `DATA` | Header + 200 baris kosong siap isi; kolom sesuai §15.3 SPEC |
| `REFERENSI` | Daftar nilai enum (dari `ref_*`), dipakai sebagai sumber dropdown; sheet disembunyikan |
| `CONTOH` | 3 baris data fiktif terisi lengkap |

Kolom `DATA` (urutan sesuai §15.3):
`resident_code, nama_bayi, jenis_kelamin, tanggal_lahir, tempat_lahir, agama, gol_darah, nik_bayi, bpjs_no, nama_ibu, nik_ibu, no_hp_ibu, berat_lahir_gram, panjang_lahir_cm, anak_ke, persalinan, ditolong_oleh, fasilitas, detail_tenaga_medis, tanggal_masuk, status, keterangan, folder_drive`

### 3. Kontrol kualitas di dalam file
- **Data Validation (dropdown)** untuk `jenis_kelamin`, `agama`, `gol_darah`, `persalinan`, `ditolong_oleh`, `status` → bersumber `REFERENSI`.
- Format sel: `tanggal_lahir`/`tanggal_masuk` = date cell `DD/MM/YYYY`; `berat_lahir_gram`/`panjang_lahir_cm`/`anak_ke` = numeric.
- Header field wajib diberi penanda `*`.
- **`template_version`** ditulis di `PETUNJUK` (dan/atau nama file).

### 4. Varian CSV
- Menghasilkan `template.csv` (header = kolom `DATA`) + `template_referensi.json`.
- CSV **tidak** mendukung dropdown — catat keterbatasan ini di `PETUNJUK`/README.

### 5. Opsi `--no-pii`
- Menghasilkan template **tanpa** kolom PII (`nik_bayi`, `nik_ibu`, `bpjs_no`, `no_hp_ibu`) untuk versi yang dibagikan ke pihak luar (D-18).

## Non-Cakupan
- Validator & umpan balik error → WO-09.
- Impor file terisi → WO-06.

## Bukti yang Harus Dilaporkan
- `resident template generate --format both` → daftar file yang dihasilkan + ukuran.
- XLSX dibuka & diverifikasi: jumlah sheet 4, jumlah kolom `DATA` = 23, dropdown ada pada 6 kolom.
- Screenshot/teks isi sheet `PETUNJUK` (memuat `template_version`).
- Bukti **tidak ada hard-code kolom**: ubah satu label di `fields.yaml` → regenerasi → header XLSX ikut berubah.
- Varian `--no-pii` → kolom PII hilang (daftar kolom dicetak).
- `go build`/`vet`/`test`/`gofmt` lulus.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | `resident template generate` menghasilkan XLSX & CSV. |
| AC-2 | XLSX punya 4 sheet: `PETUNJUK`, `DATA`, `REFERENSI`, `CONTOH`. |
| AC-3 | Kolom `DATA` = 23 kolom sesuai §15.3 SPEC, urutan benar. |
| AC-4 | 6 kolom enum punya **dropdown** yang nilainya berasal dari `ref_*`. |
| AC-5 | Header field wajib bertanda `*`. |
| AC-6 | `template_version` tercatat di file. |
| AC-7 | Kolom & label **di-generate** dari `schema/fields.yaml` (dibuktikan dengan mengubah satu label). |
| AC-8 | `--no-pii` menghasilkan template tanpa kolom PII. |
| AC-9 | Sheet `CONTOH` berisi 3 baris data fiktif valid. |
| AC-10 | Tidak ada daftar kolom hard-code di generator. |
| AC-11 | `go build`/`vet`/`test`/`gofmt` lulus. |

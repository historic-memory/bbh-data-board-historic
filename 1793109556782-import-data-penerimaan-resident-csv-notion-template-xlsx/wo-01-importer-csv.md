---
title: "WO-01: Importer CSV/Template → Canonical + laporan validasi"
description: "Perintah impor yang membaca CSV mentah (format Notion) atau template BBH (XLSX/CSV hasil WO-08) dan mengubahnya ke skema canonical sesuai §6 SPEC, lengkap dengan dry-run, staging + idempotensi, dan laporan baris valid/ditolak. Bukti pertama diuji pada Data Bayi Yayasan (50 baris)."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-01: Importer CSV/Template → Canonical

> **Berasal dari:** topic `1793093835163-analisa-data-mentah-business` (dulu **WO-06**).
> Dipindahkan ke topic ini pada 2026-10-27 karena lingkupnya berdiri sendiri
> (importer) dan sempat ditunda; pengerjaan dilanjutkan di sini.

**SPEC acuan:** `SPEC.md` §6 (Pemetaan kolom sumber, normalisasi, klasifikasi
`Keterangan`), §6.2 (varian format) — di topic `1793093835163-analisa-data-mentah-business`.
**Prasyarat:** WO-04 (API CRUD), WO-08 (generator template), WO-09 (validator
umpan-balik) — semuanya **complete** di topic resident.
**Data uji:** `.historic/1793093835163-analisa-data-mentah-business/Data Bayi Yayasan 1b590f1a8a518093ac5bd1d366590b33_all.csv`

## Tujuan
Membaca CSV ekspor Notion **atau** template BBH dan memetakannya ke tabel
canonical **dengan normalisasi eksplisit**, disertai **laporan** dan mode
`--dry-run` (tidak menulis DB).

## Keputusan Awal & Status (2026-10-27)

> **STATUS: DITUNDA (deferred)** saat di topic resident; **dilanjutkan di topic ini.**
> Keputusan di bawah **dikunci**.

| # | Keputusan | Nilai |
|---|---|---|
| I-1 | **Bentuk importer** | **HTTP API saja** (bukan CLI). Endpoint admin, multipart upload + JSON hasil laporan. |
| I-2 | **Alur** | **Staging/temp table dulu**: baris mentah ditampung, baru dilakukan **pengecekan idempotensi** & promosi ke tabel canonical. |
| I-3 | **Kunci idempotensi** | **`resident_code` bila ada; jika kosong → `nik`** (fallback). |
| I-4 | **Cakupan format** | **Dua mode**: `raw` (CSV Notion) & `template` (XLSX/CSV hasil WO-08). Sejak WO-08 selesai, mode template **tidak lagi blocked**. |
| I-5 | **Vertikal terkait** | Validasi **tidak** digandakan: gunakan hasil/aturan WO-09 (`resident-template/validate`); importer fokus pada **staging + idempotensi + promosi**. |
| I-6 | **Kode kosong** | Bila kolom `resident_code` kosong: **tidak** auto-generate secara default (disimpan `NULL`), **tercatat di laporan**. Flag `--generate-missing` mengaktifkan generator (WO-10). (Menyelesaikan paruh importer dari AC-8 WO-10.) |

## Cakupan

Endpoint (bentuk final diputuskan saat eksekusi):
```
POST /api/admin/resident-import          (multipart file + ?format=raw|template|auto)
  ?branch=KEY            # default INSTANCE_KEY
  ?dry_run=1             # tidak menulis DB, hanya laporan
  ?on_error=skip|abort   # default skip
  ?generate_missing=1    # aktifkan generator resident_code (WO-10)
```
Respons: JSON laporan (ringkasan + baris ditolak) + opsi unduh `report.json`/`report.md`.

### Dua mode format (P-14)
| Mode | Sumber | Perlakuan |
|---|---|---|
| `raw` (default untuk `csv` Notion) | Ekspor Notion / data mentah | Normalisasi penuh (§6.1/§6.2) |
| `template` (default untuk `xlsx`) | Template BBH hasil WO-08 | Sudah canonical: validasi enum/wajib/tanggal **tanpa transformasi berat**; **wajib cek `template_version`** (aturan dari WO-09) |

### Normalisasi wajib (implementasi §6.1/§6.2 SPEC)
| Sumber | Transformasi |
|---|---|
| `\ufeffNama Bayi` | strip BOM + `TrimSpace` |
| `Berat "2600 gram"` | regex `(\d+)` → INT |
| `Panjang "48 cm"` / `"-"` | regex `(\d+)` → INT; `"-"`/kosong → NULL |
| `Tanggal Lahir "19 November 2021"` | parse format teks Indonesia → DATE |
| `Tanggal Masuk "28/02/2022"` | parse DD/MM/YYYY → DATE |
| `NOMOR RM "... (https://...)"` | regex `BBH-\d+-[PL]-\d+`, buang URL |
| `Jenis Kelamin "Perempuan"/"Laki-laki"` | → `P` / `L` |
| `Agama`, `Golongan Darah`, `Persalinan`, `Ditolong Oleh`, `Status di yayasan` | uppercase → cocokkan enum statis (D-3); tak cocok → catat di laporan |
| `NIK Bayi/Ibu`, `Nomor BPJS` | validasi digit; invalid → tolak baris & lapor |
| `Keterangan` | `ClassifyFacility` → `facility` (+ alias) & `attendant_detail` (D-2/D-5) |
| `Files & media`, `Folder Drive` | → `resident_documents` |
| `Place` | **dibuang** (catat sebagai skipped) |
| `Umur` | **dibuang** (derived) |

### Laporan (`report.json` / `report.md`)
- Total baris, baris **valid**, baris **ditolak** (dengan nomor baris + alasan).
- Daftar nilai enum yang **tidak dikenal** per kolom.
- Daftar fasilitas baru/alias baru yang terbuat.
- Daftar baris yang `resident_code` kosong/diduplikasi.
- Statistik field kosong per kolom.

## Aturan
- **`dry_run=1` wajib tidak menulis apa pun ke DB.**
- Impor idempoten: menjalankan ulang pada sumber sama tidak membuat duplikat
  (kunci: `resident_code`, fallback `nik`).
- Baris ditolak **tidak menghentikan** impor (`on_error=skip` default); `abort`
  untuk berhenti di error pertama.
- Semua impor dibungkus transaksi per-baris atau per-batch; kegagalan satu baris
  tidak merusak baris lain (kecuali `abort`).
- **Tidak ada DDL di kode Go.** Staging table dibuat lewat migrasi versioned
  (`plugins/resident/migrations/postgres/`).

## Bukti yang Harus Dilaporkan
- `POST /api/admin/resident-import?dry_run=1` → ringkas laporan (valid/ditolak +
  daftar enum tak dikenal).
- Impor nyata ke DB uji → `SELECT count(*) FROM residents` = jumlah baris valid.
- Perbandingan: 50 baris sumber → jumlah masuk + ditolak + alasan.
- Membuktikan **50 = valid + ditolak** (akuntansi baris lengkap).
- Impor ulang → jumlah tidak bertambah (idempoten).
- `Place` & `Umur` terbukti dibuang.
- `resident_code` kosong → `NULL` + tercatat di laporan.
- `grep -rn "CREATE TABLE" --include=*.go plugins/resident/` → kosong.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | Importer membaca CSV (dengan BOM) tanpa error parsing. |
| AC-2 | Normalisasi §6.1 diimplementasi & diuji untuk **setiap** varian di §6.2. |
| AC-3 | `dry_run=1` **tidak** menulis DB (dibuktikan dengan count sebelum/sesudah). |
| AC-4 | Laporan memuat baris rejected + alasan; `valid + rejected = total`. |
| AC-5 | Enum tak dikenal **dilaporkan**, tidak diam-diam dibuang. |
| AC-6 | `Keterangan` terklasifikasi ke `facilities`/`attendant_detail` (minimal 1 contoh terbukti). |
| AC-7 | `resident_code` diekstrak dari teks+URL; URL tidak tersimpan. |
| AC-8 | Impor **idempoten** (kunci `resident_code` fallback `nik`). |
| AC-9 | Tidak ada DDL di kode Go. |
| AC-10 | `go build`/`vet`/`test`/`gofmt` lulus; ada unit test untuk normalizer. |
| AC-11 | Mode `template` membaca XLSX/CSV hasil WO-08, memvalidasi `template_version`, dan **tidak** menerapkan transformasi raw. |
| AC-12 | `template_version` tak dikenal → impor ditolak dengan pesan jelas. |
| AC-13 | Staging + idempotensi: re-run tidak menduplikasi; promosi hanya baris valid. |
| AC-14 | `resident_code` kosong → `NULL` & tercatat di laporan; `--generate-missing` memakai generator WO-10. |


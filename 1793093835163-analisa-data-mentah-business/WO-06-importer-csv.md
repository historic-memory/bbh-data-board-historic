---
title: "WO-06: Importer CSV Notion → canonical + laporan validasi"
description: "Perintah impor yang membaca CSV mentah (format Notion) dan mengubahnya ke skema canonical sesuai aturan pemetaan §6 SPEC, lengkap dengan mode dry-run, laporan baris valid/ditolak, dan klasifikasi Keterangan → facility. Bukti pertama diuji pada Data Bayi Yayasan (50 baris)."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-06: Importer CSV Notion → Canonical

**SPEC acuan:** `SPEC.md` §6 (Pemetaan kolom sumber, normalisasi, klasifikasi `Keterangan`), §6.2 (varian format)
**Prasyarat:** WO-04
**Data uji:** `.historic/1793093835163-analisa-data-mentah-business/Data Bayi Yayasan 1b590f1a8a518093ac5bd1d366590b33_all.csv`

## Tujuan
Membaca CSV ekspor Notion dan memetakannya ke tabel canonical **dengan normalisasi eksplisit**, disertai **laporan** dan mode `--dry-run` (tidak menulis DB).

## Cakupan

Perintah baru (bisa sebagai subcommand CLI atau service terpisah — **putuskan di awal & catat**):
```
resident import csv <path> [--branch KEY] [--dry-run] [--report PATH] [--on-error skip|abort]
```

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
| `Agama`, `Golongan Darah`, `Persalinan`, `Ditolong Oleh`, `Status di yayasan` | uppercase → cocokkan `ref_*`; tak cocok → catat di laporan (jangan diam) |
| `NIK Bayi/Ibu`, `Nomor BPJS` | validasi digit; invalid → tolak baris & lapor |
| `Keterangan` | `ClassifyFacility` → `facility` (+ alias) & `attendant_detail` |
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
- **`--dry-run` wajib tidak menulis apa pun ke DB.**
- Impor idempoten: menjalankan ulang pada sumber sama tidak membuat duplikat (kunci natural: `resident_code`, atau `nik`, atau kombinasi — **putuskan & catat**).
- Baris ditolak **tidak menghentikan** impor (`--on-error skip` default); `abort` untuk berhenti di error pertama.
- Semua impor dibungkus transaksi per-baris atau per-batch; kegagalan satu baris tidak merusak baris lain (kecuali `abort`).

## Bukti yang Harus Dilaporkan
- `resident import csv <file> --dry-run --report /tmp/report.json` → ringkas isi laporan (jumlah valid/ditolak + daftar enum tak dikenal).
- Impor nyata ke DB uji → `SELECT count(*) FROM residents` = jumlah baris valid.
- Perbandingan: 50 baris sumber → jumlah masuk + jumlah ditolak + alasan.
- Menuunjukkan **50 = valid + ditolak** (akuntansi baris lengkap).
- Impor ulang → jumlah tidak bertambah (idempoten).
- `Place` & `Umur` terbukti dibuang.
- `grep -rn "CREATE TABLE" --include=*.go plugins/resident/` → kosong.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | Importer membaca CSV (dengan BOM) tanpa error parsing. |
| AC-2 | Normalisasi §6.1 diimplementasi & diuji untuk **setiap** varian di §6.2. |
| AC-3 | `--dry-run` **tidak** menulis DB (dibuktikan dengan count sebelum/sesudah). |
| AC-4 | Laporan memuat baris rejected + alasan; `valid + rejected = total`. |
| AC-5 | Enum tak dikenal **dilaporkan**, tidak diam-diam dibuang. |
| AC-6 | `Keterangan` terklasifikasi ke `facilities`/`attendant_detail` (minimal 1 contoh terbukti). |
| AC-7 | `resident_code` diekstrak dari teks+URL; URL tidak tersimpan. |
| AC-8 | Impor **idempoten**. |
| AC-9 | Tidak ada DDL di kode Go. |
| AC-10 | `go build`/`vet`/`test`/`gofmt` lulus; ada unit test untuk normalizer. |

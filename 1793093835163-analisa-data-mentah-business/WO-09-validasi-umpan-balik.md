---
title: "WO-09: Validasi & Umpan Balik Error per Baris"
description: "Validator template/CSV yang mengembalikan file umpan-balik (XLSX/CSV) dengan kolom hasil_validasi dan alasan_ditolak per baris, sehingga tim input tahu persis apa yang salah dan bisa memperbaiki tanpa menebak."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-09: Validasi & Umpan Balik Error

**SPEC acuan:** `SPEC.md` §15.5 (Umpan balik validasi), D-17
**Prasyarat:** WO-08

## Latar
Tanpa umpan balik yang jelas, tim input akan berhenti memakai template begitu berkasnya ditolak. Validator harus **memberi tahu baris & kolom mana yang salah, dan mengapa**.

## Cakupan

Perintah:
```
resident template validate <path> [--out PATH] [--format xlsx|csv]
```

Default: file umpan-balik ditulis di sebelah berkas sumber dengan suffix `_hasil`.

### 1. Aturan validasi (dari `schema/fields.yaml`, bukan hard-code)
| Kategori | Contoh pesan |
|---|---|
| Field wajib kosong | `nama_bayi: wajib diisi` |
| Enum tak dikenal | `agama: "Islami" tidak dikenal (pilihan: ISLAM, HINDU, ...)` |
| Format tanggal salah | `tanggal_lahir: "2021-11-19" harus DD/MM/YYYY` |
| Angka tidak valid | `berat_lahir_gram: "2600 gram" harus angka tanpa satuan` |
| NIK bukan 16 digit | `nik_bayi: harus 16 digit` |
| Kode RM tidak sesuai pola | `resident_code: harus berformat BBH-<YY>-<P\|L>-<NNN>` |
| `resident_code` duplikat | `resident_code: duplikat dengan baris 12` |
| `template_version` tidak dikenal | `template_version: 0 ditemukan, versi yang didukung: 1` (level file) |
| Status TRANSFER tanpa catatan | `status: TRANSFER sebaiknya disertai keterangan tujuan (mis. "transfer ke JCH")` — **peringatan**, bukan penolakan |

### 2. Format file umpan-balik
- Menyalin **seluruh baris sumber** dan menambahkan dua kolom di akhir:
  - `hasil_validasi` → `valid` / `ditolak`
  - `alasan_ditolak` → pesan gabungan (`kolom: alasan; kolom: alasan`)
- Baris yang salah **tetap ditampilkan** (tidak dihapus) agar tim bisa memperbaiki langsung di file yang sama.
- Opsional: pewarnaan baris (merah) bila XLSX — **bukan syarat**.

### 3. Ringkasan
- Cetak ringkasan ke stdout: `total=N valid=X ditolak=Y`, dan daftar 10 alasan terbanyak.
- **Jangan** diam saat ada error (pelajaran `build.sh` WO-08 federasi: kegagalan tidak boleh tersembunyi).

## Non-Cakupan
- Menulis ke DB (impor nyata → WO-06).
- AI-ETL (topic LLM).

## Bukti yang Harus Dilaporkan
- Jalankan validate pada **file yang sengaja dirusak** (minimal 6 jenis kesalahan berbeda) → tunjukkan file `_hasil` dan isi `alasan_ditolak`.
- Bukti `total = valid + ditolak`.
- Bukti baris salah **tidak hilang** dari file umpan-balik.
- Bukti `template_version` salah → ditolak **di level file** dengan pesan jelas.
- Bukti aturan **berasal dari `fields.yaml`** (ubah satu aturan → perilaku validator berubah, tanpa mengubah kode).
- `go build`/`vet`/`test`/`gofmt` lulus; ada unit test per kategori kesalahan.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | `resident template validate` menghasilkan file umpan-balik. |
| AC-2 | File memuat kolom `hasil_validasi` & `alasan_ditolak` per baris. |
| AC-3 | Baris invalid **tetap ada** di file umpan-balik. |
| AC-4 | Minimal 6 kategori kesalahan terdeteksi dengan pesan spesifik (bukan generik); peringatan (mis. TRANSFER tanpa catatan) dibedakan dari penolakan. |
| AC-5 | `total = valid + ditolak` terbukti. |
| AC-6 | `template_version` tidak dikenal → ditolak di level file. |
| AC-7 | Aturan validasi dibaca dari `schema/fields.yaml`, bukan hard-code. |
| AC-8 | Ringkasan dicetak ke stdout dan **gagal terlihat** (exit code non-zero bila ada baris ditolak, atau flag `--strict`). |
| AC-9 | `go build`/`vet`/`test`/`gofmt` lulus. |
| AC-10 | Ada unit test untuk setiap kategori kesalahan. |

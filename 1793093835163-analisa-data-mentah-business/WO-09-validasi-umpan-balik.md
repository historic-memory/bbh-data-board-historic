---
title: "WO-09: Validasi & Umpan Balik Error per Baris"
description: "Validator template/CSV yang mengembalikan file umpan-balik (XLSX/CSV) dengan kolom hasil_validasi dan alasan_ditolak per baris, sehingga tim input tahu persis apa yang salah dan bisa memperbaiki tanpa menebak."
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 2
---

# WO-09: Validasi & Umpan Balik Error

**SPEC acuan:** `SPEC.md` §15.5 (Umpan balik validasi), D-17
**Prasyarat:** WO-08 (selesai)

## Latar
Tanpa umpan balik yang jelas, tim input akan berhenti memakai template begitu berkasnya ditolak. Validator harus **memberi tahu baris & kolom mana yang salah, dan mengapa**.

## Keputusan bentuk: HTTP API (bukan CLI)
Dokumen awal menulis perintah CLI `resident template validate <path>`. **Owner
memilih HTTP API** (konsisten dengan keputusan WO-05 & WO-08), sehingga validasi
memakai endpoint:

```
POST /api/admin/resident-template/validate            (multipart field `file`)
  ?format=xlsx|csv   # format file umpan-balik (default = ikut format unggahan)
  ?strict=1          # balas 422 bila ada baris ditolak (default 200)
```

Validator **tidak menulis ke DB** (impor nyata = WO-06), jadi cukup guard
`resident.read`. Ringkasan dicetak ke log server (stdout) + header
`X-Validate-Total/Valid/Rejected`.

## Cakupan

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

## Hasil (2026-10-27) — status: complete

### Berkas
- `plugins/resident/services/validate.go` — parser XLSX/CSV, mesin aturan, builder file umpan-balik + sheet RINGKASAN.
- `plugins/resident/handlers/validate.go` — handler HTTP + log ringkasan.
- `plugins/resident/schema/fields.yaml` — blok `validation:` (date_format + rules).
- `plugins/resident/schema/schema.go` — struct `ValidationRule`/`Validation`, helper `RulesFor`/`EnumCodes`.
- `plugins/resident/plugin.go` — route `POST /admin/resident-template/validate`.
- `plugins/resident/services/validate_test.go` — 5 test.
- `api-tests/templates/04-Validate-template.bru` + `api-tests/fixtures/broken-template.csv`.

### Aturan (100% dari fields.yaml — AC-7)
- Diturunkan dari metadata: `required` → "wajib diisi"; enum → daftar pilihan; `type: date` → `DD/MM/YYYY`; `type: int` → "angka tanpa satuan"; `type: varchar16` → "harus 16 digit".
- Blok `validation.rules`: `resident_code` → regex `^BBH-[0-9]{2}-[PL]-[0-9]{3}$`; status → `warn_when_in [TRANSFER]` (peringatan, bukan penolakan).
- Duplikat `resident_code` terdeteksi in-file.
- **Bukti data-driven:** test `TestValidateDataDriven` mengubah `validation.date_format` → `YYYY-MM-DD`; nilai `2021-11-19` berpindah dari ditolak → diterima **tanpa** mengubah kode.

### Bukti (file sengaja dirusak, 9 baris, 7 ditolak + 1 peringatan)
```
POST /api/admin/resident-template/validate?strict=1  →  HTTP 422
Header: X-Validate-Total=9  X-Validate-Valid=2  X-Validate-Rejected=7
Content-Disposition: attachment; filename="broken_hasil.csv"
```
7 kategori kesalahan terdeteksi (pesan spesifik):
| Baris | Kolom | Pesan |
|---|---|---|
| 3 | Nama Bayi | `wajib diisi` |
| 4 | Agama | `"Islami" tidak dikenal (pilihan: ISLAM, HINDU, …)` |
| 5 | Tanggal Lahir | `"2021-11-19" harus DD/MM/YYYY` |
| 6 | Berat Lahir (gram) | `"2600 gram" harus angka tanpa satuan` |
| 7 | NIK Bayi | `harus 16 digit` |
| 8 | Kode RM | `harus berformat BBH-<YY>-<P|L>-<NNN>` |
| 9 | Kode RM | `duplikat dengan baris 2` |
| 10 | Status | `PERINGATAN … status TRANSFER sebaiknya disertai keterangan tujuan` (baris **valid**) |

- Baris invalid **tetap ada** di file umpan-balik (CSV: header + 9 baris; XLSX: sheet `HASIL` + `RINGKASAN`).
- `total(9) = valid(2) + ditolak(7)`.
- **AC-6 (file-level):** test `TestValidateXLSXVersionRejectsFile` → `template_version=99` menolak seluruh file dengan pesan `template_version: 99 ditemukan, versi yang didukung: 1`.
- **AC-8:** ringkasan tercetak di log server; `strict=1` → 422 (non-strict → 200).
- Varian: tanpa token → 401; viewer (`resident.read`) → 200.

### Bukti otomatis
- `go vet ./...` OK · `gofmt -l` bersih · `go test ./plugins/resident/...` ok (5 test WO-09 lulus).
- `bru run --env Local templates` — validate **4/4 test** lulus (200, file `_hasil.csv`, header ringkasan, isi body).
- Swagger diregenerasi di container: `paths_total=36`, `resident_paths=9`, `resident_ops=16` (route validate masuk).

---
title: "WO-10: Generator resident_code"
description: "Generator kode resident_code berformat BBH-<YY>-<P|L>-<NNN> yang unik global (D-8), aman terhadap konkurensi, dapat dijalankan dari CLI/API/importer, dan siap diperluas ke prefix cabang BBH-<CAB>-<YY>-<P|L>-<NNN>."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-10: Generator `resident_code`

**SPEC acuan:** `SPEC.md` §5 (Aturan `resident_code`), D-8
**Prasyarat:** WO-04

## Latar
19 dari 50 baris sumber memiliki `resident_code` **kosong**. D-8 memutuskan kode **UNIQUE global**, sehingga:
- kode yang kosong disimpan `NULL` (bukan string kosong) agar tidak bentrok,
- sistem butuh **generator** agar resident baru mendapat kode baku otomatis.

## Format
```
BBH-<YY>-<P|L>-<NNN>            (sekarang)
BBH-<CAB>-<YY>-<P|L>-<NNN>      (siap diperluas, bila cabang mengisi sendiri)
```
| Bagian | Arti |
|---|---|
| `BBH` | Prefix yayasan |
| `CAB` | (opsional, belum aktif) kode cabang dari `branch_key` |
| `YY` | 2 digit tahun (`time.Now().Year() % 100`) |
| `P`/`L` | Gender resident |
| `NNN` | Urutan **per gender** (per tahun), 3 digit, zero-padded |

## Cakupan

### 1. Service (`plugins/resident/services/code.go`)
```go
GenerateResidentCode(tx *gorm.DB, gender string, year int) (string, error)
```
- Ambil nomor terbesar yang sudah ada untuk prefix + gender + tahun tersebut.
- **Aman konkurensi:** kunci baris/tabel atau `SELECT ... FOR UPDATE` dalam transaksi, lalu retry bila terjadi bentrok UNIQUE.
- Bila sudah mencapai `999` → kembalikan error jelas (jangan overflow diam-diam).
- **Idempoten terhadap data lama:** kode yang sudah ada (mis. `BBH-26-P-001`) tetap dianggap nomor sah; generator melanjutkan dari yang terbesar (gap diizinkan, **tidak** mengisi gap).

### 2. Pemakaian
| Pemakai | Perilaku |
|---|---|
| API create (`POST /api/admin/residents`) | Bila `resident_code` **tidak** dikirim → generate otomatis |
| API create dengan kode eksplisit | Divalidasi format + keunikan; duplikat → `409` |
| CLI `resident add` | Sama seperti API; opsi `--code` untuk eksplisit |
| Importer CSV/template | Bila kolom kosong → **tidak** generate otomatis secara default; dicatat di laporan. Flag `--generate-missing` untuk mengaktifkan |

### 3. Validator format
- Regex: `^BBH-\d{2}-[PL]-\d{3}$` (varian prefix cabang: `^BBH-[A-Z]{2,5}-\d{2}-[PL]-\d{3}$`).
- Konsistensi gender: kode dengan `-P-` untuk gender `P`, `-L-` untuk `L`; tidak konsisten → ditolak.

## Non-Cakupan
- Migrasi/backfill kode untuk data lama (dilakukan manual/terpisah agar tidak mengubah data historis).
- Pemakaian aktif prefix cabang (hanya **disiapkan** strukturnya).

## Bukti yang Harus Dilaporkan
- `resident add` 3 bayi perempuan + 2 laki-laki → kode berurutan `...-P-001..003`, `...-L-001..002`.
- Generate **200 kode** paralel (uji konkurensi) → **nol duplikat**.
- Kode melampaui `999` → error jelas (bisa disimulasikan dengan data uji).
- Kirim kode duplikat eksplisit → `409`.
- Kode dengan gender tidak konsisten (`-L-` padahal `P`) → ditolak.
- Kode kosong pada impor → `NULL` di DB (bukan `''`), dan tercatat di laporan.
- `go build`/`vet`/`test`/`gofmt` lulus; ada unit test untuk regex & urutan.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | `GenerateResidentCode` menghasilkan format `BBH-<YY>-<P\|L>-<NNN>`. |
| AC-2 | Nomor urut **per gender** dan per tahun. |
| AC-3 | **Aman konkurensi** — uji paralel tanpa duplikat. |
| AC-4 | Tidak mengisi gap; melanjutkan dari nomor terbesar. |
| AC-5 | Melampaui `999` → error jelas, bukan overflow. |
| AC-6 | Kode eksplisit duplikat → `409`; format invalid → `400`. |
| AC-7 | Gender pada kode konsisten dengan `residents.gender`. |
| AC-8 | Kode kosong tersimpan `NULL`, tidak string kosong. |
| AC-9 | Struktur siap prefix cabang (fungsi menerima parameter cabang opsional, walau belum aktif). |
| AC-10 | `go build`/`vet`/`test`/`gofmt` lulus; ada unit test. |

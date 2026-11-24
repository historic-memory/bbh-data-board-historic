---
title: wo 02 dummy excel
description: 5 file Excel dummy per segment sesuai template import
status: complete
created: "2026-11-24"
updated: "2026-11-24"
---
# wo 02 dummy excel

## Ringkasan
Buat **5 file Excel** (data dummy fiktif) per segment, header sama persis
dengan template import yang sudah ada di `api-tests/fixtures/`:

| # | Segment | Acuan header | Format tanggal |
|---|---|---|---|
| 1 | Resident | `resident-valid-template.csv` (22 kolom) | DD/MM/YYYY |
| 2 | HSH | `hsh-valid-template.csv` | periode YYYY-MM |
| 3 | Sec | `sec-valid-template.csv` | periode YYYY-MM |
| 4 | Visit | `visit-valid-template.csv` | "05 November 2026 10:00" |
| 5 | Warehouse | `wh-import-valid.csv` | YYYY-MM-DD |

## Detail user
- 5 file terpisah (bukan satu workbook multi-sheet).
- Data dummy dari **awal 2026 (Januari)** dan seterusnya.
- Letakkan di folder yang jelas, mis. `api-tests/fixtures/dummy-excel/` atau
  `deploy/dummy/` (pilih tempat yang konsisten dengan repo; dokumentasikan).
- Nilai enum mengikuti yang valid di template/validasi import
  (mis. Jenis Kelamin P/L, Status DIRAWAT, URGENT/ENOUGH, status kunjungan).
- Fasilitas/produk/gudang boleh fiktif; pastikan nama konsisten antar baris.

## Tugas
1. [x] Susun 5 file .xlsx (generator Python + `openpyxl` di host, menyalin
       template asli server — bukan retype CSV).
2. [x] Verifikasi file bisa dibaca & header cocok 1:1 dengan template.
3. [x] Tes import via endpoint (semua segmen) → created = jumlah baris.
4. [x] Update WO dengan hasil + dokumentasi folder.

## Acceptance
- 5 file .xlsx per segment siap dipakai import manual.
- Header & format nilai sesuai template valid.

### Bukti

Header diperiksa programatik terhadap template asli hasil `GET
/api/admin/<segmen>-template?format=xlsx` — **identik** untuk kelima segmen:

| Segment | Kolom | Header identik | Sheet identik | Baris data |
|---|---|---|---|---|
| resident | 23 | ✅ | ✅ | 12 |
| hsh | 8 | ✅ | ✅ | 10 |
| sec | 10 | ✅ | ✅ | 8 |
| visit | 6 | ✅ | ✅ | 12 |
| wh | 8 | ✅ | ✅ | 10 |

Validasi (`POST /api/admin/<segmen>-template/validate?format=xlsx`):
resident 12/12, hsh 10/10, sec 8/8, visit 12/12, wh 10/10 **valid, 0 ditolak**.

Impor nyata (`POST /api/admin/<segmen>-template/import?format=xlsx`):
resident `{total:12, created:12, rejected:0}`, hsh `{10,10,0}`, sec `{8,8,0}`,
visit `{12,12,0}`, wh `{10,10,0}`.

Setelah uji, DB dibersihkan kembali: 308 baris dihapus +
`--reseed` memulihkan 7 kategori hsh / 2 kategori sec (lihat WO-01).

### Berkas

`api-tests/fixtures/dummy-excel/`

| Berkas | Ukuran | Isi |
|---|---|---|
| `generate.py` | — | generator: salin template → isi sheet `DATA` |
| `dummy-resident.xlsx` | 13.364 B | 12 warga fiktif |
| `dummy-hsh.xlsx` | 10.129 B | 10 baris kebutuhan hsh |
| `dummy-sec.xlsx` | 10.385 B | 8 baris kebutuhan sekunder |
| `dummy-visit.xlsx` | 8.588 B | 12 kunjungan |
| `dummy-wh.xlsx` | 9.600 B | 10 baris gudang |
| `README.md` | — | cara pakai, prasyarat, regenerasi, bukti |

## Catatan
- Enum yang dipakai: persalinan NORMAL/CAESAR/VAKUM/SENDIRI, penolong
  DOKTER/BIDAN/SENDIRI, status warga DIRAWAT/DIKEMBALIKAN, status kebutuhan
  URGENT/ENOUGH/UNKNOWN, status kunjungan
  SCHEDULED/DONE/CANCELLED/NO_SHOW/RESCHEDULED.
- Semua data fiktif, nama diberi sufiks `Dummy` agar mudah dikenali.
- **Prasyarat wajib impor:** kategori hsh/sec harus ada (impoter menolak kode
  tak dikenal) → jalankan `clean-data --yes --reseed`; gudang wh harus sudah
  terdaftar (fail-closed O-2). Kategori & produk gudang auto-create (O-1).

## Catatan Implementasi (Dev)

- `api-tests/fixtures/dummy-excel/generate.py:1-30` — header modul: menjelaskan
  strategi "salin template, bukan retipe header". Kenapa: jaminan 1:1 hanya
  bisa dipertahankan jika bukan hasil ketik ulang.
- `api-tests/fixtures/dummy-excel/generate.py` — `fill()` membuka workbook
  template dengan `openpyxl.load_workbook`, mencari sheet `DATA`, menulis baris
  mulai baris 2, lalu **raise `SystemExit`** bila `len(row) != jumlah kolom
  header`. Kenapa: pergeseran satu kolom adalah kegagalan senyap yang baru
  ketahuan saat impor; gagal saat generate jauh lebih murah.
- `api-tests/fixtures/dummy-excel/generate.py` — `--templates` / `--out`
  dipisah supaya template tidak muncul sebagai artefak repo. Kenapa: template
  masuk ke `/tmp/tpl` hasil unduhan server, bukan disimpan ganda di repo.
- `generate.py` — `RESIDENT_ROWS` 12 baris × 23 kolom, `Kode RM` sengaja kosong.
  Kenapa: server lalu membangkitkan `BBH-<YY>-<P|L>-<NNN>` — sekaligus menguji
  jalur auto-numbering, bukan hanya menerima nilai jadi.
- `generate.py` — `HSH_ROWS` / `SEC_ROWS` memakai kode kategori yang benar-benar
  ada di seed (`baby_needs`, `milk`, `food`, `stationery`, `medicine`,
  `housekeeping`, `linen`; `elektronik-peralatan`, `mainan-rekreasi`).
  Kenapa: importer resolve kategori lewat tabel, kode asal-asalan = ditolak.
- `generate.py` — `VISIT_ROWS` memakai format `DD Bulan YYYY HH:MM` dengan nama
  bulan Indonesia, bukan `YYYY-MM-DD HH:MM`. Kenapa: itu format yang diterima
  parser visit; mencoba format ISO akan gagal validasi.
- `generate.py` — `WH_ROWS` memakai satuan `unit` setelah 3 baris `botol`
  **ditolak** validasi gudang. Kenapa: REFERENSI unit gudang tertutup
  (pcs/unit/set/box/pak/lusin/gram/kg/liter) dan `botol` tidak termasuk.
- `api-tests/fixtures/dummy-excel/README.md:1-18` — tabel berkas → segmen →
  endpoint impor → jumlah baris. Kenapa: agar operator tidak menebak endpoint.
- `api-tests/fixtures/dummy-excel/README.md:20-45` — bagian "Bagaimana header
  dijamin sama" + perintah regenerasi lengkap (unduh template dengan token,
  lalu `generate.py`). Kenapa: dummy bisa basi bila template berubah.
- `api-tests/fixtures/dummy-excel/README.md:46-62` — **prasyarat** impor:
  kategori hsh/sec wajib ada, gudang wh wajib terdaftar, fasilitas opsional
  (NULL). Kenapa: dua kegagalan lapangan saat pengujian berasal dari sini.
- `api-tests/fixtures/dummy-excel/README.md:63-88` — tabel format tanggal per
  segmen + daftar enum yang dipakai. Kenapa: validasi segmen memakai enum
  tertutup; salah satu nilai = seluruh baris ditolak.
- `api-tests/fixtures/dummy-excel/README.md:89-101` — bukti verifikasi nyata
  (validate 12/10/8/12/10; import created = jumlah baris, rejected 0) dan
  perintah pembersihan. Kenapa: reviewer bisa mereproduksi tanpa bertanya.
- `api-tests/README.md` — baris `fixtures/dummy-excel/` ditambahkan di blok
  struktur. Kenapa: folder baru harus ditemukan dari indeks utama, bukan hanya
  dari ingatan orang yang membuatnya.

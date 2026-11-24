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
1. [x] Susun file .xlsx (generator Python + `openpyxl` di host, menyalin
       template asli server — bukan retype header).
2. [x] Verifikasi file bisa dibaca & header cocok 1:1 dengan template.
3. [x] Tes validasi via endpoint (semua segmen) → 30/30 valid, 0 ditolak.
4. [x] Update WO dengan hasil + dokumentasi folder.

## Acceptance
- Berkas .xlsx per segment siap dipakai import manual.
- Header & format nilai sesuai template valid.

## REVISI (owner, 2026-11-24): struktur per bulan + data-only

Owner minta **real case**: impor bulan demi bulan seperti operator sungguhan.
Keputusan yang mengubah bentuk keluaran:

| # | Keputusan | Alasan |
|---|---|---|
| R-1 | **30 baris per segmen per bulan**, Januari–Desember 2026 | Skala realistis; 360 baris/segmen |
| R-2 | **Satu folder per bulan** (`2026-01-januari` … `2026-12-desember`), 5 berkas di dalamnya | Real case impor bulanan; urut di file explorer |
| R-3 | **Data saja** — sheet `PETUNJUK`/`REFERENSI`/`CONTOH` dihapus, dropdown dibersihkan | Permintaan owner: "cukup datanya aja, enggak perlu petunjuk referensi dll" |
| R-4 | **Tanpa insert ke DB** — hanya berkas Excel | Permintaan owner: "hanya excelnya aja yaa bukan insertnya" |
| R-5 | Item `hsh`/`sec` **diulang** tiap bulan (30 item tetap) | Kebutuhan rumah tangga memang berulang; item baru tiap bulan hanya menggelembungkan master |
| R-6 | Verifikasi cukup `POST /validate` (read-only) | Bukti diterima server tanpa menulis DB |

R-3 aman secara teknis dan **diverifikasi di kode**, bukan diasumsikan:
validator kelima plugin membaca `f.GetRows("DATA")` saja
(`plugins/{resident,hsh,sec,visit,warehouse}/services/validate.go`), dan
`readTemplateVersion` mengembalikan `-1` bila `PETUNJUK` tidak ada; pemakaian
versi dijaga `if in.Version >= 0 && in.Version != schema.Version`, sehingga
`-1` melewati cek versi. `REFERENSI` tidak dipakai validator sama sekali
(referensi diambil dari DB untuk hsh/sec/wh, enum statis untuk resident/visit).

### Hasil akhir

**60 berkas, 1.800 baris** (12 bulan × 5 segmen × 30 baris).

### Bukti

Header & bentuk (lokal, 60 berkas diperiksa):

| Segment | Kolom | Header identik | Sheet | Baris/berkas |
|---|---|---|---|---|
| resident | 23 | ✅ | `['DATA']` saja | 30 |
| hsh | 8 | ✅ | `['DATA']` saja | 30 |
| sec | 10 | ✅ | `['DATA']` saja | 30 |
| visit | 6 | ✅ | `['DATA']` saja | 30 |
| wh | 8 | ✅ | `['DATA']` saja | 30 |

Data validation tersisa: **0** di semua berkas (dropdown dibersihkan agar Excel
tidak mengeluh sheet rujukan hilang).

Integritas lintas bulan:

| Pemeriksaan | Hasil |
|---|---|
| NIK bayi unik | 360/360 |
| NIK ibu unik | 360/360 |
| No HP ibu unik | 360/360 |
| Nama resident unik | 360/360 |
| No HP visit unik | 360/360 |
| Nama visit unik | 360/360 |
| Irisan nama resident ↔ visit | 0 |
| Kunci `(periode,item)` hsh | 360/360 unik |
| Kunci `(periode,item)` sec | 360/360 unik |
| Kunci `(gudang,produk,tgl,qty)` wh | 360/360 unik |
| Tanggal jatuh di bulan folder | 360/360 untuk kelima segmen |
| Satuan/enum di luar daftar resmi | 0 |

Validasi server (`POST /api/admin/<segmen>-template/validate?format=xlsx`,
read-only) — **seluruh 60 berkas**:

| Segment | Total | Valid | Ditolak |
|---|---|---|---|
| resident | 30 | 30 | 0 |
| hsh | 30 | 30 | 0 |
| sec | 30 | 30 | 0 |
| visit | 30 | 30 | 0 |
| wh | 30 | 30 | 0 |

(× 12 bulan; tidak ada satu pun berkas dengan `rejected > 0`.)

DB **tidak berubah** setelah validasi: `residents=0`, `visit_visits=0`,
`hsh_entries=0`, `wh_batches=0`, `admins=20` — membuktikan tidak ada insert.

### Dua bug yang ditemukan saat verifikasi

1. **Nama dobel 20 baris** — `Eka` ada di daftar nama pria *dan* wanita, jadi
   `Eka Adnyana`/`Eka Baskara`/… muncul dua kali. Diperbaiki: `Eka` (wanita)
   → `Elisa`.
2. **Nama resident = nama visit** — generator memakai offset yang sama untuk
   kedua pool. Terlihat seperti copy-paste bug pada dataset "real case".
   Diperbaiki: visit memakai offset 180 (separuh pool yang tidak dipakai
   resident), sehingga irisan = 0.

Selain itu ditemukan **satuan tidak valid** yang akan menolak baris:
`set` dipakai di item hsh (`Krayon`, `Baju bayi`) padahal daftar satuan hsh
tidak memuat `set` (hanya ada di sec/wh). Diganti `box`/`lusin`.

## Catatan Implementasi (Dev) — Revisi R-1..R-6

- `generate.py:1-35` — docstring ditulis ulang: menjelaskan bentuk keluaran
  per bulan dan **tiga keputusan owner** (30 baris, data-only, tanpa insert).
  Kenapa: pembaca harus tahu ini disengaja, bukan kelalaian.
- `generate.py:36-60` — `ROWS_PER_MONTH = 30` + `MONTHS` + `MONTH_NAMES_ID`.
  Kenapa: satu konstanta mengendalikan skala; nama bulan Indonesia dipakai
  karena parser visit menerima `DD Bulan YYYY HH:MM`.
- `generate.py:112-118` — `name_stream(gender, offset)` mengembalikan 400
  kombinasi (20 nama depan × 20 marga) lalu dirotasi. Kenapa: butuh 360 nama
  unik per segmen tanpa menulis 360 literal.
- `generate.py:120-130` — `RESIDENT_NAMES` interleave L/P; `VISIT_NAMES` pakai
  **offset 180**. Kenapa: offset sama membuat nama resident = nama visit
  (bug nyata yang ditemukan saat verifikasi).
- `generate.py:` `FEMALE_FIRST` — `Eka` diganti `Elisa`. Kenapa: `Eka` juga ada
  di `MALE_FIRST` → 20 nama dobel (bug nyata kedua).
- `generate.py:` `HSH_ITEMS` — 30 item, satuan hanya dari
  `plugins/hsh/schema/fields.yaml` (pcs/box/botol/pak/roll/unit/lusin/kg/gram/
  liter/ml/kaleng). Kenapa: `set` sempat dipakai dan **tidak ada** di daftar
  hsh → baris akan ditolak.
- `generate.py:` `SEC_ITEMS` — 30 item; satuan hanya pcs/unit/set/box/pak/
  roll/lusin/meter. Kenapa: daftar sec beda dari hsh; memakai daftar hsh akan
  menolak `botol`/`kaleng`.
- `generate.py:` `WH_ITEMS` + `WH_WAREHOUSE` — 30 produk, 3 gudang dummy
  (Utama/Obat/Linen). Kenapa: gudang **fail-closed (O-2)** harus terdaftar
  dulu; kategori/produk auto-create (O-1). Satuan wh tidak memuat `botol`.
- `generate.py:` `resident_rows()` — `Kode RM` dikosongkan dan NIK dibangun
  `5171<kec><DDMMYY><serial>` (16 digit). Kenapa: `Kode RM` kosong menguji
  jalur auto-numbering server (`BBH-26-P/L-nnn`), dan NIK harus unik 360 kali.
- `generate.py:` `hsh_rows()`/`sec_rows()` — periode = `f"2026-{month:02d}"`,
  item diambil dari katalog tetap. Kenapa: kunci idempotensi importer
  `(periode, item)`; item baru tiap bulan akan menggelembungkan master.
- `generate.py:` `visit_rows()` — HP dibangun dari 5 prefix
  (0812/0813/0821/0857/0878) + serial unik. Kenapa: `phone_e164` adalah kunci
  dedup visitor; HP kembar = jumlah visitor < 360.
- `generate.py:` `wh_rows()` — hari dibuat `2 + i % 26` sehingga tiap produk
  punya tanggal berbeda. Kenapa: kunci idempotensi
  `(gudang, produk, tanggal masuk, qty)` tidak boleh bertabrakan dalam satu bulan.
- `generate.py:` `fill()` — menghapus semua sheet selain `DATA`, lalu
  `ws.data_validations.dataValidation = []`. Kenapa dua-duanya: sheet
  rujukan hilang tanpa membersihkan dropdown membuat Excel melaporkan berkas
  rusak (dropdown menunjuk sheet yang sudah tidak ada).
- `generate.py:` `fill()` — tetap `raise SystemExit` bila jumlah sel ≠ lebar
  header. Kenapa: pergeseran satu kolom adalah kegagalan senyap yang baru
  ketahuan saat impor.
- `generate.py:` `main()` — `--months` (default `1-12`), indeks resident/visit
  naik 30 per bulan. Kenapa: memungkinkan regenerasi sebagian bulan saat
  debug tanpa kehilangan keunikan lintas bulan.
- `generate.py:` `random.Random(month * 7919 + len(segment))` — seed tetap per
  bulan+segmen. Kenapa: keluaran deterministik; regenerasi menghasilkan berkas
  identik sehingga diff git tidak berisik.
- `README.md:1-30` — pohon folder + total 60 berkas/1.800 baris + penjelasan
  kenapa `DATA` saja aman (rujuk ke perilaku validator, bukan asumsi).
- `README.md:` bagian "Karakter data" — tabel jaminan per segmen (NIK unik,
  HP unik, kunci unik). Kenapa: reviewer bisa memeriksa klaim tanpa menjalankan
  ulang generator.
- `README.md:` bagian "Prasyarat" — kategori hsh/sec wajib ada (`--reseed`),
  gudang wh fail-closed, fasilitas resident opsional. Kenapa: dua kegagalan
  lapangan saat pengujian berasal dari sini.

---

## Riwayat versi pertama (usang, disimpan sebagai catatan)

Versi awal WO-02 menghasilkan **5 berkas datar** (`dummy-resident.xlsx` dst.)
dengan 12/10/8/12/10 baris dan sudah diuji impor nyata (created = jumlah baris,
rejected 0) sebelum DB dibersihkan lagi. Berkas itu **dihapus** dan diganti
struktur per bulan (R-1/R-2); angka dan bukti impor di atas tidak lagi berlaku
untuk keluaran sekarang. Satu temuan dari versi lama tetap relevan dan
dipertahankan: satuan gudang `botol` **ditolak** REFERENSI wh sehingga diganti
`unit`.

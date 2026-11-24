---
title: konteks
description: "Scope clean data per segment + rencana data dummy excel sesuai template import"
status: create
created: "2026-11-24"
updated: "2026-11-24"
---
# konteks

## Kebutuhan user
1. **Clean data semua segment kecuali user admin** — kosongkan data transaksional
   & master data fitur, TAPI data user/admin (login, grup RBAC) tetap dipertahankan.
2. **Bikin data dummy semua segment dalam bentuk Excel** — sesuai template import
   yang sudah ada (CSV template di `api-tests/fixtures/`), supaya bisa dipakai
   mengisi aplikasi via fitur import / seed.

## Segment yang punya data
| Segment | Tabel data utama | Tabel pendukung/master |
|---|---|---|
| resident | residents, mothers, admissions, birth_records | resident_documents, resident_status_history |
| hsh | hsh_entries | hsh_periods, hsh_categories, hsh_items, hsh_item_aliases |
| sec | sec_entries | sec_periods, sec_categories, sec_items, sec_item_aliases |
| visit | visit_visits | visit_visitors, visit_status_history, visit_visitor_merges |
| warehouse | wh_movements, wh_batches | wh_warehouses, wh_categories, wh_items |

## Yang HARUS dijaga (khusus admin/user)
- `admins` (20 baris), `admin_groups`, `groups`, `group_permissions`,
  `permissions` — RBAC & login (Root/Administrator/Viewer).
- Tabel infra framework: `migrations`, `migration_targets`, `endpoints`,
  `endpoint_audit`, plugin registry.

## Data saat ini (24 Nov 2026, sebelum clean)
- residents 0, mothers 76, facilities 11, hsh_entries 9, sec_entries 8,
  visit_visits 1611, wh_movements 0, admins 20.

## Template import yang ada (api-tests/fixtures/, format CSV)
- `resident-valid-template.csv` — kolom: Nama Bayi, Jenis Kelamin, Tanggal Lahir,
  Tempat Lahir, Agama, Gol Darah, NIK Bayi, BPJS, Nama Ibu, NIK Ibu, No HP Ibu,
  Berat Lahir, Panjang Lahir, Anak Ke, Persalinan, Ditolong Oleh, Fasilitas,
  Detail Tenaga Medis, Tanggal Masuk, Status, Keterangan, Folder Drive.
- `hsh-valid-template.csv` — Periode, Kategori, Nama Kebutuhan, Satuan,
  Harga Satuan, Status, Jumlah, Catatan.
- `sec-valid-template.csv` — Periode, Kategori, Nama Kebutuhan, Merk/Brand,
  Spesifikasi, Satuan, Harga Satuan, Status, Jumlah, Catatan.
- `visit-valid-template.csv` — Nama, Alamat, Jumlah Pengunjung, Nomor HP,
  Status Kunjungan, Waktu Kunjungan.
- `wh-import-valid.csv` — Kategori, Produk, Gudang, Jumlah Masuk, Satuan,
  Tanggal Masuk, Kadaluarsa, Catatan.

## Catatan implementasi
- Excel dummy disusun DARI template di atas (header sama persis),
  isi beberapa baris data fiktif realistis per segment.
- Cara eksekusi clean: kandidat = SQL TRUNCATE/DELETE dengan urutan FK aman,
  atau `console` CLI kalau ada command reset data. Perlu cek FK dulu.
- Tanggal format import: DD/MM/YYYY (resident/visit), YYYY-MM-DD (warehouse),
  periode YYYY-MM (hsh/sec).

## Hasil (24 Nov 2026)

### WO-01 — clean data
- Dibuat plugin baru `plugins/maintenance` (tanpa route HTTP) + command
  `console maintenance clean-data`. Default **dry run**, eksekusi butuh `--yes`.
- Dry run: 8.650 baris di 37 tabel. Eksekusi: semua tabel fitur → 0.
- Dijaga utuh: `admins` 20, `members` 3, `groups` 6, `group_permissions` 39,
  `admin_groups` 16, `permissions` 32, `endpoints` 1, `migrations` 33,
  `ingest_sessions` 355, `ingest_messages` 605.
- Health setelah clean: login admin OK (token terbit), `/api/admin/residents`
  200, `/api/admin/dashboard/summary` 200, `/healthz` 200.
- **Temuan penting:** reset ikut menghapus seed `hsh_categories` (7) &
  `sec_categories` (2); importer kedua segmen menolak kode kategori tak dikenal
  sehingga impor jadi mustahil. Solusi: flag `--reseed` (idempotent) yang
  memulihkan seed dari migrasi + test paritas vs file migrasi.
- Commit: `132fdb0` (plugin), `6144915` (`--reseed`). Tanpa push.

### WO-02 — data dummy Excel
- `api-tests/fixtures/dummy-excel/` berisi `generate.py`, 5 berkas `.xlsx`
  (resident/hsh/sec/visit/wh), dan `README.md`.
- Generator **menyalin template asli server** lalu mengisi sheet `DATA`, jadi
  header 1:1 secara konstruksi (bukan ketik ulang).
- Header identik terverifikasi: resident 23 kolom, hsh 8, sec 10, visit 6, wh 8;
  daftar sheet juga identik.
- Validasi: resident 12/12, hsh 10/10, sec 8/8, visit 12/12, wh 10/10 valid.
- Impor nyata: created 12/10/8/12/10, rejected 0 untuk kelima segmen.
- Setelah uji, DB dibersihkan lagi (308 baris) + `--reseed`.
- Prasyarat impor: kategori hsh/sec harus ada (`--reseed`); gudang wh harus
  terdaftar lebih dulu (fail-closed O-2); fasilitas resident opsional (NULL).

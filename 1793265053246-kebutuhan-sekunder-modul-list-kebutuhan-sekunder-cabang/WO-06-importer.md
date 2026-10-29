---
title: "WO-06: Importer (staging → upsert)"
description: "Endpoint impor template sec: fasa staging + upsert idempoten, alias-first matching, auto-create item/alias, tolak .xls."
status: complete
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - import
  - idempotency
---

# WO-06: Importer (staging → upsert)

## Referensi
- SPEC: S-12, S-18, S-19, §Alur Impor, GAP-2.
- Pola acuan: importer 2 fasa `plugins/hsh` WO-06 (`services/import.go`) / `plugins/resident`.

## Tugas
1. **Migrasi staging** `sec_import_batches` + `sec_import_rows`
   (batch = berkas unggahan; rows = baris mentah + status validasi + pesan + aksi).
2. **Fasa 1 — staging**: baca `.xlsx`/`.csv` (pola WO-05), simpan baris mentah, jalankan validator,
   tulis laporan (dibuat/diperbarui/dilewati/ditolak) **tanpa** menyentuh master/entri.
3. **Fasa 2 — upsert** (transaksi):
   - **Periode**: cocokkan `code`; buat bila belum ada (periode **boleh** auto-create dari kolom,
     kategori **tidak**). Periode hasil impor dibuat **nonaktif** (jangan mencuri "≤1 periode aktif").
   - **Item**: cocokkan urutan **alias** → **name persis (case/space-insensitive)** → **buat item baru**
     (S-19) + catat nama asli sebagai **alias**. Isi `brand`/`specification`/`unit`/`est_price` bila ada.
   - **Entri**: upsert per `(period_id, item_id)`; isi `status_code`, `quantity`, `unit_price`, `note`,
     `recorded_by`.
4. **Idempotensi**: menjalankan impor berkas sama 2x tidak menggandakan baris.
5. **Penolakan `.xls`**: deteksi BIFF8 → 400 + pesan pra-konversi (S-18).
6. **Endpoint**: `POST /admin/sec-template/import` (guard `sec.write`),
   multipart `file` + `?format=`, `?dry_run=1`. Respons: id batch + ringkasan.
   `GET /admin/sec/import-batches/:id` untuk membaca laporan.
7. **Test**: impor valid → count master/entri sesuai; impor ulang → tidak bertambah; item baru → alias
   tercatat; `.xls` → 400; baris invalid → dilaporkan & tidak menggagalkan baris valid.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/sec/...` hijau.
- curl impor → ringkasan; `SELECT count(*)` sebelum & sesudah impor ulang (tidak berubah);
  alias item baru muncul; `.xls` → 400; `?dry_run=1` → tidak menyentuh master.

## Kriteria Selesai
- Impor idempoten; staging menyimpan laporan yang bisa diaudit.
- Auto-create item + alias berjalan; kategori tidak auto-create.
- `.xls` ditolak jelas; tidak ada penulisan DB pada fasa validasi.

## Bukti Pelaksanaan

**Kode yang ditambahkan**
- `plugins/sec/migrations/postgres/2026_10_29_130300_create_sec_import_tables.{up,down}.sql` —
  versi 2 plugin `sec`: `sec_import_batches` + `sec_import_rows` (FK cascade, CHECK status/aksi,
  indeks `(created_at DESC)` dan `(batch_id,row_no)`). Versi 1 tidak diubah.
- `plugins/sec/schema/fields.yaml` — kedua tabel staging didaftarkan (bukan master: tidak masuk
  sheet `REFERENSI` maupun template). `TestFieldsMatchMigration` hijau.
- `plugins/sec/model/models.go` — `ImportBatch`, `ImportRow` (termasuk `brand`/`specification`).
- `plugins/sec/services/import.go` — importer 2 fasa (pola `plugins/hsh`), ditambah
  `brand`/`specification` pada staging, pembuatan item, dan back-fill `COALESCE(brand, ?)` /
  `COALESCE(specification, ?)` untuk item lama.
- `plugins/sec/handlers/import.go` + rute di `plugins/sec/plugin.go`:
  `POST /api/admin/sec-template/import` (guard `sec.write`),
  `GET /api/admin/sec/import-batches/:id` (guard `sec.read`), header `X-Import-Batch`.
- `plugins/sec/services/import_db_test.go` — 5 test DB nyata.
- `api-tests/fixtures/sec-import-sample.csv` — fixture contoh (2 valid, 2 ditolak).

**Keputusan penting**
1. Periode hasil impor dibuat `is_active = false` (insert via `map[string]any`, bukan struct —
   menghindari jebakan GORM `default:true` yang sudah tercatat di memori repo).
2. Kategori **tidak** pernah auto-create: kode kategori tak dikenal → baris ditolak.
3. Item baru → alias asli dicatat (`sec_item_aliases`); urutan pencocokan alias → nama
   (case/space-insensitive) → buat baru. Item lama hanya di-*refresh* `est_price`/`unit`, dan
   `brand`/`specification` diisi hanya bila masih NULL.
4. Duplikat baris dalam satu berkas (`periode|nama item`) → baris kedua `skipped`.
5. `?dry_run=1` berhenti setelah fasa 1 (`status=staged`); master/entri/periode tidak tersentuh.

**Verifikasi**
- `go build ./...`, `go vet ./plugins/sec/...`, `gofmt -l plugins/sec` bersih;
  `go test ./plugins/sec/...` hijau (5 test importer PASS, bukan skip).
- Migrasi: `./bin/console migrate --plugin sec up` → `sec: current=2 dirty=false pending=0`;
  kedua tabel staging terverifikasi ada.
- curl CSV 4 baris (2 valid, 2 invalid): `status=committed`,
  `total 4 / created 2 / rejected 2` (alasan `kategori tidak dikenal`, `Nama Kebutuhan: wajib
  diisi`), header `X-Import-Batch` terkirim.
- Impor ulang berkas sama → `updated 2 / created 0`; `SELECT count(*)` master/entri/alias tetap
  `2/2/2` (idempoten). `GET /api/admin/sec/import-batches/:id` mengembalikan batch + rows
  (termasuk `brand`, `specification`, `status_code`, `reason`/`message`, `raw`).
- Periode hasil impor `2026-11` dan `2026-12` tercatat `is_active = f`.
- `?dry_run=1` → `status=staged` dengan master tetap `0/0/0`.
- `.xls` (magic BIFF) → **400** + pesan konversi; berkas kosong → 400; tanpa token → **401**.
- Impor `.xlsx` dari template asli (`template_version=1`) → `created 2`, data masuk lengkap
  (brand/spesifikasi/satuan/harga/status/jumlah).
- Data uji dihapus setelah verifikasi; DB kembali `2 kategori, 0 item/alias/periode/entri/berkas`.

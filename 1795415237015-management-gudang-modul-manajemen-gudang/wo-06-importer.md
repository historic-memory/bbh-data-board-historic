---
title: WO-06 — Importer staging 2 fase
description: "POST /api/admin/wh-template/import + batch audit"
status: complete
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-06 — Importer staging 2 fase

## Prasyarat

WO-05 lulus; keputusan O-1/O-2 owner.

## Lingkup

- Migrasi v2 `2026_11_23_140600_create_wh_import_tables` — staging `wh_import_batches` + `wh_import_rows`; daftarkan di `fields.yaml` (paritas `TestFieldsMatchMigration`).
- `services/import.go` (pola hsh/sec WO-06): fase 1 parse+validasi+staging (`dry_run=1` berhenti di sini), fase 2 SATU transaksi — resolve/create kategori (O-1: usulan auto-create teks), resolve/create produk (O-1), resolve gudang (O-2: usulan fail-closed 400 bila tidak dikenal), buat batch (qty, entered_at, expires_at, warehouse_id).
- Endpoint `POST /api/admin/wh-template/import` guard `warehouse.write` (`?dry_run=1`, header `X-Import-Batch`) + `GET /api/admin/wh/import-batches/:id` guard read (`{batch, rows}`).
- Importer ini HANYA memasukkan barang (batch) — movement keluar tetap manual/chat.

## Acceptance Criteria

- AC-1 Impor CSV/xlsx → batch dibuat dengan umur/expiry benar; laporan created/updated/skipped/rejected.
- AC-2 Re-impor idempoten (pola dedup: gudang+produk+tanggal masuk+qty → duplikat `skipped`).
- AC-3 `dry_run` tidak menyentuh master; gudang tak dikenal → 400 (sesuai O-2 final).
- AC-4 `.xls` 400; tanpa token 401; viewer write 403.

## Catatan Implementasi (Dev)

### A. Migrasi v2 + schema (paritas)

1. `plugins/warehouse/migrations/postgres/2026_11_23_140600_create_wh_import_tables.up.sql:19` — `wh_import_batches` (id/filename/format/template_version/status CHECK staged|committed|failed/dry_run/total/created_count/updated_count/skipped_count/rejected_count/**new_masters**/error/created_by/created_at/updated_at). Kolom `new_masters` menyimpan JSON daftar master baru (O-1) agar rekapnya ikut terbaca di GET batch, bukan cuma di respons POST.
2. `…create_wh_import_tables.up.sql:40` — `wh_import_rows` (batch_id FK ON DELETE CASCADE, row_no, category_name, item_name, warehouse_name, unit, qty_in, entered_at, expires_at, note, valid, action CHECK pending|created|updated|skipped|rejected, reason, message, **batch_ref**, raw JSON). `entered_at`/`expires_at` sengaja **TEXT apa adanya** — staging = audit berkas, parsing DATE terjadi di fasa 2 (kalau diketik DATE, baris tanggal-rusak justru gagal disimpan padahal justru itu yang mau diaudit).
3. `…create_wh_import_tables.down.sql:4` — drop `wh_import_rows` lalu `wh_import_batches` (urutan FK).
4. `plugins/warehouse/schema/fields.yaml:97` + `:120` — kedua tabel staging didaftarkan di `tables:` sehingga `schema_test.TestFieldsMatchMigration`/`TestTablePrefix` mengawal paritas secara otomatis (tidak ada daftar kolom yang di-maintain terpisah).
5. `plugins/warehouse/model/models.go:99` + `:122` — `ImportBatch`/`ImportRow` (GORM) memetakan tabel staging untuk staging dan laporan audit.

### B. Service importer (`services/import.go`)

6. `services/import.go:108` — `ImportTemplate` orkestrasi dua fase. Guard sebelum parsing: kosong → 400 `berkas kosong`; `.xls` (ekstensi **atau** magic OLE2 `xlsMagic`) → 400 pesan pra-konversi; tanpa DB → 400; **header-saja → 400 `berkas tidak berisi baris data (header saja)`** (`:127`) supaya tidak ada sesi audit kosong yang membingungkan.
7. `services/import.go:130` — fasa 1 memakai **engine WO-05 apa adanya**: `parseInput` (deteksi format dari isi: PK = XLSX), `buildColSpecs`, `referenceIndex`, `validateRow`. Tidak ada duplikasi aturan — validator dan importer tidak mungkin berbeda pendapat.
8. `services/import.go:139` — `template_version` tak cocok = **file-level reject** (semua baris ditolak dengan alasan file), sama seperti WO-05.
9. `services/import.go:161` — batch audit dibuat dulu, lalu semua baris masuk `wh_import_rows` via `CreateInBatches` (`:228`). `raw` menyimpan baris mentah JSON sebagai jaring audit terakhir.
10. `services/import.go:236` — `dry_run` **berhenti di sini**: `finishImport(..., ImportStatusStaged, "")`. Nol tulisan ke `wh_categories`/`wh_items`/`wh_warehouses`/`wh_batches`.
11. `services/import.go:245` — `commitImport` = **SATU transaksi** (`s.db.Transaction`) untuk seluruh berkas: satu baris gagal → seluruh impor gagal (`status=failed` + `error` disimpan) — tidak ada commit separuh.
12. `services/import.go:431` — `resolveWarehouse` cocokkan nama `LOWER(TRIM(name))`. Tidak ketemu → `:266` **baris ditolak** dengan alasan `gudang "X" belum terdaftar — daftarkan dulu lewat menu Gudang (O-2: gudang tidak dibuat otomatis)`. Tidak ada auto-create gudang (O-2).
13. `services/import.go:444` — `findOrCreateCategory` (O-1) insert lewat **map** (`Table("wh_categories").Create(map[string]any{... "is_active": true})`) — bukan struct — supaya kolom ber-`default:true` tidak membajak nilai (gotcha hsh B-1b).
14. `services/import.go:471` — `findOrCreateItem` (O-1) cocokkan `(category_id, LOWER(TRIM(name)))` = kunci yang sama dengan `uq_wh_items_category_name`; mengembalikan `active` supaya pemanggil bisa menolak produk nonaktif; unit dari template dinormalkan ke huruf kecil.
15. `services/import.go:283` — **kontrak WO-03 ditegakkan di jalur impor**: item nonaktif → baris `rejected` (`produk "X" tidak aktif — batch baru tidak boleh masuk ke produk nonaktif (WO-03)`), bukan 500 dan bukan tetap dibuat.
16. `services/import.go:507` — `lookupBatch` = kunci dedup AC-2 `(warehouse_id, item_id, entered_at::date, qty_in::numeric)`. Karena dibuat di dalam transaksi yang sama, duplikat **dalam satu berkas** pun tertangkap ("duplikat pertama menang") — tidak perlu map `seen` terpisah.
17. `services/import.go:524` — `enrichBatch` mendefinisikan arti `updated`: batch lama **dilengkapi** hanya bila `note`/`expires_at` masih kosong; nilai yang sudah ada tidak pernah ditimpa, dan stok **tidak pernah** dijumlahkan ke batch lama (barang masuk selalu batch baru — WO-03).
18. `services/import.go:168` — batch dibuat lewat `createBatchTx` (refactor `services/batch.go:168`, dipakai bersama `CreateBatch` di `:156`): `qty_remaining = qty_in`, validasi qty/tanggal/item-aktif/gudang ada. **Tanpa duplikasi logika** — hanya `*gorm.DB` yang diparamkan agar bisa ikut transaksi importer (`ensureItemActive`/`ensureWarehouseExists` juga menerima `tx` di `batch.go:211`/`:231`).
19. `services/import.go:374` — daftar master baru dibatasi `importMaxNewMasters = 500` (`:44`) supaya respons tidak membengkak; rekap tetap lengkap untuk penggunaan normal.
20. `services/import.go:382` — `finishImport` menulis aksi/reason/message/batch_ref per baris + counter batch + `new_masters` JSON; `services/import.go:610` — `GetImportBatch` mengembalikan `{batch, rows}` (rows urut `row_no`), batch tak dikenal → `ErrNotFound` (404).

### C. HTTP (`handlers/import.go`, `plugin.go`)

21. `plugins/warehouse/handlers/import.go:41` — `POST /api/admin/wh-template/import` guard **`warehouse.write`**; header balasan `X-Import-Batch` (+`X-Import-Total/Created/Skipped/Rejected`); ringkasan + **setiap baris ditolak** dicetak ke log server (`[wh.import]`).
22. `handlers/import.go:87` — keputusan owner 2026-11-23 (dikonfirmasi saat eksekusi): gudang asing = **baris ditolak, HTTP tetap 200 + laporan** — supaya baris lain tetap masuk dan pengguna melihat baris mana yang perlu diperbaiki (bukan 400 yang membatalkan seluruh berkas seperti bunyi awal AC-3). `strict=1` disediakan sebagai gerbang opsional → **400** dengan body laporan yang sama, sehingga pemanggil yang ingin "gagal keras" pun terlayani.
23. `handlers/import.go:98` — `GET /api/admin/wh/import-batches/:id` guard **`warehouse.read`** → `{batch, rows}`.
24. `plugins/warehouse/plugin.go:113` — dua route didaftarkan dengan prefiks flat `/admin/wh-template/...` (menghindari wildcard `/admin/wh/*`) dan `/admin/wh/import-batches/:id`.

### D. Test & fixture

25. `plugins/warehouse/services/import_test.go:12` — guard non-DB (kosong, `.xls` ekstensi, `.xls` magic OLE2, tanpa DB), `:35` pemetaan kolom template generik (termasuk angka `1234,5` tanpa hard-code posisi), `:78` `template_version` absen = NULL.
26. `plugins/warehouse/services/import_db_test.go:104` — O-1: satu baris bersih → kategori+produk dibuat `is_active=true`, batch `qty_in=qty_remaining=24`, `entered_at`/`expires_at`/`note` benar, `new_masters` berisi 2 entri, baris staging `created` + `batch_ref` terisi.
27. `import_db_test.go:203` — O-2: baris bergudang asing **ditolak per baris** (alasan `belum terdaftar`, tanpa `batch_id`), 0 kategori/produk/gudang dibuat untuk baris itu, baris sehat di berkas yang sama tetap `created`.
28. `import_db_test.go:258` — AC-2: impor ulang `created 0 / skipped 2`, jumlah `wh_batches` & `wh_items` tidak berubah.
29. `import_db_test.go:290` — duplikat **dalam satu berkas**: `created 1 / skipped 1`, `SUM(qty_remaining)` tetap 3 (stok tidak dobel).
30. `import_db_test.go:314` — `updated`: batch lama tanpa catatan dilengkapi (`updated 1`), run ketiga `skipped 1` (tidak ada yang bisa dilengkapi lagi), jumlah batch tetap 1.
31. `import_db_test.go:360` — AC-3: `dry_run` → snapshot `{categories, items, warehouses, batches, movements}` **identik** sebelum/sesudah, `status=staged`, baris staging `pending`+`valid`.
32. `import_db_test.go:395` — kontrak WO-03: item nonaktif → `rejected 1` (alasan `tidak aktif`), 0 batch.
33. `import_db_test.go:429` — guard dengan DB: kosong/format tak dikenal/.xls → `ErrInvalid`, batch tak dikenal → `ErrNotFound`.
34. `api-tests/fixtures/wh-import-sample.csv` (4 baris: 3 unik + 1 duplikat dalam berkas), `wh-import-valid.csv` (4 baris untuk Bruno), `wh-import-gudang-asing.csv` (1 baris gudang asing), `wh-header-only.csv` (header saja → 400).
35. `api-tests/warehouse/07..17-*.bru` — 11 request baru: daftarkan gudang uji (`ZZ Gudang Impor Uji`, 201/409) → dry run → commit → re-impor idempoten → audit GET → viewer 403 → tanpa token 401 → `.xls` 400 → header-saja 400 → `strict=1` 400 → gudang asing ditolak per baris.

### Bukti

**Test (container `bbb-data-board-app-1`, `go test ./... -count=1`)**: 11 test importer PASS — `TestImportCreatesMastersAndBatch`, `TestImportUnknownWarehouseRejectsOnlyThatRow`, `TestImportIsIdempotent`, `TestImportDeduplicatesWithinFile`, `TestImportEnrichesExistingBatchNote`, `TestImportDryRunTouchesNoMaster`, `TestImportRejectsBatchForInactiveItem`, `TestImportGuardsWithDatabase` (DB), `TestImportTemplateGuards`, `TestImportCellMapsTemplateColumns`, `TestImportVersionPtr`. `go build ./... && go vet ./...` hijau, `gofmt -l plugins/warehouse/` kosong, `go test ./...` 0 FAIL (semua plugin).

**Migrasi**: `console migrate up --plugin warehouse` → `warehouse applied 2026_11_23_140600_create_wh_import_tables.up.sql` (version 2); `schema_test` (paritas fields.yaml ↔ migrasi) hijau.

**curl (host 8082 → container 8080, root `warehouse.*`)** — gudang uji `Gudang Impor Uji` didaftarkan lebih dulu (O-2):

| Kasus | Hasil |
|---|---|
| CSV 4 baris (3 unik + 1 duplikat) | **200**, `X-Import-Total: 4`, `X-Import-Created: 3`, `X-Import-Skipped: 1`, `X-Import-Rejected: 0`; `status=committed`; `new_masters` = 2 kategori + 3 produk |
| Re-impor berkas sama | **200**, `created 0 / updated 0 / skipped 4`; `new_masters []`; `wh_batches` tetap 3, `SUM(qty_remaining)` tetap 44.50, `wh_movements` 0 |
| `dry_run=1` berkas ber-master baru | **200**, `status=staged`, `created 0`, baris `pending`; `dry_cat=0 dry_item=0 batches=3` |
| Gudang asing (`wh-import-gudang-asing.csv`) | **200**, `rejected 1`, alasan `gudang "Gudang Belum Terdaftar Sama Sekali" belum terdaftar …`; `hantu_cat=0 hantu_item=0 hantu_wh=0` |
| `.xls` | **400** `berkas .xls (Excel lama) tidak didukung …` |
| Berkas 0 byte | **400** `berkas kosong` |
| Header saja | **400** `berkas tidak berisi baris data (header saja)` |
| Tanpa token | **401** `unauthorized` |
| Viewer (write) | **403** `izin ditolak: butuh warehouse.write` |
| Viewer (GET batch) | **200** (read tetap boleh) |
| `strict=1` + `dry_run=1` berkas rusak | **400**, body tetap `total 9 / rejected 7` |

**Isi batch benar** (`psql`): `Susu formula tahap 1 8.50/8.50 2026-11-21→2027-05-21`, `Tisu bayi 12/12 2026-11-22`, `Pampers ukuran M 24/24 2026-11-23→2027-11-23` — `qty_remaining = qty_in`, tanpa movement.

**Bruno**: `bru run auth/00-Login.bru warehouse -r --env Local --env-var baseUrl=http://127.0.0.1:8082 --env-var viewerToken=<viewer>` → **18 request / 46 test PASS** (folder warehouse, termasuk 11 request importer baru).

**Higiene**: seluruh data uji dihapus setelah verifikasi — `cats=0 items=0 whs=0 batches=0 movements=0 ibatches=0 irows=0`, admin uji (`zzviewer.wo06@test.local`) beserta baris `admin_groups`-nya dihapus (`testadmins=0`), server uji dimatikan, tanpa push.

### Catatan/Keputusan

- **AC-3 "gudang tak dikenal → 400" ditafsirkan sebagai "baris ditolak (HTTP 200 + laporan)"** sesuai tabel keputusan O-2 di handoff ("tidak cocok → baris rejected") yang lebih spesifik dan lebih baru; sudah dikonfirmasi owner saat eksekusi. `strict=1` menyediakan jalur 400 bila pemanggil memang ingin gerbang status.
- **`updated`** bukan upsert stok: artinya batch lama **dilengkapi** (note/expires_at yang masih kosong). Barang masuk selalu batch baru — konsisten WO-03.
- Importer **hanya** memasukkan barang (batch); movement keluar tetap manual/chat (sesuai lingkup).


## Review Manager (2026-11-23) — LULUS ✅

Verifikasi independen Manager:

| Cek | Hasil |
|---|---|
| `git show --stat dd5f74f` | Scope bersih: `plugins/warehouse/**` + fixture + Bruno ✅ |
| `go test -count=1` (container) | `schema` + `services` ok (11 test importer, 8 DB) ✅ |
| DB | staging 0/0; migrasi target `warehouse` = **v2** ✅ |
| Spec §8 | O-1/O-2 ditulis TERKUNCI — sinkron dgn Review WO-05 ✅ |
| Proses | 6 commit lokal tanpa push; AdminLayout/index.css tak disentuh ✅ |

**Keputusan review:**
1. **WO-06 DITERIMA.**
2. **Interpretasi AC-3 O-2 disetujui**: gudang asing = baris rejected dalam laporan (HTTP 200), bukan HTTP 400; `strict=1` = gerbang opsional 400. Konsisten dengan keputusan O-2 "baris ditolak" — pencatatan dev jujur & tepat.
3. `updated` = melengkapi batch lama (note/expires_at kosong), stok TIDAK dijumlahkan ulang — semantik benar, anti duplikasi stok.
4. Fase 1 memakai engine WO-05 apa adanya (nol duplikasi aturan) + create batch via `createBatchTx` bersama WO-03 — DRF (don't repeat yourself) terjaga.
5. Auto-create lewat insert map menghindari bajakan `default:true` — gotcha B-1b ditegakkan kembali.

**WO-06 DITERIMA. WO-07 DIBUKA — mulai sekarang** (lihat handoff terpisah di chat).

Jabatan: Manager Agent — Nama Tugas: Review WO-06 warehouse. Sayonara.

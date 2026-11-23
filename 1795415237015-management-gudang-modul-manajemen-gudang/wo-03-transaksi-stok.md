---
title: "WO-03 — Transaksi stok: batch masuk, movement, summary"
description: "Masuk/keluar barang + stok agregat + umur + near-expiry"
status: complete
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-03 — Transaksi stok (batch masuk, movement, summary)

## Prasyarat

WO-02 lulus.

## Lingkup

- **A. Masuk barang** — `POST /api/admin/wh/batches` (item_id, warehouse_id, qty_in > 0, entered_at, expires_at?, note?). `qty_remaining = qty_in` saat create.
- **B. Movement keluar/adjust** — `POST /api/admin/wh/movements` (batch_id, qty > 0, direction OUT|ADJUST, reason wajib utk ADJUST, destination?, moved_at): OUT/ADJUST-minus mengurangi `qty_remaining` dalam SATU transaksi (`FOR UPDATE`); stok kurang → 409; ADJUST-plus menambah. Tulis 1 baris movement append-only.
- **C. Read** — `GET /api/admin/wh/batches?item_id=&warehouse_id=` (terlama dulu), `GET /api/admin/wh/summary` — stok per item, umur hari (`today - entered_at`), near-expiry (`expires_at` <= ambang 30 hari / sudah lewat). Summary didaftarkan SEBELUM route `/:id` wildcard.
- List movement per batch.

## Acceptance Criteria

- AC-1 Masuk 201 → stok muncul di summary; umur hari benar.
- AC-2 Keluar mengurangi batch terpilih; qty > sisa → 409; `qty_remaining >= 0` CHECK tak bisa dilanggar.
- AC-3 ADJUST minus tanpa reason → 400; plus menambah stok.
- AC-4 Tanpa token 401; viewer read 200/write 403.
- AC-5 Test DB benar-benar RUN (bukan skip): idempotensi/race sederhana.

## Non-tujuan

Template/impor (WO-04..06), UI (WO-07).

## Catatan Implementasi (Dev)

### A. Masuk barang (`POST /wh/batches`)

- `plugins/warehouse/services/batch.go:19` — `BatchInput` menerima `entered_at` **wajib** sebagai string `YYYY-MM-DD` (bukan time.Time) supaya batas antara "tanggal gudang" dan zona waktu server tetap eksplisit; D-4 (tanpa jam) dijaga di lapisan input, bukan di DB.
- `plugins/warehouse/services/batch.go:156` — `CreateBatch` mem-fix `QtyRemaining = QtyIn` saat insert: `qty_remaining` hanya boleh berubah lewat movement, sehingga "stok awal" tidak pernah bisa ditulis klien secara terpisah (SPEC §2, stok = agregasi batch).
- `plugins/warehouse/services/batch.go:199` — `ensureItemActive`: satu query `SELECT id, is_active ... First()` + `errors.Is(gorm.ErrRecordNotFound)` (pola hsh) untuk memisahkan "item tak ada" dari "item non-aktif" — keduanya **400** `ErrInvalid` karena yang salah adalah payload, bukan resource yang dialamatkan.
- `plugins/warehouse/services/batch.go:219` — `ensureWarehouseExists` hanya cek eksistensi: `wh_warehouses` tidak punya `is_active` (SPEC §2, WO-02), jadi tidak ada flag kedua yang perlu diperiksa.
- `plugins/warehouse/services/batch.go:234` — `parseOptionalDate` mengembalikan `*time.Time` nil untuk `expires_at` kosong/null: `expires_at` nullable (D-4) dan near-expiry harus ikut mati bila tidak diisi, bukan dianggap "expired hari ini".
- `plugins/warehouse/services/batch.go:64` — `batchSelect` menurunkan `age_days` dan `near_expiry` **di SQL**, bukan di Go: definisi umur/ambang hanya boleh ada satu tempat agar `GET /batches` dan `GET /summary` tidak bisa berbeda tafsir (M-2 + SPEC §3).
- `plugins/warehouse/services/batch.go:~72` — plaseholder ambang near-expiry **di-bind paling awal** karena muncul di SELECT list (sebelum arg WHERE); urutan `params` (ambang → filter → limit/offset) diberi komentar agar tidak diacak saat WO-04..06 menambah filter.
- `plugins/warehouse/services/batch.go:64` — `CURRENT_DATE + (?::int)`: tanpa cast, Postgres menolak `date + unknown` (`SQLSTATE 42725 operator is not unique`) — ditemukan langsung saat test pertama; parameter hari harus di-cast ke `int` karena `date + integer` satu-satunya operator yang tak ambigu.
- `plugins/warehouse/services/batch.go:90` — `ListBatches` mengurutkan `entered_at ASC, created_at ASC` (terlama dulu = FIFO/FEFO, SPEC §3) dan menghitung `COUNT(*)` terpisah supaya `total` tidak terpotong `limit`.

### B. Movement (`POST /wh/movements`)

- `plugins/warehouse/services/movement.go:64` — `CreateMovement` menormalkan `direction` ke huruf besar lalu memvalidasi lewat `model.IsValidMovementDirection` (D-7 enum statis): `OUT`/`ADJUST` ditolak di layanan, bukan hanya dipercaya ke CHECK DB, agar pesan errornya 400 yang bisa dibaca UI.
- `plugins/warehouse/services/movement.go:64` — kontrak tanda dieksekusi **server-side**: `OUT` → `delta = -qty` dengan `qty > 0` wajib; `ADJUST` → `delta = qty` bertanda dan `qty != 0`. Klien tidak bisa menitipkan tanda sendiri, jadi tidak ada `OUT` negatif yang "mengisi" stok (SPEC §3).
- `plugins/warehouse/services/movement.go:64` — `reason` **wajib** untuk `ADJUST` (400 bila kosong) tapi opsional untuk `OUT`: penyesuaian tanpa alasan menghapus jejak audit, sedangkan keluar normal sudah punya `destination`.
- `plugins/warehouse/services/movement.go:~100` — satu `s.db.Transaction`: `tx.Clauses(clause.Locking{Strength: "UPDATE"}).First(&batch, "id = ?")` → hitung `next` → `UPDATE wh_batches SET qty_remaining, updated_at` → `Create(movement)`. Batch dikunci **sebelum** dibaca sehingga dua OUT bersamaan tidak bisa sama-sama melihat sisa 1 (SPEC §3 "tanpa race").
- `plugins/warehouse/services/movement.go:~100` — `next < 0` → `fmt.Errorf("%w: ...", ErrInsufficientStock)` di dalam transaksi: rollback menghapus `UPDATE` sekaligus mencegah baris movement tercatat, jadi riwayat append-only tidak pernah memuat percobaan gagal.
- `plugins/warehouse/services/movement.go:~100` — `updated_at` di-set eksplisit di `Updates(map[string]any{...})`: `wh_batches` tidak punya trigger `updated_at` (WO-01) dan GORM tidak mengisi kolom waktu pada update berbasis map.
- `plugins/warehouse/services/movement.go:~90` — batch tak dikenal → `ErrNotFound` (404) di dalam transaksi, dibedakan dari payload salah (400): UI harus bisa membedakan "batch ini sudah tidak ada" dari "body saya salah".
- `plugins/warehouse/services/movement.go:53` — `MovementResult{QtyRemaining}` dikembalikan bersama movement: respons 201 langsung memberi sisa stok baru sehingga UI tidak perlu round-trip kedua (dan tidak berisiko menampilkan angka basi).
- `plugins/warehouse/services/movement.go:22` — batch yang item-nya sudah dinonaktifkan **tetap boleh** di-movement (tidak ada cek `is_active` di jalur movement): stok yang sudah ada di rak tidak boleh membeku karena master berubah (SPEC §3).

### C. Read + summary

- `plugins/warehouse/services/summary.go:37` — `summarySQL` satu statement agregat (`COUNT(b.id) FILTER`, `MAX(CURRENT_DATE - entered_at)`, `MIN(expires_at)`) — bukan loop per item: SPEC §3 melarang N+1, dan total instance diturunkan dari hasil yang sudah teragregasi.
- `plugins/warehouse/services/summary.go:~40` — `oldest_age_days = MAX(CURRENT_DATE - b.entered_at)` (bukan MIN): umur batch **tertua** = selisih hari **terbesar**; MIN sempat dipakai dan langsung tertangkap test (`got 0, want 12`) karena batch baru ikut terpilih.
- `plugins/warehouse/services/summary.go:37` — `LEFT JOIN wh_batches` + `HAVING (? = FALSE OR COALESCE(SUM(qty_remaining),0) > 0)`: item tanpa batch tetap muncul (stok 0 tidak boleh tak terlihat), sementara `?only_stock=1` menyembunyikannya — satu query untuk dua mode.
- `plugins/warehouse/services/summary.go:58` — `near_expiry_batches`/`total_remaining`/`total_batches` adalah penjumlahan atas baris hasil SQL, bukan query COUNT kedua; `near_expiry_days: 30` ikut dikirim agar ambang yang dipakai server transparan ke klien (tanpa angka sihir di frontend).
- `plugins/warehouse/handlers/stock.go:16` — `withActor` menyalin `authmiddleware.Admin(c).ID` ke `created_by` bila klien tidak mengirimnya (pola hsh `entry.go:83`): tanpa FK lintas-plugin, id admin adalah satu-satunya jejak audit pada batch/movement.
- `plugins/warehouse/handlers/stock.go:39` — `ListBatches` menerima `item_id`, `warehouse_id`, `near_expiry` (badge UI), limit/offset; respons tetap memakai `newListResponse` (list kosong = `[]`) dan `writeError` sentral WO-01.
- `plugins/warehouse/handlers/stock.go:125` — `CreateMovement` memakai `writeBadPayload` untuk JSON rusak dan `writeError` untuk sentinel → 409 stok kurang / 400 payload / 404 batch asing konsisten dengan CRUD master WO-02.
- `plugins/warehouse/handlers/stock.go:149` — `StockSummary` membaca `only_stock` lewat `boolQuery` (helper WO-02) sehingga konvensi flag query bersama.
- `plugins/warehouse/plugin.go:99` — grup `/admin/wh/batches` (GET/POST collection, GET `/:id`, GET `/:id/movements`) + `/admin/wh/movements` + `/admin/wh/summary`, semuanya ber-guard `warehouse.read` (GET) / `warehouse.write` (POST): tidak ada route stok baru yang lolos RBAC.
- `plugins/warehouse/plugin.go:106` — `/admin/wh/summary` didaftarkan **sebelum** grup batch (dan tidak ada wildcard `/admin/wh/:id`) sesuai peringatan WO-03: route statis tidak boleh tertelan wildcard.

### D. Test

- `plugins/warehouse/services/stock_db_test.go:17` — fixture `newStockFixture` membuat kategori+produk+gudang nyata dan membersihkannya berurutan (movements → batches → item → kategori → gudang) lewat `t.Cleanup`, jadi tidak ada baris uji tertinggal walau test gagal di tengah.
- `plugins/warehouse/services/stock_db_test.go:92` — `TestCreateBatchRequiresLiveReferences`: 400 untuk item asing, gudang asing, `qty_in=0`, dan item yang sudah `is_active=false` (empat jalur payload).
- `plugins/warehouse/services/stock_db_test.go:132` — `TestCreateBatchDerivesAgeAndNearExpiry`: near-expiry (+5 hari) true, far (+90 hari) false, tanpa expiry false, `age_days` batch hari ini = 0.
- `plugins/warehouse/services/stock_db_test.go:162` — `TestMovementOutReducesRemaining`: OUT 3 dari 10 → `qty_remaining` **dibaca ulang dari DB** = 7 dan tepat 1 baris movement (bukan hanya cek nilai struct).
- `plugins/warehouse/services/stock_db_test.go:200` — `TestMovementOutRejectsInvalidQtyAndOverdraw`: qty 0 dan −2 → 400; qty 6 dari 5 → `ErrInsufficientStock` **plus** bukti rollback (sisa tetap 5, 0 baris movement); OUT tepat habis (5) tetap sah.
- `plugins/warehouse/services/stock_db_test.go:232` — `TestMovementAdjustIsSignedAndNeedsReason`: ADJUST tanpa reason 400, qty 0 400, minus melebihi sisa 409 + rollback, −4 → 6, +9 → 15, dan `qty` negatif tersimpan apa adanya di baris movement.
- `plugins/warehouse/services/stock_db_test.go:305` — `TestMovementsOnSameBatchAreSerialized`: 20 goroutine OUT 1 pada sisa 10 → **tepat 10 sukses / 10 `ErrInsufficientStock` / 0 error lain**, sisa akhir 0, 10 baris movement — bukti langsung `FOR UPDATE` (AC-5).
- `plugins/warehouse/services/stock_db_test.go:349` — `TestStockSummaryAggregatesInSQL`: total 8 (6−2 + 4), `batch_count` 2, `oldest_age_days` 12, `near_expiry_count` 1, `only_stock` tidak mengubah total saat masih ada stok lalu menyembunyikan item setelah dikuras.
- `plugins/warehouse/services/stock_db_test.go:434` — `TestListBatchesOldestFirstAndNearExpiryFilter`: urutan tertua dulu + `age_days` batch backdate 20, filter `near_expiry=1` menyisakan batch +3 hari, filter `warehouse_id`, 404 untuk `GetBatch`/`ListBatchMovements` id asing.
- `plugins/warehouse/services/stock_db_test.go:286` — `TestMovementWorksOnBatchOfRetiredItem`: item dinonaktifkan **setelah** batch ada → OUT tetap 201 dan sisa berkurang, mengunci keputusan "stok lama tidak beku".

### Bukti (2026-11-23, container `app`, root@admin.local)

**Build/vet/test (AC-5)**

- `go build ./... && go vet ./plugins/warehouse/...` → OK.
- `go test ./plugins/warehouse/... -v` → **10 test stok DB RUN** (bukan skip) + 8 test master + 12 test schema; `ok` tanpa FAIL.

**Status code**

| # | Request | Expect | Got |
|---|---|---|---|
| 1 | `POST /wh/batches` qty_in 10 | 201 | 201, `qty_remaining:10`, `created_by` terisi otomatis |
| 2 | `POST /wh/batches` expires +5 hari | 201 | 201, `near_expiry:true`, `age_days:0` |
| 3 | `POST /wh/movements` OUT 3 | 201 | 201, `qty_remaining:7` |
| 4 | `POST /wh/movements` OUT 999 | 409 | 409 stok tidak cukup (sisa tetap 7) |
| 5 | `POST /wh/movements` OUT qty 0 | 400 | 400 |
| 6 | `POST /wh/movements` OUT qty −2 | 400 | 400 |
| 7 | `POST /wh/movements` ADJUST −1 tanpa reason | 400 | 400 reason wajib |
| 8 | `POST /wh/movements` ADJUST qty 0 | 400 | 400 |
| 9 | `POST /wh/movements` ADJUST −20 | 409 | 409 |
| 10 | `POST /wh/movements` ADJUST −2 | 201 | 201, `qty_remaining:5` |
| 11 | `POST /wh/movements` ADJUST +4 | 201 | 201, `qty_remaining:9` |
| 12 | `POST /wh/movements` direction `IN` | 400 | 400 |
| 13 | `POST /wh/movements` batch asing | 404 | 404 |
| 14 | `GET /wh/batches?item_id=` | 200 | 200, `total:2`, tertua dulu |
| 15 | `GET /wh/batches?item_id=&near_expiry=1` | 200 | 200, `total:1` |
| 16 | `GET /wh/batches/:id/movements` | 200 | 200, `total:4`, terbaru dulu |
| 17 | `GET /wh/batches/does-not-exist` | 404 | 404 |
| 18 | `GET /wh/batches/does-not-exist/movements` | 404 | 404 |
| 19 | `GET /wh/summary` | 200 | 200, `total_remaining:10`, `total_batches:2`, `near_expiry_batches:1`, `near_expiry_days:30` |
| 20 | `POST /wh/batches` item asing | 400 | 400 |
| 21 | `POST /wh/batches` gudang asing | 400 | 400 |
| 22 | `POST /wh/batches` qty_in 0 | 400 | 400 |
| 23 | `POST /wh/batches` `entered_at` `23-11-2026` | 400 | 400 |
| 24 | `POST /wh/batches` item `is_active=false` | 400 | 400 |
| 25 | `POST /wh/movements` pada batch item non-aktif | 201 | 201 (stok lama tidak beku) |
| 26 | `GET /wh/summary` tanpa token | 401 | 401 |

**RBAC**

- viewer (`warehouse.read`): `GET /wh/summary`, `/wh/batches`, `/wh/batches/:id`, `/wh/batches/:id/movements` → **200/200/200/200**; `POST /wh/batches` → **403**; `POST /wh/movements` → **403**.
- admin (`warehouse.read`+`write`): `GET /wh/summary` → **200**; `POST /wh/batches` → **201**; `POST /wh/movements` → **201**.
- (Tidak ada endpoint stok ber-guard `warehouse.delete`; delete tetap khusus master + root, sesuai WO-02.)

**Race nyata (AC-2/AC-5)**

- Test goroutine: 20 OUT paralel pada sisa 10 → 10 sukses, 10 `409`, `qty_remaining` akhir 0, 10 baris movement; tidak ada nilai negatif dan tidak ada pelanggaran CHECK `qty_remaining >= 0`.

**Cleanup**: `wh_movements/wh_batches/wh_items/wh_categories/wh_warehouses` dihapus (0/0/0/0/0), akun uji `*@test.local` dihapus berikut baris `admin_groups`-nya, server dev dimatikan. Tanpa push; commit kode dipisah dari commit dokumen historic.


## Review Manager (2026-11-23) — LULUS ✅

Verifikasi independen Manager:

| Cek | Hasil |
|---|---|
| `git show --stat 26d5a31` | Scope bersih: hanya `plugins/warehouse/**` (6 file, +1133/−2) ✅ |
| `go test ./plugins/warehouse/... -count=1` (container) | `services` + `schema` ok (18 test total RUN) ✅ |
| Spot-check `movement.go` | OUT = qty>0; ADJUST = bertanda non-zero + `reason` wajib (baris 79/84/90); `clause.Locking{Strength:"UPDATE"}` nyata (baris 122) ✅ |
| Data uji | `wh_*` 0/0/0/0/0 bersih ✅ |
| Semua 3 commit lokal, tanpa push; AdminLayout/index.css tak disentuh | ✅ |
| Log implementasi | Ada (36 entri-baris); kualitas baik ✅ |

**Catatan review:**
1. Semantik `Movement.Qty` bertanda (lanjutan Review WO-01) **terbukti ditegakkan + ditest** — kewajiban terpenuhi.
2. Dua jebakan teknis (cast `?::int`, `MAX` umur tertua) ditemukan & diperbaiki dev sendiri dengan test — kerja bagus.
3. Keputusan item/gudang nonaktif: batch lama boleh bergerak, batch baru ditolak 400 — **disetujui Manager**, tercatat sebagai kontrak modul (sebelumnya hanya usulan di handoff).
4. Race test 20 goroutine (10 sukses/10×409) membuktikan FOR UPDATE.

**WO-03 DITERIMA. WO-04 DIBUKA — mulai sekarang.**

Jabatan: Manager Agent — Nama Tugas: Review WO-03 warehouse. Sayonara.

---
title: "WO-03: Periode & entri (snapshot)"
description: "API CRUD periode dan entri (kondisi per periode) + query ringkasan per periode."
status: complete
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - api
  - data-core
---

# WO-03: Periode & entri (snapshot)

## Referensi
- SPEC: S-3, S-7, S-8, S-12, §Skema Kanonik, §API & Permission.
- Pola acuan: `plugins/hsh` WO-03 (`services/period.go`, `services/entry.go`, summary).

## Tugas
1. **Service** `services/period.go`:
   - list/create/update/delete periode; `code` UNIQUE (format `YYYY-MM` atau `YYYY-MM-DD`);
     aturan aplikasi **≤ 1 periode aktif** (menandai aktif = menonaktifkan yang lain, 1 transaksi).
   - delete ditolak (409) bila masih punya entri.
2. **Service** `services/entry.go`:
   - list dengan filter `period_id`, `category_id`, `status_code`, `q`, pagination.
   - upsert entri per `(period_id, item_id)` — `UNIQUE` di DB menjamin idempotensi.
   - `unit_price` opsional; jika NULL → tampilkan `items.est_price` sebagai harga efektif di view.
   - `recorded_by` diisi dari identitas admin pemanggil.
   - validasi `status_code ∈ {URGENT,ENOUGH,UNKNOWN}` (400 bila invalid).
3. **Ringkasan**: `GET /admin/sec/entries/summary?period_id=` →
   `{ total, by_status: {URGENT,ENOUGH,UNKNOWN}, by_category: [ ... ] }`.
   Didaftarkan **sebelum** `/:id` (gin memprioritaskan segmen statis, pola `hsh`/`residents/stats`).
4. **Route**:
   | Method | Path | Guard |
   |---|---|---|
   | GET/POST | `/admin/sec/periods` | read / write |
   | PATCH/DELETE | `/admin/sec/periods/:id` | write / delete |
   | GET | `/admin/sec/entries/summary` | read |
   | GET/POST | `/admin/sec/entries` | read / write |
   | PATCH/DELETE | `/admin/sec/entries/:id` | write / delete |
5. **Test**: periode aktif tunggal; entri upsert idempoten (2x kirim → 1 baris); status invalid 400;
   summary benar; delete periode berisi entri → 409.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/sec/...` hijau.
- curl: buat 2 periode → aktifkan p2 (p1 auto nonaktif); buat entri (p1,itemX) 2x → 1 baris;
  summary menampilkan hitungan per status; delete periode berisi entri → 409.

## Kriteria Selesai
- Riwayat per periode tersimpan tanpa overwrite; idempotensi `(period_id,item_id)` terbukti.
- Kontrak list/summary konsisten; semua route ter-guard.

## Bukti Pelaksanaan

**File yang dibuat/diubah**
- `plugins/sec/services/period.go` (baru): `ListPeriods` (code DESC, opsi `include_inactive`),
  `GetPeriod`, `resolvePeriod` (terima **id atau code**), `CreatePeriod`, `UpdatePeriod`,
  `DeletePeriod`, helper `normalizePeriodCode` (regex `^\d{4}-(0[1-9]|1[0-2])(-\d{2})?$`),
  `parseOptionalDate`, `deactivateOtherPeriods`, `hasActivePeriod`.
  Aturan ≤1 aktif: periode pertama yang dibuat otomatis aktif bila belum ada yang aktif;
  mengaktifkan periode lain menonaktifkan sisanya **dalam transaksi yang sama**.
- `plugins/sec/services/entry.go` (baru): `EntryInput/EntryFilter/EntryView/UpsertResult`,
  `ListEntries` (filter `period_id` (id|code), `category_id` (id|code), `status`, `q` pada
  nama/merk/spesifikasi/alias; limit default 100 max 500), `UpsertEntry` (idempoten per
  `(period_id,item_id)`), `UpdateEntry`, `DeleteEntry`, `Summary`.
  Harga efektif dihitung di SQL: `COALESCE(e.unit_price, i.est_price) AS effective_price`.
  Status kosong → `UNKNOWN` (`DefaultStatus`), status invalid → `ErrInvalid` (400);
  daftar status valid dibaca dari `schema.EnumCodes("need_status")`.
- `plugins/sec/handlers/period.go` (baru): `ListPeriods`, `CreatePeriod`, `UpdatePeriod`,
  `DeletePeriod` + anotasi swagger `@Tags sec`.
- `plugins/sec/handlers/entry.go` (baru): `ListEntries`, `Summary` (`period_id` wajib dari query),
  `UpsertEntry` (**201** saat baris dibuat, **200** saat update), `UpdateEntry`, `DeleteEntry`;
  `recorded_by` diisi dari `authmiddleware.Admin(c)`.
- `plugins/sec/services/entry_db_test.go` (baru): 6 test.
- `plugins/sec/plugin.go` (diubah): 9 route baru + komentar doc disesuaikan.

**Gate statis (dijalankan di container `app`)**
```
go build ./...            -> OK
go vet ./plugins/sec/...  -> OK
gofmt -l plugins/sec      -> (kosong)
go test ./plugins/sec/... -> ok (services 0.125s, handlers, schema)
```

**Test DB benar-benar jalan (bukan skip)**
```
=== RUN   TestUpsertEntryIsIdempotent
--- PASS: TestUpsertEntryIsIdempotent (0.08s)
=== RUN   TestUpsertEntryDefaultsToUnknown
--- PASS: TestUpsertEntryDefaultsToUnknown (0.03s)
=== RUN   TestInvalidStatusRejected
--- PASS: TestInvalidStatusRejected (0.00s)
=== RUN   TestSingleActivePeriod
--- PASS: TestSingleActivePeriod (0.02s)
=== RUN   TestDeletePeriodWithEntries
--- PASS: TestDeletePeriodWithEntries (0.02s)
=== RUN   TestSummaryCounts
--- PASS: TestSummaryCounts (0.02s)
```

**Verifikasi HTTP (server di-restart dari binary baru)**
1. Buat `2026-08` (aktif otomatis karena belum ada yang aktif) lalu `2026-09` dengan
   `is_active: true` → daftar `include_inactive=1` = `[('2026-09', True), ('2026-08', False)]`
   → **≤1 periode aktif terbukti**.
2. Upsert `(2026-09, item AC Uji WO03)` dua kali:
   - kirim #1 `{"status_code":"urgent","quantity":2}` → **201**, id `01a18819-7dce-...824a`
   - kirim #2 `{"status_code":"ENOUGH","unit_price":3600000}` → **200**, **id sama**
   → idempotensi `(period_id,item_id)` terbukti; `effective_price` berpindah dari 3750000
   (`est_price`) ke 3600000 (override `unit_price`) sesuai S-7.
3. `GET /entries/summary?period_id=<p>` →
   `{"total":1,"by_status":{"ENOUGH":1,"UNKNOWN":0,"URGENT":0},"by_category":[...]}`
   → ketiga key status **selalu ada** walau nilainya 0.
4. `status_code:"LOW"` → **400**
   `{"error":"sec: data tidak valid: status_code=\"LOW\" (pilih URGENT, ENOUGH, atau UNKNOWN)"}`.
5. `DELETE /periods/<p>` saat periode masih punya 1 entri → **409**
   `{"error":"sec: masih dipakai: periode masih punya 1 entri"}`; entri tetap ada (`total 1`).
6. RBAC: tanpa token → **401**; admin viewer (`sec.viewer1@test.local`, grup `viewer`):
   `periods GET 200`, `entries GET 200`, `summary GET 200`,
   `period POST 403`, `entry POST 403`, `period DELETE 403` → fail-closed terbukti.

**Data uji dibersihkan** — kondisi DB setelah WO-03:
`sec_categories=2` (seed), `sec_items=0`, `sec_entries=0`, `sec_periods=0`, `sec_item_aliases=0`.

**Catatan temuan**
- `gin` memprioritaskan segmen statis sehingga `/entries/summary` aman didaftarkan
  berdampingan dengan `/entries` (tetap didaftarkan lebih dulu untuk kejelasan).
- `console auth change-password` hanya menerima `--id` (bukan `--email`), dan
  `delete-admin` juga `--id`; admin yang sudah soft-deleted tidak bisa di-`change-password`.
  Dicatat di memori repo.
- Login admin ada di `POST /api/admin/login` dengan field respons `access_token`.

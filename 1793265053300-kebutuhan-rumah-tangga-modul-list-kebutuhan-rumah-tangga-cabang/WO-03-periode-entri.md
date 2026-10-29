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
- SPEC: N-2, N-7, N-8, N-12, §Skema Kanonik, §API & Permission.
- Pola acuan: `plugins/resident/services/query.go` (filter + pagination), `plugins/resident/handlers` (list envelope).

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
3. **Ringkasan**: `GET /admin/hsh/entries/summary?period_id=` →
   `{ total, by_status: {URGENT,ENOUGH,UNKNOWN}, by_category: [ ... ] }`.
   Didaftarkan **sebelum** `/:id` (gin memprioritaskan segmen statis, pola `residents/stats`).
4. **Route**:
   | Method | Path | Guard |
   |---|---|---|
   | GET/POST | `/admin/hsh/periods` | read / write |
   | PATCH/DELETE | `/admin/hsh/periods/:id` | write / delete |
   | GET | `/admin/hsh/entries/summary` | read |
   | GET/POST | `/admin/hsh/entries` | read / write |
   | PATCH/DELETE | `/admin/hsh/entries/:id` | write / delete |
5. **Test**: periode aktif tunggal; entri upsert idempoten (2x kirim → 1 baris); status invalid 400;
   summary benar; delete periode berisi entri → 409.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/hsh/...` hijau.
- curl: buat 2 periode → aktifkan p2 (p1 auto nonaktif); buat entri (p1,itemX) 2x → 1 baris;
  summary menampilkan hitungan per status; delete periode berisi entri → 409.

## Kriteria Selesai
- Riwayat per periode tersimpan tanpa overwrite; idempotensi `(period_id,item_id)` terbukti.
- Kontrak list/summary konsisten; semua route ter-guard.

## Bukti Pelaksanaan
- **Commit kode:** `5645154` — `feat(hsh): periode & entri snapshot + ringkasan (WO-03)`.
  File: `plugins/hsh/services/{period,entry,entry_test,entry_db_test,service}.go`,
  `plugins/hsh/handlers/{period,entry}.go`, `plugins/hsh/plugin.go`.
- **Kualitas:** `gofmt -l` bersih, `go build ./...` hijau, `go vet ./plugins/hsh/...` bersih,
  `go test ./plugins/hsh/...` hijau — 3 test DB benar-benar jalan (bukan skip):
  `TestUpsertEntryIsIdempotent`, `TestSingleActivePeriod`, `TestCreatePeriodUniqueCode` → PASS.
- **Implementasi kunci:**
  - `periodCodePattern = ^\d{4}-(0[1-9]|1[0-2])(-\d{2})?$`; `resolvePeriod` menerima **id atau code**
    (400 bila tidak dikenal) supaya UI/CLI bisa pakai `?period_id=2026-09`.
  - Aturan ≤1 periode aktif dijaga di service (`deactivateOtherPeriods` dalam satu transaksi).
    Create tanpa `is_active` → aktif hanya jika belum ada periode aktif.
  - `entrySelect` memakai `COALESCE(e.unit_price, i.est_price) AS effective_price`.
  - `UpsertEntry` transaksional: SELECT existing → UPDATE, jika tidak ada → INSERT;
    mengembalikan `UpsertResult{Entry, Created}` → handler membalas **201** saat dibuat, **200** saat update.
  - `recorded_by` diisi otomatis dari `middleware.Admin(c).ID` bila tidak dikirim.
  - `Summary` memakai `COUNT(*) FILTER (WHERE status_code='…')` sehingga ketiga kunci status
    selalu ada (0 bila kosong); `by_category` mengikuti filter yang sama.
  - Route `GET /entries/summary` didaftarkan **sebelum** `GET /entries/:id`.
- **curl (server :8082, prefix `/api/admin/hsh`):**

  | Langkah | Hasil |
  |---|---|
  | POST `/periods` `{code:"2026-09"}` | 201 |
  | POST `/periods` `{code:"2026-10-19"}` | 201; `2026-09` auto nonaktif (`include_inactive=1` → `[('2026-10-19',True),('2026-09',False)]`) |
  | GET `/periods` (aktif saja) | `total=1` |
  | POST `/entries` (period,item) #1 | **201**, `recorded_by` terisi, `effective_price=55000` (fallback `est_price`) |
  | POST `/entries` (period,item) #2 | **200**, `id` sama, status `ENOUGH`, qty 5, `unit_price=60000`, `effective_price=60000` |
  | Hitung baris DB `hsh_entries` | **1** (idempoten terbukti) |
  | POST `/entries` status `LOW` | **400** (validasi enum) |
  | GET `/entries/summary?period_id=…` | `{total:1, by_status:{ENOUGH:1,UNKNOWN:0,URGENT:0}, by_category:[{…enough:1}]}` |
  | DELETE `/periods/:id` (masih ada entri) | **409** `"periode masih punya 1 entri"` |
  | PATCH `/entries/:id` | 200 |
  | DELETE `/entries/:id`, lalu DELETE `/periods/:id` | 200 / 200 |
  | GET `/entries` akhir | `{"items":[],"total":0}` |

- **Guard (fail-closed):** admin grup `viewer` → `GET /periods` 200, `POST /periods` **403**,
  `GET /entries/summary` 400 (periode uji sudah dihapus → validasi bisnis, guard lolos);
  tanpa token → `GET /entries` **401**. Semua 6 route periode + 5 route entri memakai
  `RequirePermission("hsh.read|write|delete")`.
- **Dampak di luar plugin:** tidak ada. Semua perubahan di `plugins/hsh/` (tidak menyentuh `internal/`).
- **Pembersihan:** data uji dihapus — `0 periode / 0 entri / 0 item` (kategori seed 7 baris tetap).

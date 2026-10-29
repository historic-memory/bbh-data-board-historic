---
title: "WO-03: Periode & entri (snapshot)"
description: "API CRUD periode dan entri (kondisi per periode) + query ringkasan per periode."
status: pending
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
3. **Ringkasan**: `GET /admin/needhousehold/entries/summary?period_id=` →
   `{ total, by_status: {URGENT,ENOUGH,UNKNOWN}, by_category: [ ... ] }`.
   Didaftarkan **sebelum** `/:id` (gin memprioritaskan segmen statis, pola `residents/stats`).
4. **Route**:
   | Method | Path | Guard |
   |---|---|---|
   | GET/POST | `/admin/needhousehold/periods` | read / write |
   | PATCH/DELETE | `/admin/needhousehold/periods/:id` | write / delete |
   | GET | `/admin/needhousehold/entries/summary` | read |
   | GET/POST | `/admin/needhousehold/entries` | read / write |
   | PATCH/DELETE | `/admin/needhousehold/entries/:id` | write / delete |
5. **Test**: periode aktif tunggal; entri upsert idempoten (2x kirim → 1 baris); status invalid 400;
   summary benar; delete periode berisi entri → 409.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/needhousehold/...` hijau.
- curl: buat 2 periode → aktifkan p2 (p1 auto nonaktif); buat entri (p1,itemX) 2x → 1 baris;
  summary menampilkan hitungan per status; delete periode berisi entri → 409.

## Kriteria Selesai
- Riwayat per periode tersimpan tanpa overwrite; idempotensi `(period_id,item_id)` terbukti.
- Kontrak list/summary konsisten; semua route ter-guard.

---
title: WO-06 — Importer kunjungan 2 fasa + idempotensi
description: "Staging visit_import_batches/rows, fase 1 validasi (dry_run), fase 2 upsert transaksional: visitor dedup E.164 → visit + status awal."
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-06 — Importer kunjungan (2 fasa + idempotensi)

**Prasyarat:** WO-05.

## Lingkup

1. Migrasi v2: `visit_import_batches` + `visit_import_rows` (kolom staging = kolom sumber + hasil parse) — daftarkan di `fields.yaml` (paritas `TestFieldsMatchMigration`).
2. `services/import.go` (pola hsh/sec):
   - **Fase 1:** staging + validasi ulang; `?dry_run=1` berhenti di sini.
   - **Fase 2 (satu transaksi):**
     a. **Dedup ekspor (D-8):** baris dgn `(phone_e164, scheduled_at)` identik dgn baris yang sudah diimpor dalam batch/DB → `skipped`.
     b. **Visitor:** cari `phone_e164` (case/normalisasi sama) → reuse; tidak ada → create `visitors`.
     c. **Visit:** create + baris history status awal (`Not started`→`SCHEDULED`, `Done`→`DONE` — D-13).
   - `scheduled_at` dikonversi GMT+8 → UTC; kosong → NULL.
3. Endpoint `POST /api/admin/visit-template/import` (guard `visit.write`, header `X-Import-Batch`) + `GET /api/admin/visit/import-batches/:id` (guard `visit.read`, respons `{batch, rows}`).
4. Test DB: impor 2x → idempoten (skip duplikat ekspor); visitor sama (HP sama) → 1 visitors 2 visits; dry_run tidak menulis master.

## Acceptance Criteria

- AC-1: curl — CSV kecil → created/skipped benar; impor ulang → count sama.
- AC-2: `.xls` 400; kosong 400; tanpa token 401.
- AC-3: fixture `api-tests/fixtures/visit-import-sample.csv`.

## Realisasi

- Migrasi versi 2: `2026_11_03_100200_create_visit_import_tables.{up,down}.sql` — `visit_import_batches` (counter + status `staged|committed|failed`) dan `visit_import_rows` (kolom sumber + hasil parse + `valid`/`action`/`reason`/`message`/`raw`). Index `(batch_id, row_no)`.
- `model/models.go`: `ImportBatch`, `ImportRow`.
- `schema/fields.yaml`: `visit_import_batches` + `visit_import_rows` (paritas migrasi dijaga `TestFieldsMatchMigration`).
- `services/import.go`: `ImportTemplate` dua fasa. Fasa 1 = staging + validasi ulang (pakai `validateRow` WO-05) dan menyimpan hasil parse (`phone_e164`, `party_size`, `scheduled_at` UTC); `dry_run` berhenti di sini. Fasa 2 = satu transaksi: dedup dalam berkas, dedup ekspor `(phone_e164, scheduled_at)` → `skipped`, visitor reuse/create via `phone_e164`, visit + baris history awal. `GetImportBatch` mengembalikan `{batch, rows}`.
- `handlers/import.go`: `POST /api/admin/visit-template/import` (guard `visit.write`, header `X-Import-Batch/Created/Skipped/Rejected`) dan `GET /api/admin/visit/import-batches/:id` (guard `visit.read`).
- `api-tests/fixtures/visit-import-sample.csv`: 4 baris (3 unik + 1 duplikat dalam berkas).

**Keputusan penting:** baris history awal memakai `effective_at` = `scheduled_at` sumber (fallback waktu impor), bukan waktu impor. Tanpa ini, status hasil impor akan mengalahkan status historis asli yang ditambahkan belakangan — bug yang sudah dijaga test urutan WO-03.

## Bukti

**Commit:** `b6c4002` — `feat(visit): importer 2 fasa + idempotensi dedup HP/waktu (WO-06)` (9 file, +1059). Migrasi v2 diterapkan (`visit applied 2026_11_03_100200_create_visit_import_tables`), `\dt visit_*` → 5 tabel.

**AC-1 (curl, port 8082, token root, fixture 4 baris):**
```
import #1: 200 committed  X-Import-Created: 3, X-Import-Skipped: 1
  row2 created (pengunjung baru; party "2-4" disimpan sebagai teks)
  row3 created, row4 created, row5 skipped (duplikat dalam berkas dengan baris 2)
import #2: 200 committed  total 4 / created 0 / skipped 4   ← idempoten
```
DB setelah import #1: 3 visitors, 3 visits, 3 history; `scheduled_at` UTC benar (`23 April 2026 09:00` → `2026-04-23T01:00:00Z`), `effective_at` = `scheduled_at`, `Not started`→`SCHEDULED`, `Done`/`Selesai`→`DONE`, `party_raw="2-4"` dengan `party_size` NULL.

**AC-2:** `filename=data.xls` → 400 ("berkas .xls (Excel lama) tidak didukung"); berkas kosong → 400 ("berkas kosong"); tanpa token → 401. `dry_run=1` → 200 `staged`, 0 created, master tidak bertambah (2 baris staging tersimpan).

**AC-3:** `GET /api/admin/visit/import-batches/:id` → `batch committed 3/1/0` dan 4 baris staging dengan `phone_e164` + `scheduled_at` hasil parse.

**Test DB** (`go test ./plugins/visit/...` hijau, suite penuh hijau):
- `TestImportTemplateIsIdempotent`: import 2x → visitors/visits/history tetap 3.
- `TestImportSameVisitorTwoVisits`: 1 HP, 2 jadwal berbeda → 1 visitor, 2 visits.
- `TestImportHistoryUsesSourceEffectiveAt`: `effective_at` = hasil `ParseVisitSchedule`, status `DONE`, `party_size` NULL + `party_raw="2-4"`.
- `TestImportDryRunDoesNotWriteMaster`: status `staged`, 0 visitor, 2 baris staging.
- `TestImportSampleFixture`: total 4 / created 3 / skipped 1, `branch_key` terisi.

Data uji dibersihkan setelah verifikasi (0 visitors/0 visits/0 staging).


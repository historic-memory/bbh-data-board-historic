---
title: WO-06 — Importer kunjungan 2 fasa + idempotensi
description: "Staging visit_import_batches/rows, fase 1 validasi (dry_run), fase 2 upsert transaksional: visitor dedup E.164 → visit + status awal."
type: work-order
status: pending
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

---
title: WO-05 — Validator template
description: "POST /api/admin/wh-template/validate"
status: pending
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-05 — Validator template

## Prasyarat

WO-04 lulus.

## Lingkup

- `services/validate.go` + `handlers/validate.go` (pola hsh/sec WO-05): `POST /api/admin/wh-template/validate` (multipart file; `?format=xlsx|csv`; `?strict=1` → 422) guard `warehouse.read`; TIDAK menulis DB.
- Aturan dari blok `validation:` di `fields.yaml`; tolak `.xls` via ekstensi/magic OLE2; `template_version` mismatch → file-level; header feedback pakai header ASLI; baris kosong diabaikan; tanggal `DD/MM/YYYY`.
- Feedback: xlsx sheet HASIL+RINGKASAN / csv + kolom `hasil_validasi`/`alasan_ditolak`. Header `X-Validate-Total/Valid/Rejected`.
- Fixture `api-tests/fixtures/wh-broken-template.csv`.

## Acceptance Criteria

- AC-1 Feedback baris benar (total/valid/rejected di header & body); strict 422.
- AC-2 `.xls` → 400; version mismatch → file-level reject; read-only terbukti (0 tulisan DB).

## Catatan Implementasi (Dev)

(disi bertahap)


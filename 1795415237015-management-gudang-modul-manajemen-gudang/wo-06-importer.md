---
title: WO-06 — Importer staging 2 fase
description: "POST /api/admin/wh-template/import + batch audit"
status: pending
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

(disi bertahap)


---
title: "WO-03 — Transaksi stok: batch masuk, movement, summary"
description: "Masuk/keluar barang + stok agregat + umur + near-expiry"
status: pending
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

(disi bertahap)


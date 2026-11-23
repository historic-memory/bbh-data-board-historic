---
title: WO-04 — Generator template Excel/CSV
description: "GET /api/admin/wh-template dari fields.yaml"
status: pending
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-04 — Generator template Excel/CSV

## Prasyarat

WO-03 lulus.

## Lingkup

- `services/template.go` + `handlers/template.go` (pola hsh/sec WO-04): `GET /api/admin/wh-template?format=xlsx|csv` guard `warehouse.read`; path flat agar tak bentrok `/wh/*`.
- Template dari `schema/fields.yaml` — TANPA hard-code daftar kolom. Sheet PETUNJUK/DATA/REFERENSI(hidden)/CONTOH. Kolom DATA: Kategori, Produk, Gudang, Qty, Satuan, Tanggal Masuk, Kadaluarsa, Catatan (final di WO-06 impor). Dropdown REFERENSI untuk Kategori (data DB), Satuan, Gudang (data DB).
- CSV = BOM + header. `template_version` di sheet PETUNJUK.
- G-4 terjawab via EX-1 (owner "sertakan excel") — masuk kolom Kadaluarsa.

## Acceptance Criteria

- AC-1 xlsx 200 valid (inspeksi dropdown `REFERENSI!$X$2:$X$n`), csv 200 BOM; `format=pdf` → 400; tanpa token 401; viewer 200.
- AC-2 Unit test + Bruno template.
- AC-3 Gotcha: excelize `DataValidation.Sqref` bertipe string (split `:`).

## Catatan Implementasi (Dev)

(disi bertahap)


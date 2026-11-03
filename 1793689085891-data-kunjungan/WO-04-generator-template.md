---
title: WO-04 — Generator template impor kunjungan
description: Endpoint GET /api/admin/visit-template (xlsx/csv) digenerate dari fields.yaml; header = kolom sumber Notion agar round-trip mulus.
type: work-order
status: pending
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-04 — Generator template impor kunjungan

**Prasyarat:** WO-02.

## Lingkup

1. `schema/fields.yaml` → blok `template:` dengan kolom **header Indonesia sesuai CSV sumber**: `Nama`, `Alamat`, `Jumlah Pengunjung`, `Nomor HP`, `Status Kunjungan`, `Waktu Kunjungan` (D-9: round-trip Notion→sistem mulus).
2. `services/template.go` (pola hsh/sec): `GenerateTemplate` xlsx (sheet PETUNJUK/DATA/REFERENSI(hidden)/CONTOH) + csv (BOM+header). Dropdown REFERENSI: Status Kunjungan (5 enum).
3. Route flat `GET /api/admin/visit-template?format=xlsx|csv` (guard `visit.read`) — didaftarkan SEBELUM wildcard `/:id`.
4. `Jumlah Pengunjung` bebas teks (tidak ada dropdown) — hanya instruksi contoh format `2-4` di sheet PETUNJUK.

## Acceptance Criteria

- AC-1: Unit test — xlsx valid (4 sheet, dropdown dari REFERENSI), csv BOM+header, `format=pdf` → 400, tanpa token 401.
- AC-2: curl — xlsx 200 ukuran wajar & bisa dibuka; csv 200.
- AC-3: `template_version` tercantum di PETUNJUK (dipakai validator WO-05).

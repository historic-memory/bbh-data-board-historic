---
title: "WO-09: Swagger + README + koleksi Bruno"
description: "Dokumentasi API sec (swagger), README plugin, koleksi Bruno, dan verifikasi paritas."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - docs
  - swagger
  - bruno
---

# WO-09: Swagger + README + koleksi Bruno

## Referensi
- SPEC: §API & Permission.
- Pola acuan: `plugins/hsh` WO-09; anotasi swag `plugins/resident` (import paket `model` + blank var
  agar `swag` resolve tipe tanpa `cannot find type definition`).

## Tugas
1. **Anotasi swagger** pada `handlers/sec` untuk seluruh route (categories, items+alias, periods,
   entries+summary, `sec-template` + `/validate` + `/import`, `import-batches/:id`).
2. **Regenerasi** swagger: `./build.sh --verify-docs` (container `app`, `--parseDependency --parseInternal`).
3. **README** plugin `sec` — bagian "Plugin `sec` — Kebutuhan Sekunder" (migrasi, unduh template,
   impor, Bruno); jelaskan batas domain vs `hsh`.
4. **Koleksi Bruno** `api-tests/secondary/` (14 request) + smoke read-only + fixture
   `api-tests/fixtures/sec-valid-template.csv` (sudah ada `sec-broken-template.csv` dari WO-05).
   Tambah tag `secondary`; perbarui `bruno.json` name & README koleksi.
5. **Verifikasi paritas**: jumlah path & operasi `sec` di swagger cocok dengan route di `plugin.go`;
   regen byte-identik.

## Bukti yang harus dilampirkan
- `build.sh` exit 0; swagger memuat path `sec` (jumlah path/opsi/definisi dilaporkan).
- `bru run --env Local --tags=smoke` PASS; folder `secondary` PASS.
- README plugin tertulis.

## Kriteria Selesai
- Swagger/README/Bruno konsisten dengan implementasi; paritas dua arah OK.
- Tidak ada perubahan kode fitur selain anotasi/docs.

## Bukti Pelaksanaan
_(diisi saat WO dikerjakan)_

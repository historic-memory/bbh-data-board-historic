---
title: "WO-09: Swagger, README & koleksi Bruno"
description: "Dokumentasi API hsh (swagger), panduan di README, dan koleksi Bruno untuk alur utama."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - docs
  - swagger
  - bruno
---

# WO-09: Swagger, README & koleksi Bruno

## Referensi
- SPEC: §API & Permission, §Kontrak Template.
- Pola acuan: `plugins/resident/handlers` (anotasi swag + blank var import), `build.sh`
  (`--parseDependency --parseInternal`), `api-tests/` (Bruno), README plugin `resident`.

## Tugas
1. **Anotasi swag** pada handler hsh; daftarkan tipe via blank var import bila perlu
   (pola `resident` agar `swag` resolve `model.*`).
2. **Regenerate** `docs/swagger` di container kanonik (`app`) & pastikan `build.sh` hijau.
3. **README** — bagian "Plugin `hsh` — Kebutuhan Rumah Tangga": migrasi, permission,
   unduh template, alur impor, catatan `.xlsx` saja, UI.
4. **Koleksi Bruno** `api-tests/`:
   - `needs/` kategori & item (create/list/get/patch/alias) — mutasi butuh approval per-run.
   - `needs/` periode & entri, template unduh, validate, import (fixture).
   - `smoke/` tambahan read-only (list kategori/item).
   - Fixture template valid + rusak di `api-tests/fixtures/`.
5. **Paritas**: hitung paths/ops hsh di swagger; pastikan konsisten.

## Bukti yang harus dilampirkan
- `docs/swagger` teregen; paritas path/ops tercatat (angka) & `build.sh` exit 0.
- `bru run --env Local --tags=smoke` hijau; koleksi `needs` (mutasi) hijau saat dijalankan.
- Cuplikan README + daftar request Bruno.

## Kriteria Selesai
- Dokumentasi API lengkap & konsisten dengan implementasi.
- Bruno mencakup alur utama (master → periode → entri → template → impor).
- Tidak ada rahasia/PII di repo; `.env` tetap gitignored.

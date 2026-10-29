---
title: "WO-09: Swagger + README + koleksi Bruno"
description: "Dokumentasi API sec (swagger), README plugin, koleksi Bruno, dan verifikasi paritas."
status: complete
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

**Anotasi swagger**
Seluruh 26 operasi (15 path) sudah dianotasi pada `plugins/sec/handlers/*.go`
(`@Tags sec`, `@Router /api/admin/sec… [method]`, `@Security BearerAuth`) sejak
WO-02..WO-06 diterapkan; WO-09 hanya memverifikasi kelengkapannya — tidak ada
handler tanpa `@Router`: 13 (`handlers.go`) + 5 (`entry.go`) + 4 (`period.go`) +
1 (`template.go`) + 1 (`validate.go`) + 2 (`import.go`) = **26**.

**Regenerasi swagger**
```bash
docker compose exec -T app bash build.sh --verify-docs
```
- `BUILD OK: biner + dokumentasi swagger diperbarui` (exit 0).
- `docs/swagger/swagger.json`: **15 path** `/api/admin/sec*`, **26 operasi** —
  cocok 26/26 dengan route di `plugins/sec/plugin.go`. Total koleksi API:
  **68 path / 112 operasi / 98 definisi** (sebelumnya 86 operasi → +26 `sec`).
- Definisi baru: `model.ImportBatch`, `model.ImportRow`,
  `services.ImportBatchReport`, `services.ImportResult`, `services.ImportOptions`,
  `services.PeriodInput`, `handlers.ErrorResponse`.
- Regen **byte-identik**: run kedua menghasilkan manifest sha256 sama
  (`docs.go 9d6ecf8f…`, `swagger.json f6bee18c…`, `swagger.yaml db4ce693…`).
  `docs/` gitignored → tidak ada artefak yang di-commit.

**README**
- `README.md` — seksi baru **"Plugin `sec` — Kebutuhan Sekunder"** (118 baris):
  tabel pembeda domain vs `hsh` (cakupan, kolom `brand`/`specification`, 8 vs 12
  satuan, prefix route, prefix UI), skema 7 tabel, migrasi
  (`migrate --plugin sec up|list`), permission
  (`2026_10_29_130200_add_sec_permissions`: admin read+write, viewer read,
  root `*`), daftar route, template/impor (termasuk `COALESCE` brand, kategori
  tidak auto-create, periode non-aktif), tabel route UI, kontrak pengujian.
- `api-tests/README.md` — judul koleksi jadi tiga plugin, struktur folder
  menambah `secondary/`, tabel urutan `secondary/` (14 langkah), catatan fixture
  `sec-*`, tabel tag diperbarui (`sec` categories/items ikut `smoke`).

**Koleksi Bruno**
- BARU `api-tests/secondary/` — 14 request (`01`–`14`) mengikuti pola `needs/`
  dengan tambahan domain: item menguji `brand` + `specification`, `PATCH` mengubah
  `brand`, dan langkah `14` memastikan baris staging menyimpan keduanya.
- BARU `api-tests/smoke/07-List-sec-categories.bru`,
  `api-tests/smoke/08-List-sec-items.bru` (read-only, tag `smoke`; `08` toleran
  terhadap `omitempty` brand/spesifikasi).
- BARU `api-tests/fixtures/sec-valid-template.csv` (2 baris valid, periode
  `2030-02`).
- `api-tests/bruno.json` → `name: "BBH Admin API (resident + hsh + sec)"`.

**Hasil `bru run`**
```bash
bru run --env Local --tags=smoke   # Requests 8/8,  Tests 21/21 → PASS
bru run --env Local secondary      # Requests 14/14, Tests 37/37 → PASS
```
Assertion penting: `12-Validate-template` membaca `X-Validate-Total=9 /
Valid=2 / Rejected=7`; `13-Import-template` (`?dry_run=1`) melaporkan
`status=staged`, `summary.total=2`, `created=0`; `14-Get-import-batch`
memastikan `rows[0].brand="Cosmos"` dan
`rows[0].specification="16 inch 3 speed"`.

Data uji hasil run Bruno dibersihkan setelah verifikasi: `sec_*` kembali
2 kategori (seed) / 0 item / 0 periode / 0 entri / 0 batch staging.

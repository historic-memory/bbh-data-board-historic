---
title: "Work Order (Master Index): Kebutuhan Sekunder"
description: "Rencana implementasi plugin sec: skema kanonik, master kategori/item, periode & entri snapshot, template Excel ter-generate, validator, importer, UI admin, dan dokumentasi. Kembar dari plugin hsh."
status: draft
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - data-core
  - schema
  - import
  - template
  - admin-panel
---

# Work Order (Master Index): Kebutuhan Sekunder

## Referensi
- **SPEC topik ini:** `spec.md` (S-1..S-19, GAP-1..GAP-9, O-1..O-8).
- **Aset mentah:** `List Kebutuhan Elektronik BBH - Updated 22 SEPT 26 2026.xlsx` + analisis di `notes.md`.
- **Kembar struktur:** plugin `hsh` (`1793265053300`) — bentuk dokumen & implementasi identik, beda prefix.
- **Pola acuan (wajib ditiru):** `plugins/resident` — `schema/fields.yaml` sebagai *single source of
  truth*, generator template, validator umpan-balik, importer 2 fasa, permission di-seed via migrasi plugin `auth`.
- **Aturan user:** fitur baru HARUS plugin di `plugins/<id>/`; `internal/` hanya framework/bootstrap.

## Prinsip Pelaksanaan
1. **Plugin baru** `plugins/sec`; seluruh kode fitur di dalamnya. Jangan menyentuh `internal/`.
2. **Prefix tabel wajib** `sec_*` (satu DB dengan `hsh_*`).
3. **Tanpa dimensi cabang** (S-2); batas data = instance (`INSTANCE_KEY`).
4. **Kategori = tabel**, **satuan = daftar REFERENSI statis** (S-4/S-16), **status = enum statis**
   `URGENT|ENOUGH|UNKNOWN` + `CHECK` (S-8); data mentah tanpa legend → default `UNKNOWN`.
5. **Nama item apa adanya** + `brand` + `specification`; normalisasi/dedup lewat **alias** (S-5/S-19).
6. **Template digenerate** dari `fields.yaml`, tidak di-maintain manual (S-11).
7. **Importer 2 fasa** (staging → upsert), idempoten (S-12); **impor hanya `.xlsx`/`.csv`** (S-18).
8. **Permission** `sec.*` di-seed via migrasi plugin `auth` — kalau lupa, semua route 403.
9. ID = **UUID v7** dibuat di Go (`model.NewID`), `TEXT`, tanpa `DEFAULT` DB (S-10).
10. Setiap WO berakhir: `historic rebuild` → `historic save` → `historic git commit` (**tanpa push**).

## Daftar WO
| # | File | Scope | Prasyarat | Status |
|---|---|---|---|---|
| 1 | `WO-01-skeleton-plugin-skema.md` | Skeleton `plugins/sec`, `schema/fields.yaml`, migrasi 5 tabel + enum, seed 2 kategori | — | complete |
| 2 | `WO-02-permission-crud-master.md` | Seed permission `sec.*` di migrasi `auth`; CRUD kategori/item/alias + test | WO-01 | complete |
| 3 | `WO-03-periode-entri.md` | CRUD periode + entri + query per periode (ringkasan status) | WO-02 | complete |
| 4 | `WO-04-generator-template.md` | `GET /admin/sec-template?format=xlsx\|csv` dari `fields.yaml` + dropdown REFERENSI | WO-01 | complete |
| 5 | `WO-05-validator.md` | `POST /admin/sec-template/validate` (read-only, umpan-balik per baris) | WO-04 | complete |
| 6 | `WO-06-importer.md` | `POST .../sec-template/import` staging → upsert, alias-first, auto-create (S-19), tolak `.xls` | WO-05 | pending |
| 7 | `WO-07-seed-data-awal.md` | **_DITUNDA_** — impor berkas "Updated 22 SEPT 26" + laporan normalisasi (GAP-2/7) | WO-06 | pending |
| 8 | `WO-08-ui-admin.md` | UI: Kategori, Item (+alias), Periode, Entri (matriks), Unduh/Impor | WO-03, WO-06 | pending |
| 9 | `WO-09-swagger-docs-bruno.md` | Swagger + README + koleksi Bruno + paritas | WO-08 | pending |

## Definisi Selesai (topik)
- Plugin `sec` terpasang, migrasi jalan bersih dari DB kosong, 5 tabel + enum + 2 kategori.
- Permission ter-seed; route read/write/delete fail-closed sesuai guard.
- Template XLSX/CSV ter-generate dari `fields.yaml` (dropdown kategori/satuan/status; kolom Periode).
- Validator & importer berjalan; `.xls` ditolak dengan pesan pra-konversi; impor idempoten.
- UI admin: master item, periode, dan entri (matriks) berfungsi.
- Swagger + README + Bruno konsisten; `build.sh` hijau.
- **Seed data awal tetap ditunda** (WO-07) sampai diminta owner.

## Catatan
- Topik **draft** — daftar WO dapat berubah setelah pelaksanaan.
- Urutan: **WO-01 → 02 → 03 → 04 → 05 → 06 → (07 ditunda) → 08 → 09**.
- Topik kembar `hsh` (`1793265053300`) = **habis pakai/pokok**; `sec` = **barang tahan lama**
  (elektronik, peralatan, mainan). Batas domain ini dijaga konsisten di kedua plugin.

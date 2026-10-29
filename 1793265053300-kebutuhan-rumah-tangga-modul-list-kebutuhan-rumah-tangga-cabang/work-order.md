---
title: "Work Order (Master Index): Kebutuhan Rumah Tangga"
description: "Rencana implementasi plugin hsh: skema kanonik, master kategori/item, periode & entri snapshot, template Excel ter-generate, validator, importer, UI admin, dan dokumentasi."
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

# Work Order (Master Index): Kebutuhan Rumah Tangga

## Referensi
- **SPEC topik ini:** `spec.md` (N-1..N-18, GAP-1..GAP-9, O-1..O-7 terkuncil).
- **Aset mentah:** `List Kebutuhan RT  BBH  SEPT 19 OKT 26 d.xls` + analisis di `notes.md`.
- **Pola acuan (wajib ditiru):** `plugins/resident` — `schema/fields.yaml` sebagai *single source of
  truth*, generator template (WO-08), validator umpan-balik (WO-09), importer 2 fasa (WO-06),
  permission di-seed via migrasi plugin `auth`.
- **Aturan user:** fitur baru HARUS plugin di `plugins/<id>/`; `internal/` hanya framework/bootstrap.

## Prinsip Pelaksanaan
1. **Plugin baru** `plugins/hsh`; seluruh kode fitur di dalamnya. Jangan menyentuh `internal/`.
2. **Prefix tabel wajib** `hsh_*` (satu DB dengan `elec_*` nanti).
3. **Tanpa dimensi cabang** (N-1); batas data = instance (`INSTANCE_KEY`).
4. **Kategori = tabel**, **satuan = daftar REFERENSI statis** (N-3/N-13), **status = enum statis**
   `URGENT|ENOUGH|UNKNOWN` + `CHECK` (N-8).
5. **Nama item apa adanya**; normalisasi/dedup lewat **alias** (N-4/N-16).
6. **Template digenerate** dari `fields.yaml`, tidak di-maintain manual (N-11).
7. **Importer 2 fasa** (staging → upsert), idempoten (N-12); **impor hanya `.xlsx`/`.csv`** (N-17).
8. **Permission** `hsh.*` di-seed via migrasi plugin `auth` — kalau lupa, semua route 403.
9. ID = **UUID v7** dibuat di Go (`model.NewID`), `TEXT`, tanpa `DEFAULT` DB (N-10).
10. Setiap WO berakhir: `historic rebuild` → `historic save` → `historic git commit` (**tanpa push**).

## Daftar WO
| # | File | Scope | Prasyarat | Status |
|---|---|---|---|---|
| 1 | `WO-01-skeleton-plugin-skema.md` | Skeleton `plugins/hsh`, `schema/fields.yaml`, migrasi 5 tabel + enum, seed 7 kategori (referensi) | — | **complete** (`cf6367f`) |
| 2 | `WO-02-permission-crud-master.md` | Seed permission `hsh.*` di migrasi `auth`; CRUD kategori/item/alias + test | WO-01 | **complete** (`8128072`) |
| 3 | `WO-03-periode-entri.md` | CRUD periode + entri + query per periode (ringkasan urgen/enough) | WO-02 | **complete** (`5645154`) |
| 4 | `WO-04-generator-template.md` | `GET /admin/hsh-template?format=xlsx\|csv` dari `fields.yaml` + dropdown REFERENSI | WO-01 | **complete** (`18eaf8f`) |
| 5 | `WO-05-validator.md` | `POST /admin/hsh-template/validate` (read-only, umpan-balik per baris) | WO-04 | **complete** (`7c85b71`) |
| 6 | `WO-06-importer.md` | `POST .../template/import` staging → upsert, alias-first, auto-create (N-16), tolak `.xls` | WO-05 | **complete** (`532eb81`) |
| 7 | `WO-07-seed-data-awal.md` | **_DITUNDA (N-15)_** — impor berkas SEPT–OKT + laporan normalisasi (GAP-2/7/8) | WO-06 | pending |
| 8 | `WO-08-ui-admin.md` | UI: Kategori, Item (+alias), Periode, Entri (matriks), Unduh/Impor | WO-03, WO-06 | **complete** (`d554950`) |
| 9 | `WO-09-swagger-docs-bruno.md` | Swagger + README + koleksi Bruno + paritas | WO-08 | **complete** (`37e7e1f`) |

## Definisi Selesai (topik)
- Plugin `hsh` terpasang, migrasi jalan bersih dari DB kosong, 5 tabel + enum + 7 kategori.
- Permission ter-seed; route read/write/delete fail-closed sesuai guard.
- Template XLSX/CSV ter-generate dari `fields.yaml` (dropdown kategori/satuan/status; kolom Periode).
- Validator & importer berjalan; `.xls` ditolak dengan pesan pra-konversi; impor idempoten.
- UI admin: master item, periode, dan entri (matriks) berfungsi.
- Swagger + README + Bruno konsisten; `build.sh` hijau.
- **Seed data awal tetap ditunda** (WO-07) sampai diminta owner.

## Catatan
- Topik **draft** — daftar WO dapat berubah setelah pelaksanaan.
- Urutan: **WO-01 → 02 → 03 → 04 → 05 → 06 → (07 ditunda) → 08 → 09**.
- Topik kembar `elec` (`1793265053246`) **ditunda** sampai data mentah elektronik ada;
  plugin itu menyalin bentuk ini dengan prefix `elec_*`.

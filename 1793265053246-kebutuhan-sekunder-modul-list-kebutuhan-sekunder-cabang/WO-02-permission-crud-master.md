---
title: "WO-02: Permission & CRUD master (kategori/item/alias)"
description: "Seed permission sec.read/write/delete via migrasi plugin auth; API + service CRUD kategori, item, dan alias + test."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - api
  - rbac
---

# WO-02: Permission & CRUD master (kategori/item/alias)

## Referensi
- SPEC: S-4, S-5, S-6, S-9, S-19, §API & Permission.
- Pola acuan: `plugins/resident` (RBAC fail-closed), `plugins/hsh` (WO-02).

## Tugas
1. **Seed permission** di migrasi **plugin `auth`** (`sec.read`, `sec.write`, `sec.delete`):
   - grup `admin` → read+write; grup `viewer` → read; root via `*`.
   - **Wajib** — kalau lupa, semua route `sec` 403 (fail-closed).
2. **Service** `services/service.go`:
   - `CreateCategory/UpdateCategory/DeleteCategory/ListCategories`; `code` slug
     `^[a-z0-9][a-z0-9_-]*$` UNIQUE.
   - `CreateItem/UpdateItem/DeleteItem/ListItem` (filter `category_id`, `q`, pagination).
   - `AddAlias/RemoveAlias/ListAliases`; alias UNIQUE global case-insensitive → 409 bila bentrok.
   - `unit` wajib dari daftar REFERENSI `fields.yaml` (case-insensitive, disimpan huruf kecil).
   - Hapus kategori/item yang masih dipakai → **409 `ErrInUse`**.
3. **Handler** `handlers/handlers.go`: kontrak list `{items:[], total}`; map error → status (400/404/409).
4. **Route** (grup `api` sudah ber-prefix `/api`, path relatif):
   | Method | Path | Guard |
   |---|---|---|
   | GET/POST | `/admin/sec/categories` | read / write |
   | PATCH/DELETE | `/admin/sec/categories/:id` | write / delete |
   | GET/POST | `/admin/sec/items` | read / write |
   | GET/PUT/PATCH/DELETE | `/admin/sec/items/:id` | read / write / write / delete |
   | GET/POST | `/admin/sec/items/:id/aliases` | read / write |
   | DELETE | `/admin/sec/items/:id/aliases/:aliasId` | write |
5. **Test**: validasi slug/unit; alias unik; delete in-use → 409; guard read-only (viewer POST → 403).

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/sec/...` hijau.
- `console auth permission list | grep sec` → 3 baris.
- curl CRUD kategori/item + alias; 409 in-use; 403 viewer / 401 tanpa token.

## Kriteria Selesai
- Permission ter-seed; semua route ter-guard dan fail-closed.
- Aturan validasi (slug, unit, alias unik, in-use) terbukti.

## Bukti Pelaksanaan
_(diisi saat WO dikerjakan)_

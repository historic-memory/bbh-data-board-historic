---
title: "WO-02: Permission & CRUD master (kategori/item/alias)"
description: "Seed permission sec.read/write/delete via migrasi plugin auth; API + service CRUD kategori, item, dan alias + test."
status: complete
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
**Status: `complete`** (2026-10-29). Kode commit `1fddd1c`.

### Berkas
**Baru**
- `plugins/auth/migrations/postgres/2026_10_29_130200_add_sec_permissions.{up,down}.sql` — seed `sec.read/write/delete`; `admin` = read+write, `viewer` = read, `root` via `*`. Down menghapus 3 permission (cascade `group_permissions`).
- `plugins/sec/services/{category,item,alias}.go` — CRUD lengkap. `ItemInput/ItemView` memuat `brand` + `specification` (S-6); pencarian `q` mencakup nama, brand, spesifikasi, dan alias.
- `plugins/sec/handlers/errors.go` — envelope `{error}`, `newListResponse` (kosong → `[]`), pemetaan sentinel → 400/404/409/500.
- `plugins/sec/services/validation_test.go` (4 test murni), `plugins/sec/services/catalog_db_test.go` (4 test DB), `plugins/sec/handlers/errors_test.go` (2 test).

**Diubah**
- `plugins/sec/services/service.go` — `ErrUnknownUnit`, `normalizeUnit` (case-insensitive → huruf kecil), `normalizeCode` (slug `^[a-z0-9][a-z0-9_-]*$`), `isForeignKeyViolation`.
- `plugins/sec/handlers/handlers.go` — 13 handler (kategori 4, item 6, alias 3) + anotasi swagger `@Tags sec`.
- `plugins/sec/plugin.go` — 13 route di bawah `/api/admin/sec/...` dengan guard `sec.read`/`sec.write`/`sec.delete`.

### Hasil verifikasi
- `go build ./...` hijau; `go vet ./plugins/sec/...` bersih; `gofmt -l plugins/sec` kosong.
- `go test ./plugins/sec/...` hijau, 4 test DB benar-benar RUN (bukan skip): `TestCreateHonoursInactiveFlag` PASS, `TestCreateDefaultsToActive` PASS, `TestAliasUniqueGlobally` PASS, `TestDeleteCategoryInUse` PASS.
- `migrate --plugin auth up` → `auth applied 2026_10_29_130200_add_sec_permissions.up.sql`; `SELECT key FROM permissions WHERE key LIKE 'sec.%'` → 3 baris; `group_permissions` → admin(read,write) + viewer(read).
- curl (server `:8082`):
  - tanpa token → **401**; `GET /api/admin/sec/categories` → 200 `{items:[2 seed],total:2}`.
  - `POST /api/admin/sec/categories` → **201**; item dengan `brand`+`specification`+`unit:"UNIT"` → **201** dan `unit` tersimpan `"unit"` (ternormalisasi).
  - item duplikat nama dalam kategori sama (`ac midea 1pk`) → **409**.
  - `unit:"karung"` → **400** `sec: satuan tidak dikenal`.
  - alias `AC Midea` → 201; alias sama (huruf kecil) pada item lain → **409** `uq_sec_item_aliases_alias`.
  - `PATCH` item `{"brand":"","unit":"set"}` → 200, `brand` kembali `null` (pengosongan didukung), `unit` → `set`.
  - `DELETE` kategori yang masih punya 2 item → **409** `kategori masih dipakai 2 item`.
- Fail-closed RBAC: admin `sec.viewer1@test.local` (grup `viewer`) → `GET` **200**, `POST` **403** (`butuh sec.write`), `DELETE` **403**.
- Data uji dibersihkan: item/alias/kategori uji dihapus, akun viewer dihapus; sisa DB `sec_categories` = 2 baris seed, `sec_items`/`sec_item_aliases` = 0.

### Catatan
- `console auth create-admin` **wajib** menyertakan `--name` (kalau tidak: `--name and --email are required`).
- Kategori **tidak** auto-create (berbeda dari item, S-19); `category_id` tak dikenal → 400.
- Guard delete kategori/item memakai `sec.delete`; alias cukup `sec.write` (tanpa permission khusus).

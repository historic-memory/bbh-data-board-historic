---
title: "WO-02: Permission & CRUD master (kategori, item, alias)"
description: "Seed permission hsh.* via migrasi plugin auth, lalu API CRUD kategori, item, dan alias dengan service + test."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - api
  - rbac
---

# WO-02: Permission & CRUD master (kategori, item, alias)

## Referensi
- SPEC: N-4, N-9, N-16, §API & Permission.
- Pola acuan: `plugins/resident/plugin.go` (guard `middleware.RequirePermission`),
  `plugins/auth/migrations/` (seed permission — pola `resident.read/write/delete`),
  `plugins/resident/handlers/handlers.go` (list kosong `[]` bukan `null`).

## Tugas
1. **Seed permission** di migrasi plugin `auth` (bukan migrasi plugin hsh):
   `hsh.read`, `hsh.write`, `hsh.delete`; daftarkan ke grup sistem
   (`admin`=read+write, `viewer`=read) mengikuti pola `resident` supaya tidak fail-closed.
2. **Service** `services/category.go`, `services/item.go`, `services/alias.go`:
   - Category: list/create/update/delete — `code` unik, `delete` ditolak bila masih dipakai item (409).
   - Item: list (filter `category_id`, `q`, pagination), get, create, put/patch, delete;
     normalisasi `unit` terhadap daftar REFERENSI; `est_price` numeric.
   - Alias: list/create/delete per item; `UNIQUE (lower(alias))` → bentrok 409.
3. **Handler** + route (grup `api` sudah prefix `/api`, path relatif):
   | Method | Path | Guard |
   |---|---|---|
   | GET/POST | `/admin/hsh/categories` | read / write |
   | PATCH/DELETE | `/admin/hsh/categories/:id` | write / delete |
   | GET/POST | `/admin/hsh/items` | read / write |
   | GET/PUT/PATCH/DELETE | `/admin/hsh/items/:id` | read / write / write / delete |
   | GET/POST | `/admin/hsh/items/:id/aliases` | read / write |
   | DELETE | `/admin/hsh/items/:id/aliases/:aliasId` | write |
4. **Kontrak list**: selalu `{ items: [], total }` (bukan `null`).
5. **Test** unit + (bila ada) DB test untuk: create/list/get/patch, 409 kategori terpakai,
   409 alias bentrok, 400 enum/format invalid, guard 403 untuk viewer POST.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/hsh/...` hijau.
- Migrasi auth ter-seed: `SELECT key FROM permissions WHERE key LIKE 'hsh.%'` → 3 baris;
  `auth group show admin/viewer` memuat permission tersebut.
- curl: create kategori 201, create item 201, list 200 (kosong → `[]`), alias create 201,
  duplikat alias → 409, viewer POST → 403, tanpa token → 401.

## Kriteria Selesai
- Semua route ter-guard; fail-closed; kontrak API sesuai SPEC.
- Tidak ada perubahan pada `internal/`; perubahan plugin `auth` hanya berupa migrasi seed.

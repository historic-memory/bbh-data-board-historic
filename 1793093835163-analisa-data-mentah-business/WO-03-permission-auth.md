---
title: "WO-03: Permission resident.* di plugin auth"
description: "Migrasi tambahan di plugin `auth` untuk seed permission resident.read / resident.write / resident.delete dan memetakannya ke grup sistem root, admin, viewer. Prasyarat wajib sebelum route ber-guard berfungsi (fail-closed)."
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-03: Permission `resident.*` di plugin `auth`

**SPEC acuan:** `SPEC.md` §14 (jebakan permission), AC-12
**Prasyarat:** WO-02

## Latar
Model RBAC (WO-04 topic federasi) bersifat **fail-closed**: admin tanpa permission akan **403** di setiap route ber-guard `RequirePermission`. Permission belum ada → route WO-04 tidak akan pernah bisa diuji.

Permission di-seed **lewat migrasi plugin `auth`**, bukan bootstrap Go, mengikuti pola `2026_10_27_100000_create_rbac_tables.up.sql`.

## Cakupan

Buat migrasi baru di plugin `auth`:
```
plugins/auth/migrations/postgres/2026_10_27_130000_add_resident_permissions.up.sql
plugins/auth/migrations/postgres/2026_10_27_130000_add_resident_permissions.down.sql
```

### `.up.sql`
1. `INSERT INTO permissions (key, name)` → `resident.read`, `resident.write`, `resident.delete` (idempoten: pakai `ON CONFLICT (key) DO NOTHING`).
2. Petakan ke grup sistem:
   | Grup | Permission |
   |---|---|
   | `root` | sudah memegang `*` — pastikan **tidak** ada perubahan (jangan tambah baris duplikat) |
   | `admin` | `resident.read`, `resident.write` (tanpa `delete`) |
   | `viewer` | `resident.read` |
3. Gunakan **subquery berbasis `key`** untuk `group_id`/`permission_id`, jangan hard-code UUID.

### `.down.sql`
Hapus baris `group_permissions` untuk ketiga key, lalu hapus baris `permissions` (idempoten).

## Aturan
- Migrasi **tidak boleh** menyentuh tabel `admins`/`groups` selain menambah baris `group_permissions`.
- Key permission harus **stabil** (`resident.read` dst.) karena dirujuk kode di WO-04 & WO-05.
- Tidak mengubah permission existing.

## Bukti yang Harus Dilaporkan
- `./bin/console migrate up --plugin auth` pada DB yang sudah punya RBAC → menambah 3 permission.
- `SELECT key FROM permissions WHERE key LIKE 'resident.%'` → 3 baris.
- `SELECT g.name, p.key FROM group_permissions gp JOIN groups g ON ... JOIN permissions p ON ... WHERE p.key LIKE 'resident.%'` → `root`(via `*` tidak tampil sebagai baris — jelaskan), `admin` 2 baris, `viewer` 1 baris.
- Migrasi dijalankan **dua kali** → tidak ada duplikat (idempoten).
- `./bin/console migrate down --plugin auth` versi ini → 3 permission hilang.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | 3 permission `resident.read/write/delete` ter-seed. |
| AC-2 | `admin` mendapat `read`+`write` (bukan `delete`); `viewer` mendapat `read`. |
| AC-3 | Grup `root` tetap memegang `*` (tidak ada perubahan perilaku). |
| AC-4 | Migrasi **idempoten** (dijalankan 2x aman). |
| AC-5 | `.down.sql` membersihkan baris yang ditambahkan, tanpa menyentuh data lain. |
| AC-6 | Tidak ada `INSERT` di kode Go. |
| AC-7 | `go build`/`vet`/`test`/`gofmt` lulus. |
| AC-8 | Dibuktikan: admin grup `viewer` bisa **GET**, tapi **POST** → 403 (uji minimal via curl setelah WO-04). |

## Status Eksekusi (complete)
**Status: complete** — commit kode `e464765` (`feat(auth): WO-03 seed resident.read/write/delete permissions`). Tidak ada push.

Artefak: `plugins/auth/migrations/postgres/2026_10_27_130000_add_resident_permissions.{up,down}.sql`.

| # | AC | Hasil |
|---|---|---|
| AC-1 | 3 permission ter-seed | ✅ `resident.read/write/delete` (3 baris) |
| AC-2 | admin read+write; viewer read | ✅ mapping = admin→read,write; viewer→read (3 baris `group_permissions`) |
| AC-3 | root tetap `*` | ✅ `root` masih memegang `*` (tanpa baris tambahan) |
| AC-4 | Idempoten | ✅ re-run body UP → `INSERT 0 0`; total tetap 3 perm / 3 mapping |
| AC-5 | down bersihkan tanpa sentuh lain | ✅ `down` → 0 permission `resident.%`; tabel lain tak berubah |
| AC-6 | Tidak ada INSERT di Go | ✅ `grep INSERT INTO permissions --include=*.go plugins` → NONE |
| AC-7 | build/vet/test/gofmt | ✅ lulus di container `app` |
| AC-8 | GET ok / POST 403 (viewer) | ⏳ butuh route ber-guard WO-04 (curl) |

Catatan: `migrate down --plugin auth` versi ini (`current=6`→`5`) menghapus 3 permission; `up` memulihkan.`

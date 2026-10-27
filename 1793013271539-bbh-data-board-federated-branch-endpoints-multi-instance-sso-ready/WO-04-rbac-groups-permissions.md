---
title: 'WO-04: RBAC — groups, permissions, admin_groups'
description: 'Ganti is_root dengan model izin berbasis grup: admins ↔ admin_groups ↔ groups ↔ group_permissions ↔ permissions. Izin efektif = union grup aktif. Guard per permission (RequirePermission). Fail-closed. Prasyarat WO-03 karena otorisasi cabang ditentukan lokal.'
status: complete
created: "2026-10-26"
updated: "2026-10-27"
revision: 1
---

# WO-04: RBAC — groups, permissions, admin_groups

**SPEC acuan:** `SPEC.md` (D-19..D-24, AC-25, AC-27..AC-31)
**Fase:** 2b
**Prasyarat:** tidak ada (hanya plugin `auth` yang sudah ada).
**Harus selesai sebelum:** **WO-03** — karena otorisasi di cabang ditentukan dari RBAC lokal (D-19).

## 1. Tujuan

Mengganti satu tombol boolean `is_root` dengan sistem izin yang bisa diatur:

```
admins ──< admin_groups >── groups ──< group_permissions >── permissions
```

Satu admin boleh punya **banyak** grup; satu grup boleh punya **banyak** permission. Izin efektif = **gabungan** semuanya.

## 2. Prinsip Wajib

| # | Prinsip | Konsekuensi |
|---|---|---|
| P-1 | **Fail-closed** (D-22) | Admin tanpa grup = tanpa izin. Tidak ada "default allow". |
| P-2 | **Izin di-resolve per request dari DB** (D-23) | Perubahan grup berlaku seketika; tidak disimpan di token. |
| P-3 | **Satu sumber kebenaran** (D-21) | `is_root` **dihapus**, bukan dipertahankan berdampingan. |
| P-4 | **Grup nonaktif tidak menyumbang izin** (AC-31) | `groups.is_active = false` → seluruh izinnya gugur. |
| P-5 | **Jangan sampai terkunci total** | Aksi yang menghilangkan admin `*` terakhir **harus ditolak**. |
| P-6 | **Bawaan ramah (D-25)** | Admin baru otomatis masuk grup `is_default` (`viewer`) → bisa melihat, belum bisa mengubah. Tidak melanggar fail-closed: grup tetap bisa dicabut. |

## 3. Skema (migrasi versioned, `CREATE TABLE` nol di Go)

```sql
CREATE TABLE permissions (
    key         TEXT PRIMARY KEY,          -- stabil, dirujuk kode (D-24)
    name        TEXT NOT NULL,
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE groups (
    id          TEXT PRIMARY KEY,          -- UUIDv7 (app-generated)
    key         TEXT UNIQUE NOT NULL,
    name        TEXT NOT NULL,
    description TEXT,
    is_system   BOOLEAN NOT NULL DEFAULT FALSE,   -- tidak boleh dihapus
    is_default  BOOLEAN NOT NULL DEFAULT FALSE,   -- diberikan otomatis ke admin baru (D-25)
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ NULL
);
CREATE INDEX idx_groups_deleted_at ON groups(deleted_at);
-- Hanya boleh ada satu grup default
CREATE UNIQUE INDEX uq_groups_one_default ON groups (is_default) WHERE is_default;

CREATE TABLE group_permissions (
    group_id       TEXT NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
    permission_key TEXT NOT NULL REFERENCES permissions(key) ON DELETE CASCADE,
    PRIMARY KEY (group_id, permission_key)
);
CREATE INDEX idx_group_permissions_perm ON group_permissions(permission_key);

CREATE TABLE admin_groups (
    admin_id   TEXT NOT NULL REFERENCES admins(id) ON DELETE CASCADE,
    group_id   TEXT NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (admin_id, group_id)
);
CREATE INDEX idx_admin_groups_group ON admin_groups(group_id);
```

### Seed permission (D-24)

```sql
INSERT INTO permissions (key, name) VALUES
    ('*',              'Semua izin'),
    ('admin.read',     'Lihat admin'),
    ('admin.write',    'Ubah admin'),
    ('admin.delete',   'Hapus admin'),
    ('group.read',     'Lihat grup'),
    ('group.write',    'Ubah grup & keanggotaan'),
    ('permission.read','Lihat daftar permission'),
    ('member.read',    'Lihat member'),
    ('member.write',   'Ubah member'),
    ('member.delete',  'Hapus member'),
    ('endpoint.read',  'Lihat endpoint'),
    ('endpoint.write', 'Ubah endpoint'),
    ('endpoint.test',  'Uji koneksi endpoint'),
    ('audit.read',     'Lihat audit')
ON CONFLICT (key) DO NOTHING;
```

### Seed grup sistem (id deterministik, bukan random)

**Tingkat izin (D-25):**

| Grup | Izin | Ringkas |
|---|---|---|
| `viewer` (**default**) | semua `*.read` **kecuali** `audit.read` | boleh lihat, tidak boleh ubah |
| `admin` | `*.read` + `*.write` + `endpoint.test` | boleh ubah, **tidak** boleh hapus / atur grup / lihat audit |
| `root` | `*` | semua |

```sql
INSERT INTO groups (id, key, name, description, is_system, is_default) VALUES
    ('01930000-0000-7000-8000-000000000001','root',  'Root',          'Semua izin',              TRUE, FALSE),
    ('01930000-0000-7000-8000-000000000002','admin', 'Administrator', 'Ubah data, tanpa hapus',  TRUE, FALSE),
    ('01930000-0000-7000-8000-000000000003','viewer','Viewer',        'Hanya baca (default)',    TRUE, TRUE)
ON CONFLICT (key) DO NOTHING;

INSERT INTO group_permissions (group_id, permission_key) VALUES
    -- root: semua
    ('01930000-0000-7000-8000-000000000001','*'),

    -- admin: baca + tulis, TANPA delete, TANPA group.write, TANPA audit.read
    ('01930000-0000-7000-8000-000000000002','admin.read'),
    ('01930000-0000-7000-8000-000000000002','admin.write'),
    ('01930000-0000-7000-8000-000000000002','group.read'),
    ('01930000-0000-7000-8000-000000000002','permission.read'),
    ('01930000-0000-7000-8000-000000000002','member.read'),
    ('01930000-0000-7000-8000-000000000002','member.write'),
    ('01930000-0000-7000-8000-000000000002','endpoint.read'),
    ('01930000-0000-7000-8000-000000000002','endpoint.write'),
    ('01930000-0000-7000-8000-000000000002','endpoint.test'),

    -- viewer (DEFAULT): hanya baca, TANPA audit.read
    ('01930000-0000-7000-8000-000000000003','admin.read'),
    ('01930000-0000-7000-8000-000000000003','group.read'),
    ('01930000-0000-7000-8000-000000000003','permission.read'),
    ('01930000-0000-7000-8000-000000000003','member.read'),
    ('01930000-0000-7000-8000-000000000003','endpoint.read')
ON CONFLICT DO NOTHING;
```

> ⚠️ **`audit.read` sengaja tidak diberikan ke `viewer` maupun `admin`.** Audit memuat jejak request lintas instance (endpoint, path, status, durasi) — cukup diketahui pemegang akses penuh (D-25, AC-34).

### Grup default (D-25)

Saat admin dibuat — **CLI maupun API** — keanggotaan grup default ditambahkan **dalam transaksi yang sama**:

```sql
INSERT INTO admin_groups (admin_id, group_id)
SELECT $1, g.id FROM groups g
WHERE g.is_default = TRUE AND g.is_active AND g.deleted_at IS NULL;
```

**Alasan:** tanpa ini, admin baru tidak punya izin apa pun (fail-closed D-22) dan langsung bingung karena tidak bisa melihat apa-apa. Dengan ini perilaku bawaannya: **bisa melihat, belum bisa mengubah** — tepat seperti yang diharapkan untuk admin non-root.

**Yang tetap berlaku:** grup default **bukan** hak istimewa — ia bisa dicabut per admin tanpa mengubah yang lain. Tanpa grup sama sekali → tetap tanpa akses.

### Migrasi data `is_root` → grup `root` (D-21, AC-29)

```sql
INSERT INTO admin_groups (admin_id, group_id)
SELECT a.id, '01930000-0000-7000-8000-000000000001'
FROM admins a
WHERE a.is_root = TRUE
ON CONFLICT DO NOTHING;
```

> **Urutan aman:** migrasi ini **harus** berjalan **sebelum** kolom `is_root` dihapus. Penghapusan kolom dilakukan di migrasi **terpisah**, **setelah** kode berhenti membacanya. Jangan digabung — kalau digabung dan ada bug, tidak ada jalan pulang.

## 4. Resolusi Izin Efektif

```sql
SELECT DISTINCT gp.permission_key
FROM admin_groups ag
JOIN groups g           ON g.id = ag.group_id
                       AND g.is_active = TRUE
                       AND g.deleted_at IS NULL
JOIN group_permissions gp ON gp.group_id = g.id
WHERE ag.admin_id = $1;
```

Lalu di kode:

```go
func (p PermissionSet) Has(need string) bool {
    if p.HasAll { return true }        // berisi "*"
    return p.keys[need]
}
```

**Aturan:** bila hasil memuat `*` → **semua** izin dianggap ada. Ini satu-satunya jalan "root".

## 5. Guard

| Lama | Baru |
|---|---|
| `RequireRootAdmin(svc)` | `RequirePermission(svc, "*")` |
| — | `RequirePermission(svc, "admin.write")` dst. |

```go
func RequirePermission(svc *services.Service, perm string) gin.HandlerFunc
```

Urutan pemeriksaan (semua wajib, berhenti di kegagalan pertama):

1. Token sah & `typ` benar → tidak → **401**
2. Admin ada, `is_active`, `status = active` → tidak → **403**
3. Izin efektif memuat `perm` atau `*` → tidak → **403**

Pengelompokan route (contoh, selaras D-24):

| Route | Izin |
|---|---|
| `GET /api/admin/me`, `POST /api/admin/change-password` | *cukup terautentikasi* |
| `GET /api/admins`, `GET /api/admins/:id` | `admin.read` |
| `POST/PUT/PATCH /api/admins/:id` | `admin.write` |
| `DELETE /api/admins/:id` | `admin.delete` |
| `GET /api/members*` | `member.read` |
| `POST/PUT/PATCH /api/members*` | `member.write` |
| `DELETE /api/members/:id` | `member.delete` |
| `/api/groups*`, `/api/permissions*` | `group.read` / `group.write` / `permission.read` |
| `GET /api/endpoints*` | `endpoint.read` |
| `POST/PUT/DELETE /api/endpoints*`, `POST .../test` | `endpoint.write` / `endpoint.test` |
| `GET /api/audit*` | `audit.read` (root saja secara bawaan) |

> ✅ **Keputusan (D-25):** `GET /api/admins` dibuka untuk `admin.read` — **memang disengaja**. Grup `viewer` (default) memegang `admin.read` tetapi **tidak** memegang `admin.write`, sehingga admin non-root **bisa melihat daftar, tidak bisa mengubah**. Yang tetap dijaga: `admin.write` **tidak boleh** memberi izin melebihi milik pelaku (aturan anti-eskalasi di bawah).

### Aturan anti-eskalasi

- Pelaku **tidak boleh** memberi grup yang mengandung izin di luar miliknya.
- Pelaku **tidak boleh** mengubah keanggotaan grup dirinya sendiri bila itu menambah izin.
- Alternatif lebih ketat: hanya pemegang `*` boleh mengubah keanggotaan grup.

## 6. Anti-lockout (P-5)

Wajib menolak aksi berikut:

| Aksi | Syarat ditolak |
|---|---|
| Hapus/nonaktifkan grup `root` | grup `is_system = TRUE` → **selalu ditolak** |
| Lepas grup `root` dari admin | bila itu **admin terakhir** yang memegang `*` → tolak |
| Nonaktifkan / hapus admin | bila itu **admin aktif terakhir** pemegang `*` → tolak |

Pesan error harus menyebutkan sebabnya ("ini admin terakhir dengan akses penuh"), bukan sekadar "forbidden".

## 7. CLI

```
auth group list
auth group create --key gudang --name "Gudang" --permissions endpoint.read,audit.read
auth group update --key gudang --activate|--deactivate
auth group update --key gudang --set-default      # memindahkan grup default
auth group assign   --admin <email|id> --group gudang
auth group unassign --admin <email|id> --group gudang
auth group show     --key gudang          # daftar permission-nya
auth permission list
auth admin groups --admin <email|id>      # daftar grup admin
```

**Perilaku `create-admin` (D-25):** setelah admin dibuat, otomatis ditambahkan ke grup `is_default = TRUE`. Tambahkan flag `--no-default-group` bila memang ingin admin tanpa grup (untuk pengujian fail-closed).

## 8. API

| Method | Path | Izin |
|---|---|---|
| `GET` | `/api/permissions` | `permission.read` |
| `GET` | `/api/groups` | `group.read` |
| `POST` | `/api/groups` | `group.write` |
| `GET/PUT/DELETE` | `/api/groups/:id` | `group.read` / `group.write` |
| `GET` | `/api/admins/:id/groups` | `group.read` |
| `PUT` | `/api/admins/:id/groups` | `group.write` |

`PUT /api/admins/:id/groups` menerima **daftar lengkap** (`{"group_keys":["gudang"]}`) — bukan tambah/kurang — agar tidak ada kehilangan data karena partial update (pelajaran dari bug `status` yang ter-reset).

## 9. Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | Admin 2 grup mendapat **gabungan** izin; lepas satu grup → izin berkurang seketika (AC-27) | test + SQL |
| 2 | Admin tanpa grup → **403** di route ber-izin, bukan diizinkan (AC-28) | test |
| 3 | Setiap admin `is_root=true` lama **punya akses penuh** setelah migrasi (AC-29) | seed + test |
| 4 | Cabut grup → request berikutnya langsung **403**, tanpa login ulang (AC-30) | test |
| 5 | Grup `is_active=false` → izinnya **tidak dihitung** (AC-31) | test |
| 6 | Grup sistem (`root`) **tidak bisa** dihapus / dinonaktifkan | test |
| 7 | Tidak bisa melepas admin `*` terakhir | test |
| 8 | **Grup default (D-25):** admin baru via CLI **dan** API otomatis jadi `viewer`; bisa `GET`, **403** saat `POST/PUT/DELETE` (AC-32) | test |
| 9 | **Pemisahan baca/tulis (AC-33):** `viewer` tidak bisa ubah; `admin` bisa ubah tetapi **tidak** bisa hapus & tidak bisa `PUT /api/admins/:id/groups` | test |
| 10 | **Audit terbatas (AC-34):** `viewer` & `admin` → **403** di `GET /api/audit*`; hanya `root` | test |
| 11 | Hanya satu grup `is_default` (unique index menolak yang kedua) | test |
| 12 | Semua route lama berpindah dari `RequireRootAdmin` ke `RequirePermission` | grep: `RequireRootAdmin` = 0 hasil |
| 13 | `go build ./... && go vet ./... && go test ./...` lulus | CI |

## 10. Definition of Done

- [x] Semua AC lulus dengan bukti output nyata
- [x] Migrasi **dipisah** dari penghapusan kolom `is_root` (bisa pulang)
- [x] Bukti `is_root` benar-benar tidak dibaca kode lagi: grep bersih
- [x] Seed idempotent (`ON CONFLICT DO NOTHING`) — migrasi dijalankan ulang tidak error
- [x] Anti-lockout diuji (uji negatif: pastikan aksinya **ditolak**)
- [x] Rebuild binary eksplisit (`go build -o bin/console ./cmd/console`)
- [x] Bagian yang belum terbukti dinyatakan belum terbukti
- [x] Tidak ada push ke remote

## 13. Status Implementasi (2026-10-27)

**Implementasi:** di dalam plugin `auth` (fitur = plugin, bukan `internal/`).

Migrasi (3 file, urut):
1. `2026_10_27_100000_create_rbac_tables` — tabel + seed 14 permission + 3 grup sistem.
2. `2026_10_27_101000_migrate_is_root_to_root_group` — data `is_root=true` → grup `root`; admin non-root tanpa grup → grup default.
3. `2026_10_27_102000_drop_admins_is_root` — hapus kolom (down: re-add + backfill dari grup root).

Kode: `model/rbac.go`, `services/rbac.go`, `handlers/rbac.go`, `cli_rbac.go`, middleware `RequirePermission`, guard di plugin `endpoint` (`endpoint.read/write/test`, `audit.read`).

Bukti uji (server + DB lokal):

| AC | Hasil |
|---|---|
| 1 | Admin `viewer`+`admin` → `POST /api/members` 201; setelah `admin` dicabut → **403** seketika (token sama) |
| 2 | Admin tanpa grup (`--no-default-group`) → `/api/admins` **403**, `/api/admin/me` 200 |
| 3 | Setelah migrasi `root@admin.local` & `root@bbh.dev` → grup `root` (akses penuh) |
| 4 | Cabut grup berlaku tanpa login ulang (AC-1 di atas) |
| 5 | Grup `gudang` dinonaktifkan → `endpoint.read` langsung **403** |
| 6 | `DELETE /api/groups/root` → 409; `PUT` `is_active=false` pada grup sistem → 409 |
| 7 | `auth group unassign --admin root@admin.local --group root` (satu-satunya pemegang `*`) → ditolak "last admin with full access" |
| 8 | Admin baru via API otomatis `["viewer"]`; via CLI juga; `GET` 200 / `POST` 403 |
| 9 | Grup `admin`: `POST /api/members` 201, `DELETE /api/admins/:id` 403, `PUT /:id/groups` 403 |
| 10 | `GET /api/audit/endpoints` → viewer 403, admin 403, root 200 |
| 11 | `set_default` pindah grup; `COUNT(*) WHERE is_default = 1`; UPDATE kedua ditolak `uq_groups_one_default` |
| 12 | `grep RequireRootAdmin` = **0** |
| 13 | `go build ./... && go vet ./... && go test ./...` lulus |

Rollback diuji: `migrate down --plugin auth` (versi 4) → kolom `is_root` kembali berisi nilai benar (`t` hanya untuk anggota grup root); `migrate up` → kolom hilang lagi.

**Catatan:** `docs/swagger/docs.go` masih memuat `is_root` karena file generated (`docs/` gitignored). Regenerasi `swag init` diperlukan bila ingin dokumentasi bersih.

**Belum terbukti:** UI pengelolaan grup (out of scope), RBAC untuk member (out of scope).

## 11. Out of Scope

- RBAC untuk **member** (federasi scope admin saja)
- Row-level / data-scoped permission (mis. "hanya cabang sendiri")
- UI pengelolaan grup (CLI + API dulu)
- Permission inheritance antar grup

## 12. Hubungan dengan WO Lain

| WO | Hubungan |
|---|---|
| WO-01 (endpoint registry) | Izin `endpoint.*` dipakai di route endpoint; **tidak** memblokir WO-01 |
| WO-02 (routing) | Tidak mengubah auth; aman berjalan paralel |
| **WO-03 (SSO)** | **Prasyarat.** Otorisasi cabang = RBAC cabang (D-19). Tanpa WO ini, SSO tidak punya dasar menentukan hak akses |

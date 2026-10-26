---
title: 'WO-04: RBAC — groups, permissions, admin_groups'
description: 'Ganti is_root dengan model izin berbasis grup: admins ↔ admin_groups ↔ groups ↔ group_permissions ↔ permissions. Izin efektif = union grup aktif. Guard per permission (RequirePermission). Fail-closed. Prasyarat WO-03 karena otorisasi cabang ditentukan lokal.'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
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
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ NULL
);
CREATE INDEX idx_groups_deleted_at ON groups(deleted_at);

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

```sql
INSERT INTO groups (id, key, name, description, is_system) VALUES
    ('01930000-0000-7000-8000-000000000001','root',  'Root',          'Semua izin', TRUE),
    ('01930000-0000-7000-8000-000000000002','admin', 'Administrator', 'Kelola admin & data', TRUE),
    ('01930000-0000-7000-8000-000000000003','viewer','Viewer',        'Hanya baca', TRUE)
ON CONFLICT (key) DO NOTHING;

INSERT INTO group_permissions (group_id, permission_key) VALUES
    ('01930000-0000-7000-8000-000000000001','*'),

    ('01930000-0000-7000-8000-000000000002','admin.read'),
    ('01930000-0000-7000-8000-000000000002','admin.write'),
    ('01930000-0000-7000-8000-000000000002','group.read'),
    ('01930000-0000-7000-8000-000000000002','permission.read'),
    ('01930000-0000-7000-8000-000000000002','member.read'),
    ('01930000-0000-7000-8000-000000000002','member.write'),
    ('01930000-0000-7000-8000-000000000002','endpoint.read'),
    ('01930000-0000-7000-8000-000000000002','endpoint.write'),
    ('01930000-0000-7000-8000-000000000002','endpoint.test'),
    ('01930000-0000-7000-8000-000000000002','audit.read'),

    ('01930000-0000-7000-8000-000000000003','admin.read'),
    ('01930000-0000-7000-8000-000000000003','group.read'),
    ('01930000-0000-7000-8000-000000000003','permission.read'),
    ('01930000-0000-7000-8000-000000000003','member.read'),
    ('01930000-0000-7000-8000-000000000003','endpoint.read'),
    ('01930000-0000-7000-8000-000000000003','audit.read')
ON CONFLICT DO NOTHING;
```

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

> ⚠️ Route `/api/admins` **sekarang** dijaga `RequireRootAdmin` untuk **semua** method. Setelah WO ini, `GET /api/admins` terbuka untuk `admin.read` — pastikan ini memang diinginkan, dan **jangan** sampai `admin.write` bisa memberi izin lebih tinggi dari yang dimiliki pelaku.

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
auth group assign   --admin <email|id> --group gudang
auth group unassign --admin <email|id> --group gudang
auth group show     --key gudang          # daftar permission-nya
auth permission list
auth admin groups --admin <email|id>      # daftar grup admin
```

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
| 8 | Semua route lama berpindah dari `RequireRootAdmin` ke `RequirePermission` | grep: `RequireRootAdmin` = 0 hasil |
| 9 | `go build ./... && go vet ./... && go test ./...` lulus | CI |

## 10. Definition of Done

- [ ] Semua AC lulus dengan bukti output nyata
- [ ] Migrasi **dipisah** dari penghapusan kolom `is_root` (bisa pulang)
- [ ] Bukti `is_root` benar-benar tidak dibaca kode lagi: grep bersih
- [ ] Seed idempotent (`ON CONFLICT DO NOTHING`) — migrasi dijalankan ulang tidak error
- [ ] Anti-lockout diuji (uji negatif: pastikan aksinya **ditolak**)
- [ ] Rebuild binary eksplisit (`go build -o bin/console ./cmd/console`)
- [ ] Bagian yang belum terbukti dinyatakan belum terbukti
- [ ] Tidak ada push ke remote

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

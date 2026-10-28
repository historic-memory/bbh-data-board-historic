---
title: "WO-01: Kontrak Backend Tipis (permissions, groups, filter status, total)"
description: "Menutup GAP-5/6/7: expose permissions & groups di /api/admin/me, tambah filter status + total pada list resident, dan (opsional) endpoint stats. Perubahan additive, tanpa migrasi."
status: complete
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - backend
  - auth
  - resident
---

# WO-01: Kontrak Backend Tipis

## Referensi
- SPEC topik ini: `spec.md` — A-10, A-11, GAP-5..GAP-7, AC-8.
- `plugins/auth/plugin.go` (route), `plugins/auth/handlers/{handler.go,auth.go}`.
- `plugins/auth/services/rbac.go` — `EffectivePermissions`, `AdminGroupKeys`, `ListAdminGroups`.
- `plugins/resident/{handlers/handlers.go,services/service.go,model/view.go}`.

## Tujuan
Menyediakan data minimum yang dibutuhkan UI tanpa menambah fitur baru:
1. **B-1** `/api/admin/me` mengembalikan `permissions: []string` dan `groups` terisi.
2. **B-2** `GET /api/admin/residents` mendukung `?status=` dan mengembalikan `total`.
3. **B-3** (opsional) `GET /api/admin/residents/stats` untuk dashboard.

## Batasan
- Additive & backward-compatible: field/param baru, bukan perubahan nama.
- **Tanpa migrasi DB baru.** Tanpa perubahan model tabel.
- `internal/` tidak disentuh.
- Permission guard tetap sama (`resident.read`, dll).

## Langkah

### T-1 — `/api/admin/me` (B-1)
1. Tambah `Permissions []string \`json:"permissions"\`` pada `AdminView` (`handler.go`).
2. Ubah `adminView` agar menerima data grup & permission (atau buat
   `adminViewWithRBAC(svc, admin)`).
3. Isi `Groups` dari `svc.AdminGroupKeys(admin.ID)` (bukan `[]string{}` hard-code).
4. Isi `Permissions` dari `svc.EffectivePermissions(admin.ID)`; wildcard `*`
   diteruskan apa adanya agar UI bisa menampilkan "semua akses".
5. Terapkan juga ke `MeAdmin`; endpoint lain yang memakai `adminView` tetap
   menghasilkan `groups`/`permissions` kosong **atau** diisi (pilih konsisten —
   rekomendasi: isi `Groups` saja di list, `Permissions` hanya di `/me` untuk
   menghindari N+1).

### T-2 — List resident (B-2)
1. `ListFilter` tambah `Status string`.
2. Query: bila `Status != ""`, filter berdasarkan status terakhir
   (`resident_status_history` baris terbaru per resident). Implementasi:
   subquery `DISTINCT ON (resident_id) ... ORDER BY effective_at DESC`, atau
   join ke view status terkini — pilih yang tidak mengubah skema.
3. Validasi `status` terhadap enum statis; nilai tak dikenal → 400 `ErrInvalidEnum`
   (konsisten WO-04 lama).
4. `ListResponse` tambah `Total int64 \`json:"total"\``; hitung dengan query
   `COUNT(*)` memakai filter yang sama. `items` tetap `[]` bila kosong.
5. Handler `List` membaca `c.Query("status")`.

### T-3 — Stats (B-3, opsional)
`GET /api/admin/residents/stats?branch_key=` → `{ "by_status": {"AKTIF": 12, ...},
"by_branch": {"jktbaru": 5, ...}, "total": 17 }`. Guard `resident.read`. Skip bila
T-2 sudah cukup untuk WO-07.

### T-4 — Dokumentasi
- Regenerasi swagger (`build.sh` di container `app`) → `docs/` (gitignored).
- Tambah request Bruno: `api-tests/auth/02-Me.bru` (cek `permissions`),
  `api-tests/smoke/04-List-residents-filtered.bru` (`?status=&limit=`).

## Kriteria Penerimaan
- [AC-1] `GET /api/admin/me` mengembalikan `groups` berisi key grup nyata dan
  `permissions` berisi permission efektif (root → `["*"]`).
- [AC-2] `GET /api/admin/residents?status=<valid>` memfilter benar (dibuktikan
  dengan data uji >1 status).
- [AC-3] `GET /api/admin/residents?status=<invalid>` → 400.
- [AC-4] Respons list memuat `total` yang konsisten dengan `items` (uji
  `limit=1` + `offset`).
- [AC-5] Viewer (tanpa `resident.write`) tetap 200 pada GET; 403 pada mutasi
  (tidak berubah).
- [AC-6] `go build ./...`, `go vet ./...`, `go test ./...` hijau.
- [AC-7] Swagger & Bruno diperbarui; paritas path/op tidak turun.

## Bukti yang Diharapkan
Output `curl` (me, list+filter, list invalid, total) + hasil `go test` + angka
paritas swagger. Tanpa push.

## Risiko
- Filter status dengan `DISTINCT ON` khusus PostgreSQL (DB proyek = Postgres;
  bila kelak multi-DB, catat sebagai batasan).
- `EffectivePermissions` per admin di `/me` = 1 query tambahan; aman karena
  endpoint dipanggil jarang.

## Bukti Pelaksanaan (2026-10-28)

### Implementasi
**B-1 — `/api/admin/me`**
- `plugins/auth/handlers/handler.go`: `AdminView` + `Permissions []string \`json:"permissions"\``
  (selalu non-null, default `[]`).
- `plugins/auth/handlers/rbac.go`: helper baru `adminViewWithRBAC` = `adminViewWithGroups`
  + `svc.EffectivePermissions(id).Keys()`. `adminViewWithGroups` (dipakai list/detail)
  tetap tanpa `permissions` → hindari N+1.
- `plugins/auth/handlers/auth.go`: `MeAdmin` memakai `adminViewWithRBAC`.
- **BUG DITEMUKAN & DIPERBAIKI**: `PermissionSet.Keys()` mengeluarkan `"*"` dua kali
  (dari map `keys` dan dari cabang `hasAll`) → `/me` root mengembalikan `["*","*"]`.
  Kini wildcard di-skip di loop map lalu ditambahkan sekali. Regresi dikunci
  `TestPermissionSetWildcard` + `TestPermissionSetKeysSortedWithoutDuplicates`.

**B-2 — filter status + total**
- `plugins/resident/services/service.go`: `ListFilter.Status`; tipe baru `ResidentStats`.
- `plugins/resident/services/query.go`: `listFrom` (satu sumber FROM + LATERAL status
  terakhir) dipakai bersama oleh list, COUNT, dan stats; `normalizeListFilter`
  (trim, validasi `ValidStatuses` → `ErrUnknownEnum`, clamp limit 1..200, offset ≥ 0);
  `buildListWhere` dipakai list **dan** count agar `total` tak pernah melenceng.
- `plugins/resident/handlers/handlers.go`: `ListResponse` + `Total int64`;
  `newListResponse(rows, total)`; helper `countOf` untuk list non-paginated
  (`status`, `documents`, `mothers`, `facilities`) supaya `total` tetap konsisten.
- **Catatan desain**: `GET /api/admin/residents` sekarang **selalu** memuat `total`.
  Ini additive; item tetap array. Bentuk ini dipilih agar paginasi UI (WO-04) tidak
  perlu endpoint terpisah.

**B-3 — stats**
- `GET /api/admin/residents/stats?branch_key=` (guard `resident.read`), didaftarkan
  **sebelum** `/:id`; gin 1.9.1 memprioritaskan segmen statis sehingga `/stats`
  tidak tertangkap `Get` (terverifikasi dari log route + hasil 200).
- Bentuk: `{total, by_status, by_branch}`. Resident tanpa baris status tidak masuk
  bucket mana pun → UI menghitung sisanya `total - sum(by_status)`.

### Gate lulus (curl, port 8082)
```
# B-1 /me
root     -> groups [root]        permissions ["*"]        (tidak duplikat)
viewer   -> groups [viewer]      permissions [admin.read, endpoint.read,
                                  group.read, member.read, permission.read,
                                  resident.read]

# B-2 list (data uji: 2 resident berstatus + 1 tanpa status)
?q=Uji WO01                    -> total 3, n 3, current_status tiap baris benar
?q=Uji WO01&status=DIRAWAT     -> total 1, n 1 [Uji WO01 Satu]
?q=Uji WO01&status=DIKEMBALIKAN-> total 1, n 1 [Uji WO01 Dua]
?q=Uji WO01&limit=1&offset=0   -> total 3, n 1
?q=Uji WO01&limit=1&offset=1   -> total 3, n 1 [Uji WO01 Dua]   (total stabil)
?status=NGASAL                 -> 400 {"error":"unknown enum value: status=\"NGASAL\" tidak dikenal"}

# B-3 stats
?branch_key=local -> {"total":3,"by_status":{"DIKEMBALIKAN":1,"DIRAWAT":1},
                      "by_branch":{"local":3}}

# AC-5 viewer (grup viewer, tanpa resident.write)
GET  list   -> 200
GET  stats  -> 200
POST create -> 403
```

### Test
- `go build ./...`, `go vet ./...`, `go test ./...` **hijau** (exit 0).
- Test baru: `plugins/resident/services/query_test.go` (validasi filter, WHERE
  bersama), `plugins/resident/services/query_db_test.go` (filter status pakai
  status TERAKHIR, `total` vs `limit`, stats), `plugins/auth/services/rbac_test.go`
  (+2 regresi wildcard).

### Swagger & Bruno (AC-7)
- `build.sh` di container `app` → `docs/swagger` regenerated (gitignored).
  `paths_total=37` (sebelum 36), `ops_total=58`, `/api/admin/residents/stats` ada,
  `AdminView.permissions` ada, `ListResponse{items,total}`.
  Catatan: penghitung lama (`grep resident`) menghitung *path*; angka resident kini
  5 path (`/residents`, `/stats`, `/{id}`, `/{id}/status`, `/{id}/documents`) +
  mothers/facilities/template = 10 path total.
- Bruno: `api-tests/auth/02-Me.bru` (groups + permissions, wildcard ≤1),
  `api-tests/smoke/04-List-residents-filtered.bru` (`total` independen dari `limit`).
  Hasil: `bru run --env Local auth` 2 req / 6 test PASS; `--tags=smoke` 4 req /
  9 test PASS.

### Data uji
Semua resident uji (`Uji WO01 *`) dan admin `uji-wo01-viewer@test.local` sudah
**dihapus** setelah verifikasi; DB kembali `{"items":[],"total":0}`. Tidak ada
PII/token ter-commit.

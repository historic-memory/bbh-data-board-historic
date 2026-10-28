---
title: "WO-01: Kontrak Backend Tipis (permissions, groups, filter status, total)"
description: "Menutup GAP-5/6/7: expose permissions & groups di /api/admin/me, tambah filter status + total pada list resident, dan (opsional) endpoint stats. Perubahan additive, tanpa migrasi."
status: draft
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

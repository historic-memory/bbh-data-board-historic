---
title: "WO-01: Endpoint agregat dashboard"
description: "Plugin `plugins/dashboard` (skeleton tanpa tabel sendiri), seed permission `dashboard.read` via migrasi plugin auth, dan endpoint `GET /api/admin/dashboard/summary` yang menggabungkan resident + hsh + sec."
type: work-order
status: pending
created: "2026-11-03"
updated: "2026-11-03"
tags:
  - work-order
  - dashboard
  - backend
---

# WO-01: Endpoint agregat dashboard

## Referensi
- SPEC D-1..D-6 (endpoint, plugin penempat, periode aktif, izin, fail-open, bentuk respons).
- Pola skeleton plugin: `plugins/sec` (WO-01 topic `1793265053246`, commit `5619cd4`).
- Pola seed permission: migrasi plugin auth `2026_10_29_120200_add_hsh_permissions`.

## Langkah

1. **Skeleton plugin `plugins/dashboard`**
   - `plugin.go` (`ID() = "dashboard"`, `New(db)` menerima koneksi DB), terdaftar di
     `cmd/server/main.go` + `cmd/console/main.go`.
   - `cli.go` minimal (`dashboard version`) mengikuti pola `hsh`/`sec`.
   - TANPA folder `model/` + TANPA migrasi tabel sendiri — hanya baca.
2. **Seed permission di plugin `auth`**
   - Migrasi baru plugin auth: seed `dashboard.read` (admin = read; viewer = read;
     root lewat `*`). **JANGAN** menaruh seed permission di plugin dashboard.
3. **Endpoint agregat**
   - Route `GET /api/admin/dashboard/summary` (guard `dashboard.read`) via
     `middleware.RequirePermission`.
   - Service menghitung 3 blok dengan SQL/GORM langsung atas tabel
     `residents`, `hsh_periods`/`hsh_entries`/`hsh_items`/`hsh_categories`,
     `sec_periods`/`sec_entries`/`sec_items`/`sec_categories` — **tanpa import
     services plugin lain** (loose coupling antar-plugin).
   - Blok `residents`: total, `by_status`, `by_branch` (semantik sama dengan
     `ResidentStats` existing).
   - Blok `hsh`/`sec`: ambil **periode aktif** (`is_active = TRUE`); bila tidak ada
     → `period: null` + semua counter `0`. Bila (tidak seharusnya) lebih dari satu,
     ambil `updated_at` terbaru dan catat log warning.
   - Ringkas status: `{total, urgent, enough, unknown, by_category[]}` — `by_category`
     mengikuti `CategorySummary` (per kategori dengan urgent/enough/unknown).
4. **Fail-open per sub-blok (D-5)**
   - Aktor TANPA `hsh.read` → blok `hsh` = `null`; TANPA `sec.read` → `sec = null`.
     Cek permission dari konteks RBAC yang sudah diletakkan middleware; verifikasi
     mekanismenya saat implementasi.
   - Endpoint utama tetap 200 selama `dashboard.read` lolos — kegagalan satu blok
     (mis. DB tabel belum ada) diisolasi per-blok dengan error info, bukan 500 total.

## Kriteria Penerimaan

- [AC-1] `go build ./...`, `go vet ./...`, `go test ./...` hijau.
- [AC-2] Migrasi up → `dashboard.read` ada di `permissions` + ter-link ke grup
  admin & viewer; down bersih. `migrate --plugin all` idempoten.
- [AC-3] Tanpa token → 401; viewer/admin (punya `dashboard.read`) → 200.
- [AC-4] Respons berisi 3 blok dengan periode aktif yang benar; tanpa periode aktif
  hsh → `"period": null` + counter 0, HTTP tetap 200.
- [AC-5] Admin tanpa `sec.read` → `sec: null` tetapi `residents` + `hsh` tetap terisi.
- [AC-6] Angka blok hsh/sec **identik** dengan `GET /admin/hsh/entries/summary` &
  `GET /admin/sec/entries/summary` pada data yang sama.
- [AC-7] Angka blok residents identik dengan `GET /api/admin/residents/stats`.
- [AC-8] Route terdaftar SEBELUM wildcard apa pun yang bisa menelan `summary`
  (pelajari pola summary-existing).

## Hasil Implementasi
(commit + bukti menyusul)

---
title: 'WO-05: Plugin business + Migration File'
description: Plugin business service-only (ping endpoint, ConnectionName=business) + migration file business_meta + verifikasi per-DB tracking.
status: complete
created: "2026-10-26"
updated: "2026-10-26"
---
# WO-05: Plugin `business` + Migration File

## Goal
Plugin HTTP service terisolasi dengan koneksi `business`, plus migration file pertama agar AC-5 bisa diverifikasi nyata.

## A. Plugin (`plugins/business/`)
```
plugins/business/
├── plugin.go           // Plugin struct { bizDB, branchDBs }, ConnectionName()="business"
├── handlers/handler.go // GET /api/business/ping + health koneksi
├── services/service.go // factory pattern, inject bizDB
└── migrations/postgres/ (diisi di bagian B)
```
1. `RegisterServices(deps)`: ambil `deps.Connections["business"]`; bila nil → log warning `[business] DB not configured (BUSINESS_DB_*), routes degraded`, tetap return nil (**non-fatal**, AC-8).
2. `RegisterRoutes`: `api.GET("/business/ping")` → `{status:"ok", db:"business", connected:bool, branches:[...]}`.
3. `RegisterMiddleware`, `Seed` → no-op.
4. `ConnectionName() string` → `"business"` (implement ConnectionBinder dari WO-02).
5. Registrasi di `cmd/server/main.go` dan `cmd/console/main.go`.

## B. Migration File
1. `docker compose exec app console migrate make --plugin business create_business_tables` → isi tabel placeholder:
   - `business_meta`: `id TEXT PK, key TEXT UNIQUE, value TEXT, updated_at TIMESTAMPTZ` (UUID v7 dari `internal/uuid`, pola auth).
   - Buat pasangan `.up.sql` + `.down.sql`.
2. Verifikasi per-DB: `console migrate up --plugin business` → via psql, tabel `migrations` muncul di **db business** dan **tiap db cabang**, TIDAK di db utama.

## Acceptance
- [ ] `/api/business/ping` return `connected:true` + daftar branches (AC-7 terkait routing, AC-9 terkait non-fatal).
- [ ] Migrasi business masuk ke DB business + cabang, bukan db utama (AC-4, AC-5).
- [ ] Auth tetap bekerja — tidak ada regresi.

## Dependencies
- WO-02 (ServiceDeps.Connections), WO-03 (guard + migrate make terdeteksi).

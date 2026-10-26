---
title: work order
description: "Master index WO multi-database donisgo — pecahan per fase: WO-01..WO-09. Opsi B: CLI bin/business terpisah. Revisi 2: registry DBMS-centric (dbms key)."
status: draft
created: "2026-10-26"
updated: "2026-10-26"
---
# Work Order (Master Index): Multi-Database — Named DB Registry + CLI Business Terpisah

## Referensi
- SPEC: `SPEC.md` di topic ini (D-1..D-20, AC-1..AC-22).
- Prinsip: **auth tetap di DB `default`**; plugin `business` = PostgreSQL terpisah; 1 DB per cabang; **server DBMS didaftarkan sekali (`dbms.key`), cabang mengacu `dbms_key`** (D-19/D-20).
- **Opsi B dipilih (2026-10-26)**: CLI khusus `bin/business` untuk operasi business/cabang → **defense in depth**: (1) isolasi fisik — proses CLI tidak pernah membuka DB `default`; (2) guard logical `--connection` di console tetap berlaku.

## Konteks Kode Aktual (hasil riset)
| File | Temuan |
|---|---|
| `internal/db/gorm.go` | `GetGormDB()` singleton via `sync.Once` → `openGormDB()` membaca env `DB_*` langsung, build DSN (postgres/mysql), `configurePool(sqlDB)`, ping. Helper: `intFromEnv`, `detectDBType`, `maskDSN` |
| `internal/db/db.go` | `GetDB()` (`*sql.DB`) — pola sama, dipakai tidak oleh runtime server (fallback CLI) |
| `internal/plugins/types.go` | `ServiceDeps{DB *gorm.DB; Store storage.Store}`; interface `Plugin` 5 method |
| `internal/plugins/registry.go` | `RegisterAllServices(db, store)` loop semua plugin → `p.RegisterServices(deps)` |
| `internal/app/bootstrap.go` | `New()` → `db.GetGormDB()` → `attachPlugins()` → `RegisterAllServices(a.gdb, a.store)` |
| `internal/console/migrate.go` (±780 baris) | `migrationTarget{Name, Path}`; `collectTargets(target, dbType)`; `applyUp/Down/DownAll` memanggil `db.GetGormDB()` **langsung** (hard-coded default); tracking tabel `migrations`, `migration_targets` dibuat di DB sasaran; topo-sort antar plugin via regexp `create table`/`references`; flags: `--plugin`, `--db` |
| `cmd/server/main.go`, `cmd/console/main.go` | Registrasi `authplugin.New()` |
| `docker-compose.yml` | Service `postgres` (host 5433), `keydb`, `pgadmin`, `app` (IDLE, `depends_on` healthy); volumes `pgdata`, dll; network `bbb-data-backend` |

## Daftar WO per Fase
WO dipecah per fase agar ringan dan bisa dikerjakan/commit bertahap. File detail ada di folder topic ini.

| # | File | Scope | Status |
|---|---|---|---|
| 1 | `WO-01-named-db-registry.md` | `internal/db/registry.go`: skema `dbms`/`dbms_secrets`/`db_registry`, `GetGorm(name)`, JOIN, loader | complete (rev 2) |
| 2 | `WO-02-servicedeps-connections.md` | `ServiceDeps.Connections`, bootstrap resolve (default wajib, business best-effort), `ConnectionBinder` | complete |
| 3 | `WO-03-migration-extract-guard.md` | Extract `internal/migration/` + flag `--connection` + guard isolasi D-13/D-14 | complete |
| 4 | `WO-04-business-cli.md` | **Opsi B**: `cmd/business` + `internal/businesscli/` — migrate/ping/create-branch/list-status + `dbms add/list/rm` | complete (rev 2) |
| 5 | `WO-05-business-plugin.md` | Plugin `business` (ping endpoint) + migration file `business_meta` + verifikasi per-DB | complete |
| 6 | `WO-06-infrastructure.md` | Compose `postgres-business`, Dockerfile `bin/business`, env `BUSINESS_DB_*`/`BRANCH_*`, README | complete |
| 7 | `WO-07-verification-commits.md` | Verifikasi AC + commit granular | complete |
| 8 | `WO-08-security-registry.md` | Enkripsi (`dbms_secrets`), TLS, audit, CLI aman | complete (rev 2) |
| 9 | `WO-09-dbms-registry-refactor.md` | **KOREKSI KONTRAK** — refactor DBMS-centric + CLI `--key/--branch-key/--branch-name` + migrasi bentuk lama | ready |

## Urutan Eksekusi & Dependency
```
WO-01 (pool) ──► WO-02 (deps) ──► WO-03 (migration+guard) ──► WO-04 (CLI bisnis) ──► WO-06 (infra)
                                        │                        │
                                        └──► WO-05 (plugin+migr) ┘
                                                                            WO-07 (verifikasi) ← semua
```

## Out of Scope
- Model/entitas bisnis detail (menunggu spesifikasi entitas).
- Middleware `X-Branch-Code` routing HTTP (WO terpisah).
- Sharding/replikasi/HA; secret manager.
- Push ke remote (hanya commit lokal).
- SPA/frontend integration untuk business dashboard.

## Catatan Status
- WO-01..WO-08 sudah diimplementasikan (commit `627f016`..`282dd30`) — **namun menyimpan kredensial per-baris koneksi** (melanggar kontrak D-19/D-20).
- **WO-09 = koreksi kontrak**: refactor DBMS-centric + CLI `--key/--branch-key/--branch-name`. Kerjakan WO-09 sebelum melanjutkan fitur bisnis.

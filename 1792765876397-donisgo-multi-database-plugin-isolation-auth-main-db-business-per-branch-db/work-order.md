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
| 9 | `WO-09-dbms-registry-refactor.md` | **KOREKSI KONTRAK** — refactor DBMS-centric + CLI `--key/--branch-key/--branch-name` + migrasi bentuk lama | complete |
| 10 | `WO-10-business-generator-plugin.md` | **Plugin `business_generator`** — migrasi registry (dbms/dbms_secrets/db_registry/audit), pindah CLI ke `console`, generator scaffolding entity, admin API `/api/admin/dbms` | complete |
| 11 | `WO-11-sudo-like-password-prompt.md` | **Prompt password gaya `sudo`** — baca dari `/dev/tty` (abaikan pipe), fallback `os.Stdin`, error non-TTY yang manusiawi, unit test + docs `-it` | complete |
| 12 | `WO-12-fix-tty-prompt-invisible.md` | **FIX bug** — label prompt tidak muncul: `/dev/tty` dibuka `O_RDONLY` (EBADF, error diabaikan). Buka `O_RDWR`, tangani error tulis, regression test | draft |

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
- WO-01..WO-08 sudah diimplementasikan (commit `627f016`..`282dd30`).
- **WO-09 SELESAI** (commit repo `0afb1ef`..`0b5ecfa`): registry DBMS-centric terverifikasi e2e — `dbms` 2 baris, `db_registry` tanpa kredensial, forward migration legacy teruji, `create-branch --key/--branch-key/--branch-name` sukses, `dbms rm` guard OK, migrate 4 koneksi OK.
- **WO-10 SELESAI** (commit repo `e37d5ab`..`b0258ea`): plugin `business_generator` — DDL registri pindah ke migrasi versioned (target `business_generator` v1 tercatat di `migrations`), CLI console-first (`./bin/console dbms|create-branch|migrate|ping|list-status|generate`), admin API `/api/admin/{dbms,registry}` (200, tanpa kebocoran password), generator entity scaffold.
- **WO-11 SELESAI** (commit repo `ed6cb3b`..`2ba73e8`): prompt password gaya `sudo` — baca `/dev/tty` (pipe diabaikan), fallback `os.Stdin` bila terminal, `errNoTTY` actionable, empty ditolak, seam `ttyOpen` + 6 unit test lulus. Diverifikasi manager: non-TTY error actionable & exit 1; CI path `--password-stdin` tetap jalan.

## Temuan Verifikasi Manager (WO-10/WO-11)

**⚠️ Berulang: binary `bin/` stale.** Dua kali kejadian (WO-10 & WO-11) — `bin/console`/`bin/business` tidak ikut ter-rebuild karena `go build ./...` tidak menulis ke `bin/`, dan `bin/` untracked. Gejala: fitur baru "tidak berfungsi" padahal kode benar. **Wajib ditangani** sebelum makin sering menyesatkan.

## Temuan Verifikasi Manager (WO-10)
Diverifikasi independen: build/vet/test lulus, migrasi tercatat, `dbms rm` guard exit 1, `create-branch` OK, admin API OK, `db_registry` tanpa kredensial, DDL 100% di SQL (0 `CREATE TABLE` di Go).

**Perlu tindak lanjut (non-blocking, belum ada WO):**
1. **Binary stale** — `bin/console`/`bin/business` artefak lokal tidak di-rebuild setelah perubahan source; `bin/` untracked sehingga mudah basi. Pertimbangkan rebuild otomatis di `build.sh`/entrypoint atau dokumentasikan.
2. **Orphan registry rows** — `cbg-jkt`, `cbg-tes`, `taratara` menunjuk database yang sudah di-drop → `migrate up --plugin business` gagal 3/6 (`FATAL: database does not exist`). Perlu dibersihkan (hapus baris atau buat ulang DB).
3. **Hint text basi** — `internal/db/registry.go:147` masih menyarankan `bin/business dbms add`; seharusnya `bin/console dbms add`.
4. **`migrate up` polos** → `no migration paths found for target "core"` dengan **exit 0**; membingungkan karena default command terlihat sukses padahal tidak melakukan apa pun.
5. **Tanpa warning runtime** — `bin/business` hanya menandai DEPRECATED di `Long` help, bukan saat eksekusi.
6. **Plugin `business_generator` belum punya unit test** (0 file `_test.go`).

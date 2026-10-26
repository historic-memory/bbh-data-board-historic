---
title: 'WO-07: Verifikasi & Commit Terstruktur'
description: Verifikasi seluruh AC SPEC (build, guard refusal, isolasi fisik CLI, auth regression) + commit granular per fase.
status: complete
created: "2026-10-26"
updated: "2026-10-26"
---
# WO-07: Verifikasi & Commit Terstruktur

## Goal
Semua AC SPEC (AC-1..AC-18) terverifikasi, dengan commit granular yang rapi.

## Verifikasi (jalankan di container)
1. `go build ./...` + `go test ./internal/db/... ./internal/migration/... ./internal/businesscli/...` lulus.
2. `go run ./cmd/server` start tanpa error; log `Database connected` untuk default; warning bila business off (AC-8).
3. `console migrate up --plugin business` → output per koneksi: `business` + tiap cabang (AC-5).
4. `console migrate up --plugin business --connection default` → error "refusing", SQL tidak jalan — cek tabel `migrations` di business TIDAK bertambah (AC-12).
5. `console migrate up` (polos) → hanya `default`; di psql tabel business/cabang TIDAK berubah (AC-13).
6. `/api/business/ping` return `connected:true` + daftar branches.
7. Auth endpoints masih bekerja (`/api/admin/login`) — tidak ada regresi.
8. `docker compose up -d` → `postgres-business` healthy (AC-9).
9. **Isolasi fisik CLI (Opsi B)**: `env -u DB_HOST -u DB_TYPE -u DB_USER bin/business migrate up` → tetap sukses; bila env `DB_*` kebetulan ada, CLI tidak membacanya (cek log TIDAK ada `Connecting DB [name=default]`).
10. **Keamanan (WO-08)**: `SELECT password_enc FROM dbms_secrets` = envelope AES-GCM (bukan plaintext, AC-15); `bin/business --help` tidak memuat `--password` (AC-17); log app `grep password` = 0 hit; `db_registry_audit` terisi setelah UPDATE (AC-18).

## Commit Granular (urut)
1. `feat(db): tabular connection registry (internal/db/registry.go)` — WO-01
2. `feat(plugins): ServiceDeps.Connections + ConnectionBinder` — WO-02
3. `refactor(migration): extract internal/migration + console guard --connection` — WO-03
4. `feat(business-cli): standalone bin/business CLI (physical DB isolation)` — WO-04
5. `feat(business-plugin): isolated business service plugin` — WO-05
6. `feat(migrations): business postgres migration + per-DB tracking` — WO-05b
7. `build: postgres-business service + bin/business build target` — WO-06
8. `docs: multi-database env vars + README section` — WO-06
9. `feat(security): registry secret encryption + TLS + audit (internal/secrets)` — WO-08
10. `test: pool + migration + guard refusal verification` — WO-07

## Catatan
- TIDAK push ke remote (aturan Manager Agent / kebijakan repo).
- Setelah semua selesai: update status file WO di `_meta.yaml` + `historic save`.

## Acceptance
- [ ] Semua 9 verifikasi lulus.
- [ ] Commit granular selesai, `git status` bersih.
- [ ] Tidak ada push.

## Dependencies
- Semua WO sebelumnya.

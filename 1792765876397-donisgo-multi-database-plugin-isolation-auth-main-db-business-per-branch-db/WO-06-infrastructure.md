---
title: 'WO-06: Infrastruktur — compose, Dockerfile, seed registry'
description: Service postgres-business, Dockerfile build target bin/business, seed awal db_registry via migration, README multi-database. Tanpa file YAML registry.
status: complete
created: "2026-10-26"
updated: "2026-10-26"
---
# WO-06: Infrastruktur — compose, Dockerfile, seed registry

## Goal
Stack lokal mendukung dua server Postgres + binary `bin/business`, terdokumentasi. Konfigurasi koneksi cabang dikelola di **tabel `db_registry`** (bukan YAML).

## Task
1. **Seed registry** (menyertai migration WO-01):
   - Migration `create_db_registry` menyertakan seed idempotent (`ON CONFLICT (name) DO NOTHING`) untuk `default` (role main) dan `business` (role business).
   - Seed membaca nilai dari env saat migrate dijalankan **atau** gunakan placeholder yang di-update manual oleh operator lewat SQL.
2. **`.env.example`**: blok `DB_*` (main/default app — tetap seperti sekarang), blok `BUSINESS_DB_*` (`HOST`, `PORT`, `USER`, `PASSWORD` — dipakai CLI create-branch sebagai default server tujuan). Tidak ada `BRANCH_DB_LIST`/`DB_REGISTRY_PATH`.
3. **`docker-compose.yml`**: service `postgres-business` (image postgres:16-alpine, env POSTGRES_USER/PASSWORD/DB, port host 5434:5432, volume pgdata-business, healthcheck pg_isready, network backend). `app.environment`: tambah `BUSINESS_DB_*`; `depends_on` tambah `postgres-business: {condition: service_healthy}`.
4. **Dockerfile**: build target `bin/business`: `RUN CGO_ENABLED=0 go build -o bin/business ./cmd/business`.
5. **`.gitignore`**: tidak perlu pattern `config/*.yaml` (YAML registry dihapus). Pastikan `.env` tetap di-ignore.
6. **README.md**: section "Multi-Database (Business + Branches)" — arsitektur `db_registry`, cara tambah cabang (`bin/business create-branch`), cara suspend via SQL, contoh command console & CLI.

## Acceptance
- [ ] `docker compose up -d` -> postgres-business healthy; volume terpisah.
- [ ] `bin/business` tersedia di container app.
- [ ] Migration `db_registry` + seed apply sukses; `SELECT * FROM db_registry` menampilkan `default` + `business`.
- [ ] README memuat penjelasan registry tabular dan cara tambah cabang.

## Dependencies
- WO-01 (migration `db_registry`), WO-04 (cmd/business).

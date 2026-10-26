---
title: 'GUIDE: Multi-Database — Registry, Branch, Generator'
description: 'Panduan pengujian end-to-end: daftar DBMS server, daftar cabang (branch), create-branch, migrate, generate entity, dan verifikasi status. Berlaku setelah WO-12 complete.'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
---

# Guide: Multi-Database — Registry, Branch, Generator

Panduan ini untuk **validasi manual** fitur multi-database setelah semua WO (WO-01..WO-12) complete.
Semua command dijalankan dari root repo:

```bash
cd /home/donny/workspaces/bbh-data-board
```

---

## Prasyarat

1. Docker & compose up:
   ```bash
   docker compose up -d
   docker compose ps
   ```
   Harus ada service `app` (idle), `postgres` (host 5433), `postgres-business` (host 5434), `keydb`, `pgadmin`.

2. Binary fresh (WAJIB — `bin/` tidak auto-rebuild oleh `go build ./...`):
   ```bash
   docker compose exec -T app sh -c 'cd /src && go build -o bin/console ./cmd/console && go build -o bin/business ./cmd/business'
   ```

3. Cek state registry saat ini:
   ```bash
   docker compose exec -T app ./bin/console dbms list
   docker compose exec -T app ./bin/console list-status
   ```

---

## 1. Daftar DBMS Server (`dbms list`)

```bash
docker compose exec -T app ./bin/console dbms list
```

**Ekspektasi:** tabel berisi baris registry `dbms`, minimal `main-dbms` dan `business-dbms` (seed dari migrasi `business_generator`), plus server yang kamu daftarkan manual (`localnet`, dll).

---

## 2. Daftarkan DBMS Server Baru (interaktif — prompt sudo-like)

```bash
docker compose exec -it app ./bin/console dbms add \
  --key srv-test --host postgres --port 5432 \
  --user postgres --sslmode disable --label "Server Test"
```

**Ekspektasi (WO-12):**
- Label `Database password (no echo):` **TAMPIL** di terminal.
- Ketik password (mis. `postgres`) — tidak di-echo.
- Sukses → output `dbms registered: key=srv-test ... password=REDACTED`.

**Alternatif CI / non-interaktif:**
```bash
echo 'postgres' | docker compose exec -T app ./bin/console dbms add \
  --key srv-ci --host postgres --port 5432 \
  --user postgres --sslmode disable --label "Server CI" --password-stdin
```
**Ekspektasi:** sukses tanpa prompt.

**Negative test (pipe tanpa flag → harus ditolak):**
```bash
echo 'postgres' | docker compose exec -T app ./bin/console dbms add --key x ...
```
**Ekspektasi:** `Error: password prompt requires a TTY; run with docker compose exec -it ... or pass --password-stdin`, exit 1.

---

## 3. Daftar Cabang (Branch Registry)

```bash
docker compose exec -T app ./bin/console list-status
```

**Ekspektasi:** tabel `db_registry` — per cabang: `key`, `role`, `dbms_key`, `database`, `migrated_version`, `pending`, `dirty`. Minimal ada:
- `default` → main DB (`bbh_data_board` di `main-dbms`)
- `business` → `bbh_business` di `business-dbms`
- branch lain yang sudah dibuat (`jktbaru`, `smg`, dst.)

---

## 4. Buat Cabang Baru

```bash
docker compose exec -T app ./bin/console create-branch \
  --key srv-test \
  --branch-key demo \
  --branch-name "Cabang Demo"
```

**Ekspektasi:**
- Database fisik baru dibuat di server `srv-test` (nama DB: `bbh_cbg_demo` atau pola serupa — cek output).
- Baris baru di `db_registry` dengan `role=branch`, `dbms_key=srv-test`.
- **Tanpa kredensial** di `db_registry` (WO-09: kredensial hanya di `dbms_secrets`).

Verifikasi fisik di DB:
```bash
docker compose exec -T postgres psql -U postgres -lqt | cut -d '|' -f1 | grep -i demo
```

---

## 5. Migrate Plugin `business` ke Cabang

```bash
docker compose exec -T app ./bin/console migrate up --plugin business
```

**Ekspektasi:**
- Migration `business` (tabel bisnis) ter-apply ke semua target yang terdaftar.
- Output menunjukkan target `business`, `demo` (dan branch lain yang ada).
- Setelah selesai:
  ```bash
  docker compose exec -T app ./bin/console list-status
  ```
  **Ekspektasi:** `migrated_version` naik, `pending=0`, `dirty=false` untuk semua baris.

---

## 6. Generate Entity (Plugin `business_generator`)

```bash
docker compose exec -T app ./bin/console generate entity Product \
  --fields "name:string,price:int,active:bool"
```

**Ekspektasi:** 5 file baru di `plugins/business/`:
- `models/product.go`
- `services/entity_product.go`
- `handlers/entity_products.go`
- `migrations/postgres/<stamp>_create_products.up.sql`
- `migrations/postgres/<stamp>_create_products.down.sql`

**Catatan (belum otomatis):**
- Rute **tidak** ter-registrasi otomatis di `plugin.go` — perlu wiring manual.
- Field types yang didukung generator: `string`, `int`, `bool` (cek `generators/entity.go` untuk lainnya).

---

## 7. Migrate Lagi (Apply Tabel `products`)

```bash
docker compose exec -T app ./bin/console migrate up --plugin business
docker compose exec -T app ./bin/console list-status
```

**Ekspektasi:**
- `migrated_version` naik 1 (v1 → v2) untuk semua target `business` plugin.
- `pending=0`, `dirty=false`.

---

## 8. Hapus DBMS Server (Guard)

```bash
docker compose exec -T app ./bin/console dbms rm --key srv-test
```

**Ekspektasi:**
- **GAGAL** bila masih ada `db_registry` yang menunjuk `srv-test` → error jelas + exit 1.
- Berhasil hanya setelah semua registry rows yang menunjuk `srv-test` dihapus (via `dbms rm` guard).

**Negative test tambahan (jangan hapus `main-dbms`/`business-dbms`):**
```bash
docker compose exec -T app ./bin/console dbms rm --key main-dbms
```
**Ekspektasi:** ditolak (guard) karena dipakai oleh `default`/`business`.

---

## 9. Checklist Final

- [ ] `dbms list` menampilkan semua server yang diharapkan.
- [ ] `dbms add` interaktif: label tampil, password tersimpan AES-GCM (cek `dbms_secrets` — envelope `{"v":1,"alg":"AES-GCM",...}`).
- [ ] `dbms add --password-stdin` (CI) sukses.
- [ ] `create-branch` membuat DB fisik + baris registry tanpa kredensial.
- [ ] `migrate up --plugin business` sukses, version naik, pending 0, dirty false.
- [ ] `generate entity` menghasilkan 5 file.
- [ ] `migrate up` setelah generate memasukkan tabel baru (version naik lagi).
- [ ] `dbms rm` guard bekerja (tolak saat masih ada registry rows).

---

## Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| Perubahan source "tidak terasa" | `bin/` stale | Rebuild: `go build -o bin/console ./cmd/console` di dalam container |
| `Error: password prompt requires a TTY` | Jalankan via `exec -T` tanpa flag | Gunakan `-it` atau `--password-stdin` |
| `FATAL: database does not exist` saat migrate | Orphan registry rows (branch dihapus tapi registry masih ada) | Bersihkan baris registry yang menunjuk DB yang sudah di-drop |
| `no migration paths found for target "core"` exit 0 | `migrate up` polos tanpa `--plugin` | Gunakan `migrate up --plugin <name>` (masih tech-debt — lihat WO-10 backlog) |
| `write error: Bad file descriptor` saat prompt | Bug WO-12 (O_RDONLY) | Sudah diperbaiki di commit `6e05497` — pastikan binary rebuild |

---
title: 'WO-04: CLI Business Terpisah (bin/business) — isolasi fisik'
description: 'Binary terpisah: migrate up/down (auto-iterasi cabang dari tabel db_registry), ping, create-branch (pilih server -> CREATE DATABASE -> INSERT db_registry -> auto migrate), list-status.'
status: complete
created: "2026-10-26"
updated: "2026-10-26"
revision: 2
---
# WO-04: CLI Business Terpisah (`bin/business`)

> **REVISI 2 (2026-10-26)** — Kontrak CLI DBMS (SPEC D-20): tambah `dbms add|list|rm`; `create-branch` memakai `--key <dbms_key> --branch-key <name> --branch-name <label>` (**bukan** `--host/--port/--code`). Implementasi koreksi di **WO-09**.

## Goal
Binary terpisah untuk operasi business & cabang dengan **isolasi fisik**: tidak meng-import `internal/plugins`, tidak pernah membuka/menulis ke DB `default` sebagai target. Sumber daftar cabang = **tabel `db_registry` di database utama** (read-only dari sisi CLI untuk routing; write hanya INSERT/UPDATE pada tabel registry saat create-branch).

## Struktur
```
cmd/business/main.go        # thin entry: _ = godotenv.Overload(); rootCmd.Execute()
internal/businesscli/
    root.go                 # root cobra: migrate/ping/create-branch/list-status
    registry.go             # helper: baca/tulis tabel db_registry + dbms via koneksi default
    dbms.go                 # dbms add/list/rm — registrasi server DBMS (acuan key)
    migrate.go              # migrate up/down — loop business + branches dari registry
    ping.go                 # ping --select
    create_branch.go        # create-branch: pilih server -> CREATE DATABASE -> INSERT -> auto migrate
    status.go               # list-status — per-branch version/dirty/pending
```

## Perintah
| Command | Deskripsi |
|---|---|
| `business migrate up` | Satu running -> apply ke `business` + semua cabang aktif (`role='branch' AND status='active'`) dari tabel `db_registry`; non-blocking per cabang; ringkasan; exit non-zero bila ada gagal |
| `business migrate up --connection <name>` | Apply ke satu koneksi saja (tidak auto-iterasi) |
| `business migrate up --select` | Prompt interaktif memilih koneksi target dari registry sebelum eksekusi |
| `business ping --select` | Prompt pilih koneksi -> tampilkan health status |
| `business list-status` | Tabel: name, role, dbms_key, database, status, versi terakhir, pending, dirty |
| `business dbms add --key <key> [--host --port --user --sslmode --label] [--password-stdin]` | Daftarkan server DBMS **sekali** (INSERT `dbms` + `dbms_secrets` terenkripsi) |
| `business dbms list` | Tabel: key, driver, host, port, user, #connections, status |
| `business dbms rm --key <key>` | Hapus server DBMS; **ditolak** bila masih direferensikan `db_registry.dbms_key` |
| `business create-branch --key <dbms_key> --branch-key <name> --branch-name <label> [--db-name <db>] [--password-stdin]` | **Alur lengkap**: resolve server via `--key` -> CREATE DATABASE -> INSERT db_registry (dbms_key) -> **auto migrate** |

## Detail Teknis

### dbms add (registrasi server sekali)
`bin/business dbms add --key business-dbms --host postgres-business --port 5432 --user postgres --sslmode verify-full --label "Server Cabang" --password-stdin`:
1. Validasi `--key` unik + regex `^[a-z0-9-]+$`; tolak duplikat dengan error jelas.
2. INSERT `dbms (key, driver, host, port, user_name, sslmode, label, status)`.
3. `SaveDBMSSecret(main, key, password)` → enkripsi AES-GCM ke `dbms_secrets`.
4. Print ringkasan: `dbms "business-dbms" terdaftar (postgres-business:5432, user=postgres)`.

`dbms list` menampilkan jumlah koneksi yang mengacu (`LEFT JOIN db_registry`).
`dbms rm --key X` → cek `SELECT count(*) FROM db_registry WHERE dbms_key='X'`; jika >0 → **error** `dbms "X" masih dipakai oleh N koneksi; suspend/ganti dulu`.

### create-branch (alur satu perintah)
1. **Resolve server**: `--key <dbms_key>` **wajib** → `LoadDBMS(key)`. Tanpa `--key` → error `--key wajib (daftarkan server lewat dbms add)`. Tidak ada `--host/--port/--user/--password`.
2. Validasi: `--branch-key` unik (`^[a-z0-9-]+$`), `--branch-name` label; DB default `bbh_cbg_<branch-key>` (override `--db-name`). Tolak bila `db_registry.name=<branch-key>` sudah ada.
3. **CREATE DATABASE**: buka maintenance conn (database `postgres`) ke server yang **dituju `--key`** → `CREATE DATABASE <name>` (idempotent). Password dari `dbms_secrets` (dekripsi).
4. **INSERT INTO `db_registry`** (hanya logis): `{ name: <branch-key>, role: 'branch', dbms_key: <key>, database_name: <name>, label: <branch-name>, status: 'active' }` — **tanpa** host/port/user/password. **Tidak perlu** INSERT secrets per cabang (rahasia sudah di level `dbms`).
5. **Auto migrate**: buka koneksi baru -> jalankan `internal/migration.MigrateFiles` (semua file migrasi business) -> verifikasi tabel `migrations` terisi.
6. Output ringkasan: `dbms=<key>`, database dibuat, entri registry, jumlah migrasi applied.

### Registry access
- CLI baca `db_registry` lewat koneksi `default` (**read-only**, hanya SELECT untuk routing).
- INSERT/UPDATE `db_registry` hanya terjadi di `create-branch` (dan opsional `disable-branch`).
- `GETGORM(name)` untuk koneksi cabang lewat `internal/db.Pool`.
- **TIDAK meng-import `internal/plugins`**.

### Prompt interaktif (`--select`)
`bufio.Scanner`, tanpa dependency eksternal:
```
? Pilih database target:
   1. [business]  business       dbms=business-dbms  bbh_business        postgres-business:5432
   2. [branch]    cbg-jakarta    dbms=business-dbms  bbh_cbg_jakarta     postgres-business:5432  "Cabang Jakarta Pusat"
   3. [branch]    cbg-surabaya   dbms=business-dbms  bbh_cbg_surabaya    postgres-business:5432  "Cabang Surabaya"
> _  (nomor atau nama, Ctrl+C cancel)
```

### Guard isolasi
- Sebelum eksekusi, validasi: `role` entri target harus `business`/`branch`. Entri `role='main'` -> tolak dengan error (`refusing to run business migration against role=main connection "default"`).
- Tidak ada code path yang menjalankan migrasi ke koneksi `default`.

## Acceptance
- [ ] `go build ./...` + `go test ./internal/businesscli/...` lulus.
- [ ] Target migrate = `business` + semua `role='branch' AND status='active'` dari `db_registry`.
- [ ] Registry kosong/tabel belum ada -> error jelas (bukan panic).
- [ ] `--select` menampilkan semua koneksi aktif + label dari tabel.
- [ ] `create-branch --key <dbms_key> --branch-key <name> --branch-name <label>` selesai sukses -> DB dibuat, entri registry (tanpa host/port/user) ada, **dan schema sudah ter-migrasi penuh** tanpa perintah terpisah.
- [ ] `create-branch` tanpa `--key` -> error jelas `--key wajib`; `--key` tidak terdaftar -> error.
- [ ] `dbms add` membuat satu baris `dbms` + satu baris `dbms_secrets`; password terenkripsi.
- [ ] `dbms list` menampilkan jumlah koneksi yang mengacu per server.
- [ ] `dbms rm` ditolak bila masih direferensikan `db_registry`.
- [ ] `migrate up --connection default` ditolak (guard).

## Dependencies
- WO-01 (`internal/db/registry.go`, Pool, migration `db_registry`), WO-03 (`internal/migration.MigrateFiles`).

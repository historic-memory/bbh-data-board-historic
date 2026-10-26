---
title: 'WO-04: CLI Business Terpisah (bin/business) — isolasi fisik'
description: 'Binary terpisah: migrate up/down (auto-iterasi cabang dari tabel db_registry), ping, create-branch (pilih server -> CREATE DATABASE -> INSERT db_registry -> auto migrate), list-status.'
status: complete
created: "2026-10-26"
updated: "2026-10-26"
---
# WO-04: CLI Business Terpisah (`bin/business`)

## Goal
Binary terpisah untuk operasi business & cabang dengan **isolasi fisik**: tidak meng-import `internal/plugins`, tidak pernah membuka/menulis ke DB `default` sebagai target. Sumber daftar cabang = **tabel `db_registry` di database utama** (read-only dari sisi CLI untuk routing; write hanya INSERT/UPDATE pada tabel registry saat create-branch).

## Struktur
```
cmd/business/main.go        # thin entry: _ = godotenv.Overload(); rootCmd.Execute()
internal/businesscli/
    root.go                 # root cobra: migrate/ping/create-branch/list-status
    registry.go             # helper: baca/tulis tabel db_registry via koneksi default (read-only untuk routing)
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
| `business list-status` | Tabel: name, role, database, status, versi terakhir, pending, dirty |
| `business create-branch --code X [--label "Z"]` | **Alur lengkap**: pilih server -> CREATE DATABASE -> INSERT db_registry -> **auto migrate** |

## Detail Teknis

### create-branch (alur satu perintah)
1. **Prompt pilih server tujuan**: tampilkan daftar server dari env + entri `role='business'` di registry. Flag `--host/--port/--user` untuk skip prompt (password **tidak** lewat flag — lihat WO-08). Default ambil `BUSINESS_DB_HOST/PORT/USER`.
2. Validasi: code unik & regex `^[a-z0-9_]+$`; nama DB default `bbh_cbg_<code>` (override `--db-name`). Cek `db_registry` -> tolak bila `name=cbg-<code>` sudah ada.
3. **CREATE DATABASE**: buka maintenance conn (database `postgres`) ke server terpilih -> `CREATE DATABASE <name>` (idempotent: skip bila sudah ada).
4. **INSERT INTO `db_registry`** (metadata saja) + `SaveRegistrySecret()` (password terenkripsi ke `db_registry_secrets`, lihat WO-08): `{ name: cbg-<code>, role: 'branch', host, port, user_name, database_name: <name>, sslmode: 'verify-full', label, status: 'active' }`.
   - Password diperoleh via `--password-stdin` atau prompt `term.ReadPassword` (tanpa echo). Flag `--password` **tidak ada**.
5. **Auto migrate**: buka koneksi baru ke `<name>` -> jalankan `internal/migration.MigrateFiles` (semua file migrasi business) -> verifikasi tabel `migrations` terisi.
6. Output ringkasan: database dibuat + entri registry + jumlah migrasi applied.

### Registry access
- CLI baca `db_registry` lewat koneksi `default` (**read-only**, hanya SELECT untuk routing).
- INSERT/UPDATE `db_registry` hanya terjadi di `create-branch` (dan opsional `disable-branch`).
- `GETGORM(name)` untuk koneksi cabang lewat `internal/db.Pool`.
- **TIDAK meng-import `internal/plugins`**.

### Prompt interaktif (`--select`)
`bufio.Scanner`, tanpa dependency eksternal:
```
? Pilih database target:
   1. [business]  business       bbh_business        postgres-business:5432
   2. [branch]    cbg-jakarta    bbh_cbg_jakarta     postgres-business:5432  "Cabang Jakarta Pusat"
   3. [branch]    cbg-surabaya   bbh_cbg_surabaya    postgres-business:5432  "Cabang Surabaya"
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
- [ ] `create-branch` selesai sukses -> DB dibuat, entri registry ada, **dan schema sudah ter-migrasi penuh** (tabel `migrations` terisi) tanpa perintah terpisah.
- [ ] `migrate up --connection default` ditolak (guard).

## Dependencies
- WO-01 (`internal/db/registry.go`, Pool, migration `db_registry`), WO-03 (`internal/migration.MigrateFiles`).

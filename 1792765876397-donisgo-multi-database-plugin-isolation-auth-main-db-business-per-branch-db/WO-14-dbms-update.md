---
title: 'WO-14: Command `dbms update` — ubah metadata server tanpa retype password'
description: 'Partial update metadata dbms (label/host/port/user/sslmode/pool/status) per key. Password OPSIONAL: kosong = pertahankan secret lama. Tampilkan diff before→after. Key tidak bisa diubah (identitas).'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
revision: 0
---

# WO-14: Command `dbms update` — Partial Update Metadata Server

## Latar Belakang

`dbms add` sebenarnya **sudah upsert** (`ON CONFLICT (key) DO UPDATE` di `internal/db/registry.go:183`), jadi update metadata secara teknis bisa. **Masalahnya:** `readPassword()` selalu dipanggil di `newDBMSAddCmd()` dan menolak input kosong — sehingga mengganti label saja **memaksa retype password**.

`dbms update` menghilangkan friksi itu: hanya field yang **benar-benar diberikan** yang berubah, password dipertahankan kecuali diminta rotasi.

## Aturan Kontrak (keputusan user)

| Aspek | Aturan |
|---|---|
| Semantik | **Partial update** — hanya flag yang diberikan yang berubah |
| Password | **Opsional.** Tidak diberikan → secret lama dipertahankan. `--rotate-password` untuk memicu prompt/no-echo. `--password-stdin` didukung untuk CI (hanya bila rotasi) |
| `key` | **Tidak bisa diubah** — identitas server, dirujuk `db_registry.dbms_key`. Ganti key = hapus + daftar ulang (ditolak dengan pesan jelas) |
| Diff | Tampilkan `field: lama → baru` per field yang berubah sebelum commit |
| No-op | Jika tidak ada flag yang diberikan → error: "nothing to update; specify at least one field flag" |
| Not found | Key tidak ada → error jelas `dbms "x" is not registered; add it with: bin/console dbms add --key x` |
| Status | `--status active\|suspended` boleh diubah |

## CLI Contract

```bash
# Ganti label saja — tanpa password
bin/console dbms update --key srv-test --label "Server Produksi Jakarta"

# Koreksi port + user
bin/console dbms update --key srv-test --port 5433 --user postgres

# Ubah pool + status (maintenance)
bin/console dbms update --key srv-test --max-open-conns 50 --max-idle-conns 10
bin/console dbms update --key srv-test --status suspended

# Rotasi password (interaktif no-echo)
bin/console dbms update --key srv-test --rotate-password

# Rotasi password (CI)
echo 'newpass' | bin/console dbms update --key srv-test --rotate-password --password-stdin

# Multiple field sekaligus
bin/console dbms update --key srv-test --host postgres-new --port 5433 --sslmode verify-full
```

### Output format

```
dbms updated: key=srv-test
  label:        "Server Test" → "Server Produksi Jakarta"
  port:         5432 → 5433
  sslmode:      disable → verify-full
  password:     unchanged (use --rotate-password to change)
```

Jika hanya satu field:
```
dbms updated: key=srv-test
  label:        "Server Test" → "Server Produksi Jakarta"
  password:     unchanged
```

Tanpa perubahan:
```
dbms unchanged: key=srv-test (no field differs)
```

## Implementasi

### `internal/db/registry.go`

Tambah fungsi partial update:

```go
// DBMSUpdate carries only the fields to change (nil = leave unchanged).
// Using pointers distinguishes "not provided" from "explicitly set to empty".
type DBMSUpdate struct {
    Driver       *string
    Host         *string
    Port         *int
    UserName     *string
    SSLMode      *string
    MaxOpenConns *int
    MaxIdleConns *int
    Label        *string
    Status       *string
}

// UpdateDBMS applies a partial update. It returns the previous entry and the
// updated entry so callers can render a diff. Password is NOT touched here —
// use SaveDBMSSecret/RotateDBMSSecret separately when rotating.
//
// Implementation note: read-modify-write inside a transaction, or a single
// UPDATE with COALESCE(NULLIF(?, ''), column) per field. Prefer read-modify-write
// so the "before" snapshot is exact and the diff is reliable.
func UpdateDBMS(main *gorm.DB, key string, upd *DBMSUpdate) (before, after *DBMSEntry, err error)
```

**Keputusan implementasi:** pilih **read-modify-write dalam transaksi** (`main.Transaction(...)`), bukan `COALESCE` dinamis, karena:
1. Butuh `before` snapshot untuk diff — sudah didapat dari read awal.
2. `COALESCE(NULLIF(...))` tidak bisa mengekspresikan "set ke NULL" (misal mengosongkan `label`), sedangkan pointer bisa.
3. Lebih mudah ditest.

### `commands.go`

```go
func newDBMSUpdateCmd(p *Plugin) *cobra.Command {
    var (
        key, host, user, sslmode, label, driver, status string
        port, maxOpen, maxIdle                           int
        rotatePassword, passwordStdin                    bool
    )
    cmd := &cobra.Command{
        Use:   "update",
        Short: "Update a registered DBMS server's metadata (password optional)",
        RunE: func(cmd *cobra.Command, args []string) error {
            // 1. Reject if no field flag changed
            // 2. Build *DBMSUpdate using cmd.Flags().Changed(...) for pointers
            // 3. Load current entry (before) → verify exists
            // 4. Apply UpdateDBMS
            // 5. If rotatePassword: readPassword(passwordStdin) → RotateDBMSSecret
            // 6. Render diff
        },
    }
    cmd.Flags().StringVar(&key, "key", "", "server key to update (required)")
    cmd.Flags().StringVar(&driver, "driver", "", "driver (postgres)")
    cmd.Flags().StringVar(&host, "host", "", "server host")
    cmd.Flags().IntVar(&port, "port", 0, "server port")
    cmd.Flags().StringVar(&user, "user", "", "server user")
    cmd.Flags().StringVar(&sslmode, "sslmode", "", "TLS mode")
    cmd.Flags().StringVar(&label, "label", "", "human-readable label")
    cmd.Flags().StringVar(&status, "status", "", "active|suspended")
    cmd.Flags().IntVar(&maxOpen, "max-open-conns", 0, "pool max open connections")
    cmd.Flags().IntVar(&maxIdle, "max-idle-conns", 0, "pool max idle connections")
    cmd.Flags().BoolVar(&rotatePassword, "rotate-password", false, "re-seal the password (prompt or --password-stdin)")
    cmd.Flags().BoolVar(&passwordStdin, "password-stdin", false, "read the new password from stdin (with --rotate-password)")
    return cmd
}
```

**Catatan penting:** `cmd.Flags().Changed(name)` adalah satu-satunya cara benar untuk membedakan "tidak diberikan" vs "diberikan nilai default". Jangan pakai `if label != ""` karena user mungkin ingin **mengosongkan** label.

**Validasi `--password-stdin`:** hanya valid bersama `--rotate-password`. Jika `--password-stdin` tanpa `--rotate-password` → error actionable.

**Pool invalidation:** setelah update berhasil, wajib panggil `db.InvalidateAll()` (atau invalidation per-name) karena DSN berubah — pool lama masih memegang host/port lama.

### `services/admin_service.go`

```go
func (s *AdminService) UpdateDBMS(key string, upd *db.DBMSUpdate, newPassword string) (*db.DBMSEntry, *db.DBMSEntry, error)
```
Delegate ke `db.UpdateDBMS` + `db.RotateDBMSSecret` bila `newPassword != ""`, lalu `db.InvalidateAll()`.

### `handlers/admin_handler.go`

`PUT /api/admin/dbms/:key` **sudah ada** (baris 78) tetapi saat ini memanggil `AddDBMS` (upsert penuh). Perubahan:
- Payload partial (field opsional, `*string`/`*int`)
- Hanya field yang hadir di JSON yang diupdate
- `password` opsional; bila kosong/tidak ada → tidak sentuh secret

Response:
```json
{
  "key": "srv-test",
  "changed": ["label", "port"],
  "before": {"label": "Server Test", "port": 5432},
  "after":  {"label": "Server Produksi Jakarta", "port": 5433}
}
```
Password tidak pernah masuk response.

## Unit Tests

1. `TestUpdateDBMSLabelOnly` — hanya label berubah; host/port/user/sslmode tetap; secret utuh.
2. `TestUpdateDBMSNoFields` — `DBMSUpdate{}` kosong → error "nothing to update".
3. `TestUpdateDBMSUnknownKey` — key tidak ada → error dengan hint `bin/console dbms add`.
4. `TestUpdateDBMSPasswordPreserved` — update metadata tidak menyentuh `dbms_secrets` (bandingkan `password_enc` sebelum vs sesudah).
5. `TestUpdateDBMSRotatePassword` — rotasi mengubah `password_enc` dan password baru bisa decrypt + dipakai konek.
6. `TestUpdateDBMSStatusSuspended` — `--status suspended` mengubah kolom status.
7. `TestUpdateDBMSDiffRendered` — output memuat `label: "lama" → "baru"`.
8. `TestUpdateDBMSKeyImmutable` — error bila user mencoba `--key baru` pada satu-satunya argumen (tidak ada flag `--new-key`).

## Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | `dbms update --key k --label "x"` berubah tanpa prompt password | manual + test |
| 2 | Password lama **tidak berubah** saat update metadata (`password_enc` identik) | query DB sebelum/sesudah |
| 3 | `--rotate-password` memunculkan prompt no-echo & secret berubah | manual (TTY) + test |
| 4 | `--password-stdin` tanpa `--rotate-password` → error actionable | test |
| 5 | Tanpa flag apa pun → error "nothing to update" | test |
| 6 | Key tidak dikenal → error + hint `bin/console dbms add` | test |
| 7 | Diff `lama → baru` tampil per field | review output |
| 8 | `dbms update --status suspended` mengubah status | test |
| 9 | Pool invalidated setelah update (perubahan host langsung berlaku) | test konek setelah update |
| 10 | `PUT /api/admin/dbms/:key` partial, tanpa kebocoran password | curl test |

## File yang Diubah (estimasi)

| File | Perubahan |
|---|---|
| `internal/db/registry.go` | Tambah `DBMSUpdate` struct + `UpdateDBMS()` |
| `plugins/business_generator/commands.go` | Tambah `newDBMSUpdateCmd`, daftarkan di `NewDBMSCmd` |
| `plugins/business_generator/services/admin_service.go` | Tambah `UpdateDBMS()` wrapper |
| `plugins/business_generator/handlers/admin_handler.go` | Ubah `UpdateDBMS` handler → partial semantics |
| `plugins/business_generator/commands_test.go` | 6–8 test baru |

## Catatan Terkait

- **`dbms add` tetap dipertahankan** sebagai "create or replace full record". `dbms update` untuk perubahan parsial.
- **Tidak ada migrasi/schema baru** — semua kolom sudah ada di `dbms`.
- **Jangan** menambahkan flag `--new-key`/rename. Rename key tidak didukung: itu artinya `db_registry.dbms_key` harus ikut di-update (FK) → risiko tinggi, di luar scope.
- `internal/db/registry.go:147` hint basi (`bin/business`) sebaiknya diperbaiki di WO ini karena satu file & satu baris.

---
title: 'WO-13: Command `dbms test` — diagnostik koneksi bertingkat'
description: 'Uji konektivitas DBMS server dari A sampai Z: resolusi secret → TCP dial → TLS → auth/startup → ping DB per-cabang. Laporkan status tiap stage + durasi ms + alasan spesifik kegagalan.'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
revision: 0
---

# WO-13: Command `dbms test` — Diagnostik Koneksi Bertingkat

## Latar Belakang (dari user request)
Command `ping` sudah ada tetapi:
- Targetnya hanya koneksi `db_registry` (business/branch), **bukan server `dbms`**
- Alasan kegagalan **dibuang**: `STATUS=down` tanpa sebab → tidak menjawab "entah kenapa"

`dbms test` mengisi celah ini: uji **server register** dari akar sampai ke ujung, dengan laporan **bertingkat** sehingga user tahu persis di mana & mengapa koneksi gagal.

## Desain: Staged Diagnostic Probe

Setiap uji terbagi 4 stage serial:

| Stage | Yang diuji | Contoh output sukses | Contoh output gagal |
|---|---|---|---|
| 1 | Resolve secret dari `dbms_secrets` | ✅ resolved (12 chars masked) | ❌ secret missing / cannot decrypt (APP_KEY mismatch?) |
| 2 | TCP dial `host:port` dengan timeout 5s | ✅ dial ok (12ms) | ❌ connection refused (server down?), no such host (DNS?), i/o timeout (firewall?) |
| 3 | TLS handshake jika sslmode != disable | ✅ SSL ok (34ms) | ❌ certificate signed by unknown authority / sslmode=verify-full tapi server menolak |
| 4 | Auth + startup (Ping ke DB target) | ✅ ping ok (8ms) | ❌ password authentication failed for user "x" / database "y" does not exist |

Output akhir: tabel ringkas + exit 1 bila ada yang gagal.

**Perlu fitur baru di `internal/db`:**
- `TestDBMS(main *gorm.DB, key string) (*DBTestResult, error)` — main function orchestrate 4 stages
- Struct `DBTestResult` { Key, ServerOK bool, Branches []BranchTest }
- Struct `BranchTest` { RegistryName, Database, OK bool, Error string, Duration }

## CLI Contract

```bash
# Uji satu server
bin/console dbms test --key srv-test

# Uji semua server
bin/console dbms test --all
```

### Output format (success)
```
KEY           SERVER   BRANCHES TESTED   OVERALL
srv-test      ✅ ok    3/3                ✅ PASS
  ├─ Stage 1: ✅ resolved (***-postgres)
  ├─ Stage 2: ✅ dial ok (14ms)
  ├─ Stage 3: ✅ SSL ok (37ms)
  └─ Stage 4: ✅ ping ok (9ms)
     jktbaru      ✅ (12ms)    smg          ✅ (18ms)    business       ✅ (8ms)
```

### Output format (server fail, stages ditunjukkan)
```
KEY           SERVER   BRANCHES TESTED   OVERALL
srv-old       ❌ fail  0/0                ❌ FAIL
  ├─ Stage 1: ✅ resolved (***-userX)
  ├─ Stage 2: ❌ dial refused - connection refused (server down?) ← ROOT CAUSE
  └─ Stage 3: skipped (stage 2 failed)
     (no branches tested — cannot reach server)
```

### Output format (server pass, some branch fail)
```
KEY           SERVER   BRANCHES TESTED   OVERALL
srv-shared    ✅ ok    3/3                ⚠️ DEGRADED
  ├─ Stage 1: ✅ resolved (***-admin)
  ├─ Stage 2: ✅ dial ok (11ms)
  ├─ Stage 3: ✅ SSL ok (28ms)
  └─ Stage 4: ✅ ping ok (7ms)
     jktbaru      ✅ (11ms)    smg          ❌ (database does not exist) ← CRITICAL
                    business       ✅ (9ms)
```

Legend: ✅ pass, ❌ fail, ⚠️ degraded, skipped = stage dilewati karena stage sebelumnya gagal.

## Fitur Baru di `internal/db/registry.go`

```go
// BranchTest captures the result of testing one logical connection.
type BranchTest struct {
    Name     string // registry name
    DB       string // database name
    OK       bool
    Error    string // reason for failure (empty if OK)
    Duration time.Duration
}

// DBTestResult captures the complete outcome of testing a DBMS server.
type DBTestResult struct {
    Key          string
    ServerOK     bool            // all 4 server stages passed
    Stages       map[string]*StageResult // stage name → result
    Branches     []BranchTest        // one per db_registry row pointing to this key
}

type StageResult struct {
    Name     string // "secret", "dial", "tls", "ping"
    OK       bool
    Error    string
    Duration time.Duration
}
```

### New function: `TestDBMS`

```go
func TestDBMS(main *gorm.DB, key string) (*DBTestResult, error) {
    // 1. Load server + password -> Stage 1
    // 2. Dial TCP -> Stage 2
    // 3. TLS handshake -> Stage 3
    // 4. Ping database -> Stage 4
    // 5. Load registry rows for key -> iterate branches -> ping each
    // Return composite result.
}
```

> Note: Untuk TCP+TLS+auth, kita butuh DSN builder. DSN build logic bisa dipakai ulang dari existing pool code di `internal/db/gorm.go` (lihat `buildDSN` atau setara). Atau tambahkan helper kecil `buildDSN(entry *DBMSEntry, dbName string) string` kalau belum ada eksplisit.

### Helper tambahan (jika belum ada)

```go
// BuildDSN constructs a PostgreSQL connection URI for a given server.
func BuildDSN(server *DBMSEntry, dbName, password string) string
```

Jika DSN sudah bisa di-bangun via existing code path, cukup wrap dengan context+timeout wrapper.

## Perubahan `commands.go`

Tambah command group baru ke `cmd.AddCommand(newDBMSTestCmd(p))`:

```go
func newDBMSTestCmd(p *Plugin) *cobra.Command {
    var key string
    var all bool
    cmd := &cobra.Command{
        Use:   "test",
        Short: "Test connectivity to a registered DBMS server with staged diagnostics",
        RunE: func(cmd *cobra.Command, args []string) error {
            if !all && key == "" { return errors.New("specify --key or --all") }
            if all && key != "" { return errors.New("cannot specify both --key and --all") }
            // ... iterate + print table
        },
    }
    cmd.Flags().StringVar(&key, "key", "", "server key to test")
    cmd.Flags().BoolVar(&all, "all", false, "test all registered servers")
    _ = cmd.MarkFlagsMutuallyExclusive("key", "all")
    return cmd
}
```

## Perilaku `dbms add` — auto-test (non-blocking)

Di `newDBMSAddCmd()`, setelah `AddDBMS()` sukses:
```go
if err := p.Service().AddDBMS(entry, password); err != nil {
    return err
}
fmt.Printf("dbms registered: ...\n")
// Optional warning: test connection immediately
result, err := p.Service().TestDBMS(key)
if err != nil || !result.ServerOK {
    fmt.Fprintf(cmd.ErrOrStderr(), "⚠️  WARNING: server connection test failed: %v\n", 
        rootCause(result.Stages))
}
```
Ini non-blocking: tetap tersimpan meski test gagal. User dapat info segera.

## Backend Admin API

### `POST /api/admin/dbms/test/:key`
Request body kosong. Response:
```json
{
  "key": "srv-test",
  "overall": "pass",
  "stages": [
    {"name": "secret", "ok": true, "error": "", "duration_ms": 1},
    {"name": "dial", "ok": true, "error": "", "duration_ms": 12},
    {"name": "tls", "ok": true, "error": "", "duration_ms": 37},
    {"name": "ping", "ok": true, "error": "", "duration_ms": 8}
  ],
  "branches": [
    {"name": "jktbaru", "database": "bbh_cbg_jktbaru", "ok": true, "error": "", "duration_ms": 11}
  ]
}
```

Status overall: `"pass"` / `"fail"` / `"degraded"`.

### GET /api/admin/dbms?test=true (opsional)
Tidak wajib di WO ini. Bisa ditambahkan nanti saat dashboard dibuat.

## Unit Tests

1. **Stage 1 — Secret resolution:**
   - Known key → OK
   - Unknown key → wrapped error ("not found")

2. **Stage 2 — TCP refusal:** Mock listener (GORM mock) atau gunakan port yang closed.

3. **Stage 3 — TLS rejection:** Test dengan `sslmode=disable` (skip TLS) vs `sslmode=require` against localhost.

4. **Stage 4 — Auth/Ping failure:** Test terhadap DB yang tidak ada → `database does not exist`.

5. **Staged cascade:** Jika stage 2 gagal, stage 3+4 ditandai `skipped` (bukan `failed`).

## Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | `dbms test --key <k>` berhasil, output menampilkan stage diagram dengan ✅/❌ | manual test |
| 2 | `dbms test --all` berjalan, tabel multi-row | manual test |
| 3 | Root cause ditampilkan secara aksiabel (contoh: "connection refused" bukan sekadar "fail") | review output |
| 4 | Cascading: stage 2 gagal → stage 3/4 "skipped" (bukan "fail") | test kasus |
| 5 | Semua branch `db_registry` untuk key tersebut diuji sebagai baris terpisah | test dengan ≥2 branch |
| 6 | `dbms add` otomatis jalankan test, warning tampil jika gagal | test `dbms add → test` |
| 7 | POST /api/admin/dbms/test/:key returns valid JSON response | curl test |
| 8 | `go test ./plugins/business_generator/...` lulus | CI |

## File yang Diubah (estimasi)

| File | Perubahan |
|---|---|
| `internal/db/registry.go` | Tambah: `DBTestResult`, `BranchTest`, `StageResult`, `TestDBMS`, `BuildDSN` |
| `internal/db/secret.go` | Pastikan `LoadDBMSPassword` accessible (public) |
| `plugins/business_generator/commands.go` | Tambah `newDBMSTestCmd`, edit `newDBMSAddCmd` (+auto-test warning) |
| `plugins/business_generator/services/admin_service.go` | Tambah `TestDBMS` |
| `plugins/business_generator/handlers/admin_handler.go` | Tambah `TestDBMS` handler (`POST /api/admin/dbms/test/:key`) |
| `plugins/business_generator/commands_test.go` | 3–5 test baru |
| `plugins/business_generator/plugin.go` | Register command test |

## Out of Scope (deferred)

- Autopsy detail (stack trace, raw packet capture). Cukup error message postgres/driver.
- Retry logic (misalnya retry 3x sebelum rapor fail). Bisa ditambahkan nanti.
- `--select` prompt (interaktif pilih dari daftar): tidak perlu, `--key` sudah sufficient.
- Integration test end-to-end (butuh fixture multi-server; defer ke manual verification).
- Dashboard/web view: API JSON sudah cukup.

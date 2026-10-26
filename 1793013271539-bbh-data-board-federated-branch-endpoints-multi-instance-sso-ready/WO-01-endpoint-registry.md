---
title: 'WO-01: Endpoint Registry — tabel endpoints + CLI + admin API'
description: 'Fondasi federasi: registry endpoint cabang di DB lokal pusat (migrasi versioned), CLI endpoint add/list/rm/test/use, admin API, penyimpanan token tersegel AES-GCM, audit request lintas instance. Prasyarat: pulihkan command migrate (internal/migration hilang saat destroy). Scope: admin saja (D-8).'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
revision: 2
---

# WO-01: Endpoint Registry

**SPEC acuan:** `SPEC.md` di topic ini (D-1..D-15, AC-1..AC-22)
**Fase:** 1 (Endpoint Registry) — bagian registry saja. Routing/captive portal = WO-02; SSO (admin-only) = WO-03.

**Cakupan pengguna:** **admin saja** (D-8 scope-down). Tabel `members` tidak disentuh WO ini.

## Kondisi Awal Kode (terverifikasi 2026-10-26 setelah destroy)

| Komponen | Status | Implikasi |
|---|---|---|
| `plugins/auth` (+ JWT, middleware) | ✅ ada | Fondasi SSO fase 3 (admin-only, D-8) |
| `internal/secrets` (AES-GCM) | ✅ ada | Pakai ulang untuk `token_enc` |
| `plugins/*/migrations/postgres/*.sql` | ✅ pola ada (auth) | Ikuti pola ini untuk `endpoints` |
| `internal/migration` + `console migrate` | ❌ **hilang** | **Prasyarat: buat ulang minimal** |
| `internal/db/registry.go` | ❌ hilang | Tidak dibutuhkan (federasi tak akses DB cabang) |
| `go build ./...`, `go test ./...` | ✅ lulus | Titik mulai bersih |

---

## Bagian A — Prasyarat: pulihkan migrasi versioned (minimal)

`internal/migration` ikut terhapus bersama topic lama. WO ini **butuh** migrasi versioned (D-3: skema milik DB utama wajib lewat migrasi, bukan DDL bootstrap).

**Buat ulang secukupnya, jangan menghidupkan lagi mesin migrasi lama yang kompleks:**

- `internal/migration/migrator.go` — baca file dari direktori plugin, jalankan `.up.sql`/`.down.sql` berurutan berdasarkan nama, catat ke tabel `migrations` (kolom minimal: `target`, `version`, `name`, `applied_at`) dan `migration_targets` (`target`, `dirty`).
- `ConsoleCommands`: `migrate up`, `migrate down`, `migrate list` dengan flag `--plugin <id>` (default: `all`).
- **Jangan** menghidupkan: guard `--connection`, topo-sort antar plugin, dukungan MySQL. Itu kebutuhan model lama (multi-DB). Kalau nanti perlu, WO terpisah.

**DoD Bagian A:**
- `./bin/console migrate up --plugin auth` pada DB kosong membuat tabel auth.
- `./bin/console migrate down --plugin auth` menghapusnya.
- `migrate list` menampilkan versi terpasang per target.

> Catatan: tabel `migrations`/`migration_targets` **sudah ada** di `bbh_data_board` dari sisa proses lama (kita hanya rollback isinya, bukan drop DB). Jadi kode baru harus **kompatibel atau melakukan `CREATE TABLE IF NOT EXISTS`** dengan bentuk yang sama agar tidak konflik dengan tabel yang sudah ada.

---

## Bagian B — Skema `endpoints` + `endpoint_audit` (migrasi versioned)

Plugin pemilik: perluas plugin yang ada atau buat plugin baru `endpoint` (rekomendasi: **plugin baru `endpoint`** — tanggung jawab terpisah, pola sama dengan plugin lain).

```
plugins/endpoint/plugin.go
plugins/endpoint/migrations/postgres/2026_10_27_090000_create_endpoints.up.sql
plugins/endpoint/migrations/postgres/2026_10_27_090000_create_endpoints.down.sql
```

**Isi `.up.sql`:**

```sql
CREATE TABLE endpoints (
    id            BIGSERIAL PRIMARY KEY,
    key           TEXT UNIQUE NOT NULL,
    label         TEXT,
    base_url      TEXT NOT NULL,
    purpose       TEXT NOT NULL DEFAULT 'business', -- hanya core business; auth TIDAK di sini
    auth_kind     TEXT NOT NULL DEFAULT 'none',   -- none | bearer
    token_enc     TEXT,
    timeout_ms    INT  NOT NULL DEFAULT 10000,
    status        TEXT NOT NULL DEFAULT 'active', -- active | suspended
    is_local      BOOLEAN NOT NULL DEFAULT FALSE,
    is_active     BOOLEAN NOT NULL DEFAULT FALSE, -- portal aktif saat ini (satu baris; dikelola transaksi, D-6)
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Batas integritas: maksimal SATU endpoint aktif sekaligus (D-6)
CREATE UNIQUE INDEX uq_endpoints_one_active ON endpoints (is_active) WHERE is_active;

CREATE TABLE endpoint_audit (
    id            BIGSERIAL PRIMARY KEY,
    endpoint_key  TEXT NOT NULL,
    request_id    TEXT NOT NULL,
    method        TEXT NOT NULL,
    path          TEXT NOT NULL,
    status_code   INT,
    duration_ms   INT,
    error_class   TEXT,   -- unreachable | timeout | unauthorized | bad_gateway | NULL
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_endpoint_audit_key_time ON endpoint_audit (endpoint_key, created_at DESC);

-- Seed: instance sendiri selalu tersedia dan aktif secara default
INSERT INTO endpoints (key, label, base_url, purpose, auth_kind, is_local, is_active, status)
VALUES ('local', 'Instance ini', 'http://127.0.0.1:8082', 'business', 'none', TRUE, TRUE, 'active')
ON CONFLICT (key) DO NOTHING;
```

**Isi `.down.sql`:** `DROP TABLE IF EXISTS endpoint_audit; DROP TABLE IF EXISTS endpoints;`

**Aturan wajib:**
- **Nol `CREATE TABLE` di kode Go** (AC-10).
- `token_enc` **wajib tersegel** via `internal/secrets.EncryptBlob` sebelum disimpan; plaintext tidak pernah masuk DB.
- `key` immutable (tidak ada rename).

---

## Bagian C — CLI

```
bin/console endpoint add  --key <k> --base-url <url> [--label <l>] [--auth-kind none|bearer] [--token-stdin] [--timeout-ms 10000]
bin/console endpoint list [--json]
bin/console endpoint rm   --key <k>
bin/console endpoint test --key <k> [--all] [--timeout-ms N]
bin/console endpoint use  <k>       # set endpoint aktif (mode kerja)
bin/console endpoint active         # tampilkan endpoint aktif
```

### Kontrak perilaku

| Command | Aturan |
|---|---|
| `add` | Upsert? **Tidak** — `add` menolak key duplikat; perubahan lewat `update` (WO terpisah). Token hanya via `--token-stdin`, **tidak ada `--token`** (pola sama dengan WO-11/12: rahasia tidak lewat argv). |
| `add` auto-test | Setelah tersimpan, jalankan `test` sekali. **Warning saja bila gagal, exit tetap 0** (data sudah tersimpan). |
| `list` | Kolom: KEY, LABEL, BASE_URL, AUTH, STATUS, TIMEOUT, AKTIF. Tandai baris `is_local` dan endpoint aktif. |
| `rm` | **Tolak** bila `is_local` (AC-8) atau sedang aktif (AC-8). Pesan menyebut alasan. |
| `test` | Panggil `<base_url>/healthz` (fallback: `/`). Laporkan **sebab** terklasifikasi + durasi ms. Exit 1 bila gagal. |
| `use` | Set endpoint aktif ke kolom `is_active` di tabel (bukan file state) agar konsisten antar proses; hanya satu baris boleh aktif → dikelola di transaksi. Ini basis "portal aktif" untuk captive portal (WO-02). |
| `active` | Cetak endpoint aktif; default `local` bila belum pernah di-set. |

### Pemetaan error → kalimat aksiabel (untuk `test`)

| Sinyal | Kalimat |
|---|---|
| `connection refused` | endpoint menolak koneksi — instance cabang kemungkinan mati |
| `no such host` | nama host tidak resolve — periksa DNS/URL |
| `i/o timeout` / `context deadline exceeded` | tidak ada respons dalam N ms — kemungkinan firewall |
| `401`/`403` | token ditolak — periksa `--auth-kind bearer` dan token |
| `404` pada `/healthz` | endpoint hidup tetapi bukan aplikasi ini |
| `5xx` | instance cabang error — periksa log cabang |

---

## Bagian D — Admin API

```
GET    /api/admin/endpoints
POST   /api/admin/endpoints
PUT    /api/admin/endpoints/:key
DELETE /api/admin/endpoints/:key
POST   /api/admin/endpoints/:key/test
GET    /api/admin/endpoints/active
POST   /api/admin/endpoints/:key/use
```

**Aturan:**
- Response **tidak pernah** memuat `token`/`token_enc`. Hanya `auth_kind` (dan `has_token: true/false`).
- `PUT` = **partial update** (pointer field), password/token opsional. **Pelajaran dari tema lama: jangan salin struct apa adanya ke upsert penuh — `status` bisa ter-reset diam-diam.**
- Semua error 4xx memakai pesan yang bisa ditindaklanjuti.

---

## Bagian E — Audit

`endpoint_audit` diisi **hanya saat request lintas instance** (bukan endpoint `local`).

Wajib:
- `request_id` unik per request (UUIDv7 sudah tersedia di `internal/uuid`).
- **Jangan** mencatat body, header `Authorization`, atau token (AC-7).
- `error_class` terisi untuk kegagalan; `NULL` bila sukses.

---

## Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | `migrate up --plugin endpoint` membuat `endpoints` + `endpoint_audit`; seed `local` ada | query DB |
| 2 | `endpoint add --key cabang-jkt --base-url ... --label ...` sukses; `token_enc` tersegel | query `token_enc` bukan plaintext |
| 3 | `endpoint add` dengan key duplikat → error jelas | test |
| 4 | `endpoint list` menampilkan endpoint + penanda aktif | output |
| 5 | `endpoint test --key X` sukses → durasi; gagal → sebab aksiabel + exit 1 | test 2 arah |
| 6 | `endpoint use cabang-jkt` lalu `endpoint active` → konsisten | test |
| 7 | `endpoint rm --key local` → **ditolak** | test |
| 8 | `endpoint rm --key <aktif>` → **ditolak** | test |
| 9 | Migrasi `down` menghapus kedua tabel, lalu `up` ulang bersih | test |
| 10 | `grep -rn 'CREATE TABLE' internal/ plugins/ --include=*.go` → **0 hits** | grep |
| 11 | `go build ./...`, `go vet ./...`, `go test ./...` lulus | CI |
| 12 | Tidak ada token/password di output CLI maupun response API | grep output |

## Definition of Done

- [ ] Semua AC di atas lulus dengan bukti output nyata
- [ ] Rebuild binary: `go build -o bin/console ./cmd/console` (**`bin/` tidak di-update oleh `go build ./...`** — jebakan yang sudah 2× terjadi)
- [ ] Bila ada bagian yang belum bisa diverifikasi (mis. `endpoint test` ke host yang belum ada), **nyatakan belum terbukti** — jangan ditandai selesai
- [ ] Tidak ada push ke remote

## Out of Scope

- **Captive portal + routing per request** (WO-02): proxy HTTP, `X-Branch-Endpoint`, `X-Forwarded-Instance` (anti-loop D-12), klasifikasi error runtime, dua token (`main`/`portal token` D-14/D-15), UI pemilihan portal
- **SSO lintas instance (WO-03, admin-only D-8)**: portal token, guard keanggotaan `admins` cabang, enrollment admin, verifikasi lokal
- **User `members`**: di luar ruang lingkup federasi saat ini (ditunda)
- `endpoint update` CLI (WO terpisah — tapi desain `PUT` API sudah partial di WO ini)
- Sinkronisasi data antar cabang

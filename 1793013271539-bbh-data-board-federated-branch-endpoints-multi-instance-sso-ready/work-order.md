---
title: work order
description: "Master index WO arsitektur federasi endpoint: pusat & cabang menjalankan aplikasi sama, pusat beralih endpoint ke API cabang, SSO menyusul. Pengganti topic multi-database yang sudah di-destroy."
status: draft
created: "2026-10-26"
updated: "2026-10-27"
---
# Work Order (Master Index): Federated Branch Endpoints

## Referensi
- SPEC: `SPEC.md` di topic ini (D-1..D-15, AC-1..AC-22)
- Shortnote handoff: `SHORTNOTE-WO-01.md`

## Latar Belakang Arah
Arah sebelumnya (multi-DB terpusat: pusat mengakses DB cabang langsung) **dibatalkan** dan seluruh artefaknya dihancurkan pada 2026-10-26:
- Migrasi di-rollback di semua target (tabel registry + business terhapus via `.down.sql`), **tanpa drop database**
- Commit repo dipotong ke `1f5a9b2` (sebelum topic multi-DB)
- Topic historic `1792765876397` dihapus

Arah baru: **federasi instance** — satu aplikasi, banyak instance, satu DB lokal per instance, komunikasi lewat HTTP API.

## Prinsip
1. Pusat **tidak pernah** membuka koneksi DB ke cabang (D-2).
2. Registry endpoint disimpan di DB lokal pusat lewat **migrasi versioned** (D-3).
3. "Ganti cabang" = ganti **base URL + token API**, bukan ganti koneksi DB (D-4).
4. Token disegel AES-GCM; tidak pernah lewat argv, tidak pernah masuk log (D-5).
5. Satu endpoint aktif per sesi; default `local` (D-6).
6. **Setiap instance (pusat & cabang) menjalankan plugin `auth` sendiri.** SSO = token pusat + **keanggotaan lokal di cabang** (D-8). User yang belum terdaftar di cabang ditolak 403.
7. **Semua instance identik** — "pusat" bukan role kode, hanya instance yang mendaftarkan endpoint lain (D-11).
8. **Anti-loop:** request lintas instance maksimal 1 hop (`X-Forwarded-Instance`) (D-12).
9. **Captive portal:** dashboard admin pusat memilih "Masuk ke Cabang" → **API business digantikan**; **auth tetap memakai token pusat**.

## Daftar WO

| # | File | Scope | Status |
|---|---|---|---|
| 1 | [`WO-01-endpoint-registry.md`](rev 2) | **Endpoint Registry** (admin-only, D-8); `endpoints` (+`is_active`+partial index unique) + `endpoint_audit`; migrasi versioned; plugin CLI (`add/list/rm/test/use/active`); admin API (partial PUT tanpa reset `status`, audit semua request lintas endpoint) | complete |
| 2 | [`WO-02-captive-portal-routing.md`](rev 1) | **Captive Portal & Routing**; pemilihan portal → proxy routing per request (`X-Branch-Endpoint`, anti-loop `X-Forwarded-Instance`), pemilik route eksplisit (**List Portal & login selalu `local`**), dua token (**main token**/**portal token**), error klasifikasi, fallback hanya bila dikonfigurasi + penanda, audit request lintas instance | complete |
| 3 | [`WO-03-sso-admin-only.md`](rev 3) | **SSO Lintas Instance (admin-only)** — token pusat (`aud`=target branch, `iss`, `sub`=UUIDv7, `typ=portal`, `exp` pendek) **tanpa klaim peran**; verifikasi **LOKAL HS256** (D-17) tanpa callback; guard keanggotaan lokal via **`iss`+`sub`** (D-16) → **403** bila belum/`pending`; **enrollment hybrid A+B** (D-18); **otorisasi = RBAC cabang** (D-19); migrasi kolom `admins`; dua-token session | complete |
| 4 | [`WO-04-rbac-groups-permissions.md`](rev 1) | **RBAC: groups & permissions** — ganti `is_root` dengan `permissions` ↔ `group_permissions` ↔ `groups` ↔ `admin_groups` ↔ `admins`; izin efektif = union grup aktif; guard `RequirePermission`; **fail-closed**; grup bertingkat `root`(`*`) / `admin`(baca+tulis) / `viewer`(baca, **default**, tanpa audit); **grup default otomatis** ke admin baru; audit hanya `root`; migrasi `is_root=true` → grup `root` lalu kolom dihapus; CLI + API grup; anti-lockout | complete |
| 5 | [`WO-05-housekeeping-docs-env.md`](rev 2) | **Housekeeping pasca-federasi** — regenerasi swagger (`is_root` basi, route baru hilang), dokumentasikan `INSTANCE_KEY`/`SSO_SHARED_SECRET`/`FEDERATION_FALLBACK_LOCAL` di `.env.example`, **guard fail-closed** `SSO_SHARED_SECRET` ≠ `APP_KEY` (normalisasi base64, tidak mencetak kunci; kosong → SSO nonaktif + log), **bukti AC-13 lewat UI (wajib)**, lalu drop 3 DB sisa | progress |

## Status Akhir (diverifikasi Manager 2026-10-27)

| Klaim Dev Agent | Hasil verifikasi |
|---|---|
| Commit `a13645e` / `51b4f06` / `1831ec2` / `bbfeb4a` sesuai judul WO | ✅ benar |
| `go build` / `go vet` / `go test` / `gofmt` lulus | ✅ benar |
| Kode fitur di `plugins/` (`endpoint`, `federation`, `portalsso`), bukan `internal/` | ✅ benar |
| `RequireRootAdmin` sudah tidak dipakai | ✅ benar (0 hasil) |
| Kolom `is_root` hilang dari model & kode | ✅ benar |
| Migrasi `is_root` dipisah dari hapus kolom | ✅ benar |
| Portal token tanpa klaim peran (D-19) | ✅ benar (`sso.go`) |
| AC-13 belum terbukti | ✅ benar & jujur dilaporkan |
| Working tree bersih selain `bin/` | ✅ benar |

**Celah yang tidak dilaporkan** → dicatat sebagai `WO-05`:

| # | Celah |
|---|---|
| T-1 | `docs/swagger` basi: masih `is_root`, **0** route endpoint/federation/portalsso |
| T-2 | `INSTANCE_KEY`, `SSO_SHARED_SECRET`, `FEDERATION_FALLBACK_LOCAL` tidak ada di `.env.example` |
| T-3 | AC-13 belum diuji lewat UI (sudah diakui jujur, tinggal pembuktian) |
| T-4 | ⚠️ Praktik uji memakai nilai `APP_KEY` sebagai `SSO_SHARED_SECRET` — **kunci dipakai ulang untuk dua tujuan berbeda**; perlu peringatan & pemisahan |

## Urutan Eksekusi & Dependency
```
WO-01 (registry)  ─┐
WO-02 (routing)   ─┼─► WO-04 (RBAC) ──► WO-03 (SSO admin-only)
                    │
                    └─ WO-04 boleh jalan paralel dengan WO-01/WO-02
```

| Aturan | Alasan |
|---|---|
| **WO-04 WAJIB selesai sebelum WO-03** | Otorisasi di cabang ditentukan RBAC lokal (D-19). Tanpa tabel grup/permission, SSO tidak punya dasar menentukan hak akses |
| WO-02 butuh WO-01 | Routing perlu struktur endpoint |
| WO-03 butuh WO-02 | SSO perlu jalur routing untuk menerima & menguji portal token |

## Kondisi Kode Saat Ini (verified 2026-10-26)
| Ada | Hilang |
|---|---|
| `plugins/auth` (+JWT, middleware, migrations) | `internal/migration`, `console migrate` |
| `internal/secrets` (AES-GCM) | `internal/db/registry.go` |
| `internal/uuid` (UUIDv7), `internal/console` | `plugins/business*`, `cmd/business` |

`go build ./...` dan `go test ./...` **lulus** pada kondisi ini.

## Tabel Per Instance (ringkas)
| Tabel | Fungsi | Kapan |
|---|---|---|
| `admins` | Akun admin per-instance (ada sebelum WO ini) | Sejak awal |
| `members` | Akun member per-instance (ada sebelum WO ini) | Sejak awal |
| `migrations` | Jejak migrasi ada (karena DB tidak di-drop) | Leftover |
| `migration_targets` | Target migrasi (leftover) | Leftover |
| `endpoints` | Daftar endpoint + `is_active` | WO-01 |
| `endpoint_audit` | Audit request lintas instance | WO-01 |
| `permissions` | Daftar izin (key stabil) | WO-04 |
| `groups` | Kumpulan izin | WO-04 |
| `group_permissions` | Relasi grup ↔ izin | WO-04 |
| `admin_groups` | Relasi admin ↔ grup | WO-04 |

Catatan: migrator baru harus kompatibel dengan `migrations` yang sudah terisi atau menggunakan `CREATE TABLE IF NOT EXISTS`.

## Ketergantungan Terbuka (SUDAH DIPUTUSKAN 2026-10-26)
| # | Topik | Keputusan |
|---|---|---|
| 1 | Enrollment admin cabang | **D-18: Hybrid A+B** — manual oleh admin cabang **dan** auto-provision baris `pending` yang **wajib disetujui** (tanpa auto-approve) |
| 2 | Identitas pencocokan | **D-16: `iss` + `sub` (UUIDv7)** — bukan email; email hanya atribut tampilan (`atr`) |
| 3 | Kunci verifikasi SSO | **D-17: HS256 shared secret** — di balik interface verifier, dapat diupgrade ke RS256/JWKS |
| 4 | Peran / `is_root` | **D-19..D-25: otorisasi LOKAL + RBAC grup** | Token tanpa klaim peran; `is_root` diganti grup `root` ber-permission `*`; grup bertingkat `viewer`(default)/`admin`/`root`; fail-closed tapi bawaan "bisa lihat, belum bisa ubah" |

**Konsekuensi skema:** `admins` perlu migrasi tambahan (`status`, `origin`, `sso_issuer`, `sso_subject`, `password_hash` nullable, unique index `iss`+`sub`) — lihat `WO-03-sso-admin-only.md` §5. Tabel RBAC (`permissions`, `groups`, `group_permissions`, `admin_groups`) — lihat `WO-04-rbac-groups-permissions.md` §3.

## Out of Scope (keseluruhan topic)
- Sinkronisasi/penggabungan data antar cabang (laporan gabungan) — belum ada kebutuhan terkonfirmasi
- Replikasi, sharding, HA
- Frontend dashboard pemilihan cabang (menyusul setelah API stabil)

---
title: work order
description: "Master index WO arsitektur federasi endpoint: pusat & cabang menjalankan aplikasi sama, pusat beralih endpoint ke API cabang, SSO menyusul. Pengganti topic multi-database yang sudah di-destroy."
status: draft
created: "2026-10-26"
updated: "2026-10-26"
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
| 1 | [`WO-01-endpoint-registry.md`](rev 2) | **Endpoint Registry** (admin-only, D-8); `endpoints` (+`is_active`+partial index unique) + `endpoint_audit`; migrasi versioned; plugin CLI (`add/list/rm/test/use/active`); admin API (partial PUT tanpa reset `status`, audit semua request lintas endpoint) | draft |
| 2 | [`WO-02-captive-portal-routing.md`](rev 1) | **Captive Portal & Routing**; pemilihan portal → proxy routing per request (`X-Branch-Endpoint`, anti-loop `X-Forwarded-Instance`), pemilik route eksplisit (**List Portal & login selalu `local`**), dua token (**main token**/**portal token**), error klasifikasi, fallback hanya bila dikonfigurasi + penanda, audit request lintas instance | drafted |
| 3 | [`WO-03-sso-admin-only.md`](rev 2) | **SSO Lintas Instance (admin-only)** — DIPERLUKAN; token pusat (`aud`=target branch, `iss`, `sub`=UUIDv7, `typ=portal`, `exp` pendek); verifikasi **LOKAL HS256** (D-17) tanpa callback; guard keanggotaan lokal via **`iss`+`sub`** (D-16) → **403** bila belum/`pending`; **enrollment hybrid A+B** (D-18: manual + auto-provision `pending` wajib approval); SSO tidak pernah beri `is_root` (D-19); butuh migrasi kolom `admins`; dua-token session | drafted |

## Urutan Eksekusi & Dependency
```
WO-01 (registry + migrasi + CLI + API)
   └──► WO-02 (captive portal & routing)
            └──► WO-03 (SSO admin-only)
```
**Tidak bisa di-lompat** — WO-02 butuh struktur endpoint dari WO-01; WO-03 butuh routing WO-02 untuk menerima & menguji portal token.

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

Catatan: migrator baru harus kompatibel dengan `migrations` yang sudah terisi atau menggunakan `CREATE TABLE IF NOT EXISTS`.

## Ketergantungan Terbuka (SUDAH DIPUTUSKAN 2026-10-26)
| # | Topik | Keputusan |
|---|---|---|
| 1 | Enrollment admin cabang | **D-18: Hybrid A+B** — manual oleh admin cabang **dan** auto-provision baris `pending` yang **wajib disetujui** (tanpa auto-approve) |
| 2 | Identitas pencocokan | **D-16: `iss` + `sub` (UUIDv7)** — bukan email; email hanya atribut tampilan (`atr`) |
| 3 | Kunci verifikasi SSO | **D-17: HS256 shared secret** — di balik interface verifier, dapat diupgrade ke RS256/JWKS |
| 4 | Peran (tambahan) | **D-19: SSO tidak pernah memberi `is_root`**; selalu level `admin`. Skema `admins` tidak punya kolom `level` |

**Konsekuensi skema:** `admins` perlu migrasi tambahan (`status`, `origin`, `sso_issuer`, `sso_subject`, `password_hash` nullable, unique index `iss`+`sub`). Lihat `WO-03-sso-admin-only.md` §5.

## Out of Scope (keseluruhan topic)
- Sinkronisasi/penggabungan data antar cabang (laporan gabungan) — belum ada kebutuhan terkonfirmasi
- Replikasi, sharding, HA
- Frontend dashboard pemilihan cabang (menyusul setelah API stabil)

---
title: work order
description: "Master index WO arsitektur federasi endpoint: pusat & cabang menjalankan aplikasi sama, pusat beralih endpoint ke API cabang, SSO menyusul. Pengganti topic multi-database yang sudah di-destroy."
status: draft
created: "2026-10-26"
updated: "2026-10-26"
---
# Work Order (Master Index): Federated Branch Endpoints

## Referensi
- SPEC: `SPEC.md` di topic ini (D-1..D-10, AC-1..AC-10)
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
| 1 | `WO-01-endpoint-registry.md` | **Endpoint Registry** (rev 2) — pulihkan `internal/migration` minimal, plugin `endpoint` + migrasi (`endpoints` dengan `is_active` + `endpoint_audit`), CLI `endpoint add/list/rm/test/use/active`, admin API (partial PUT, tanpa token), audit request. **Scope: admin saja** (D-8); `members` di luar scope | draft |
| 2 | *(belum ditulis)* Captive Portal & Switching | Pilihan "Masuk ke Cabang" di dashboard admin pusat → SSO, reload, menu diarahkan ulang **kecuali List Portal (tetap pusat) & Admin Profile (dua versi)**; proxy per request (`X-Branch-Endpoint`), anti-loop (`X-Forwarded-Instance`), klasifikasi error, fallback `local`, audit | not started |
| 3 | *(belum ditulis)* SSO lintas instance | **DIPERLUKAN.** Pusat terbitkan token identitas; cabang verifikasi **lokal** + **guard keanggotaan lokal** (user wajib sudah terdaftar di cabang); pemetaan peran; `aud`. | not started |

## Urutan Eksekusi & Dependency
```
WO-01 (registry + migrasi + CLI + API)
   └──► WO-02 (switching per request)
            └──► WO-03 (SSO fase 2)
```

## Kondisi Kode Saat Ini (verified 2026-10-26)
| Ada | Hilang |
|---|---|
| `plugins/auth` (+JWT, middleware, migrations) | `internal/migration`, `console migrate` |
| `internal/secrets` (AES-GCM) | `internal/db/registry.go` |
| `internal/uuid` (UUIDv7), `internal/console` | `plugins/business*`, `cmd/business` |

`go build ./...` dan `go test ./...` **lulus** pada kondisi ini.

**Catatan DB:** karena database tidak di-drop, tabel `migrations` dan `migration_targets` **sudah ada**. Migrator baru wajib kompatibel atau memakai `CREATE TABLE IF NOT EXISTS`.

## Out of Scope (keseluruhan topic)
- Sinkronisasi/penggabungan data antar cabang (laporan gabungan) — belum ada kebutuhan terkonfirmasi
- Replikasi, sharding, HA
- Frontend dashboard pemilihan cabang (menyusul setelah API stabil)

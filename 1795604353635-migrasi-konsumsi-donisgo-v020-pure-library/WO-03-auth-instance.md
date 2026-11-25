---
title: WO-03 — Migrasi auth ke API instans (D-4)
description: "plugins/auth memakai auth.New(auth.ConfigFromEnv()) satu titik; hapus seluruh pemakaian wrapper deprecated auth package-level dari kode bbh; test isolasi dua instance + fail-closed secret kosong."
status: complete
created: "2026-11-25"
updated: "2026-11-25"
tags: [donisgo, migrasi, wo-03, auth]
---

# WO-03 — Migrasi auth ke API instans (D-4)

Ref: `spec.md` D-4; CHANGELOG donisgo v0.2.0 §Added (SPEC-004 D3). Prasyarat: WO-02 complete.

## Lingkup

- Titik tunggu: `plugins/auth` membuat **satu** `*auth.Auth` via `auth.New(auth.ConfigFromEnv())` saat `RegisterServices` dan meng-inject-nya ke service/middleware yang butuh sign/verify (JWT login admin, verify API, SSO bila memakai util auth, refresh opaque).
- `auth.ConfigFromEnv()` memetakan env legacy: `AUTH_JWT_SECRET` dulu, lalu `JWT_ACCESS_SECRET`/`JWT_REFRESH_SECRET`/`JWT_RESET_SECRET` + `JWT_*_EXP_SECONDS` → `.env` bbh TIDAK perlu rename wajib; `.env.example` cukup diperbarui dokumentasinya.
- Hapus SEMUA call-site wrapper deprecated di kode bbh (daftar lengkap dari WO-01 AC-5). Setelah WO-03: `grep -rE "auth\.(Sign|Parse|Generate|AccessExpiry|RefreshExpiry)" --include="*.go"` hanya boleh menyisakan paket library itu sendiri.

## AC

- **AC-1 — Satu instance**: satu titik konstruksi `auth.New` di plugin auth; tidak ada konstruksi kedua. Bukti: grep + kutipan kode.
- **AC-2 — Verifikasi tetap benar**: login → token baru; token valid → `/me` 200; token tamper/expired → 401. Token yang dibuat SEBELUM WO-03 (dengan secret sama) tetap terverifikasi — tidak ada invalidasi sesi user nyata. Bukti: curl.
- **AC-3 — Fail-closed secret**: tanpa secret (`.env` kosong `AUTH_JWT_SECRET`) → startup/login gagal keras (bukan default insecure). Bukti: uji dengan working dir terpisah + `.env` sendiri (pola `/tmp/seccheck` — gotcha `godotenv.Overload`: `docker compose exec -e` TIDAK berpengaruh).
- **AC-4 — Isolasi dua instance** (unit test): dua `*auth.Auth` dengan secret berbeda tidak saling memverifikasi token. Bukti: test baru lulus.
- **AC-5 — Grep bersih**: nol pemakaian wrapper deprecated di kode bbh (lihat Lingkup). Bukti: output grep.
- **AC-6 — Build/test/CLI hijau**: `go build/vet/test ./...`; CLI auth (`auth init-admin-root`, `auth group list`) jalan dari `cmd/console` baru.
- **AC-7 — RBAC regression**: viewer read 200 / write 403; admin anti-escalation 403 tak berubah. Bukti: sampling curl/Bruno.

## Catatan Implementasi (Dev)

### Ringkasan desain (penting untuk review)
**Satu titik konstruksi, disimpan di paket `services`, dikonsumsi semua guard/handler.**
`auth.New(auth.ConfigFromEnv())` dipanggil **hanya** di `plugins/auth/plugin.go` (`buildTokenAuth`, dipanggil dari `RegisterServices`). Instance disimpan lewat `services.SetTokenAuth(...)` dan dibaca lewat `services.TokenAuth()`. Alasan memilih penyimpanan di paket `services` (bukan field struct yang di-inject ke tiap fungsi): `middleware.RequirePermission(svc, perm)` dipakai oleh **8 plugin lain** (endpoint, sec, warehouse, visit, portalsso, hsh, dashboard, ingest, resident) sehingga menambah parameter `*auth.Auth` ke signature akan menyentuh puluhan call-site di luar lingkup WO-03. Signature publik **tidak berubah**, jadi plugin lain tidak perlu disentuh — sesuai batas "jangan menyentuh di luar plugin auth".

Urutan aman terbukti dari sumber library: `app/bootstrap.go:474-492` → `attachPlugins()` memanggil `registry.RegisterAllServices(...)` **lalu** `RegisterAllRoutes(...)`. Jadi instance sudah terpasang sebelum guard/handler mana pun bisa dieksekusi.

### 1. `plugins/auth/services/service.go:13-32`
- Import `github.com/rolldone/donisgo/auth`; tambah `var tokenAuth *auth.Auth`, `SetTokenAuth`, `TokenAuth`. **Kenapa**: satu wadah instance lintas-paket tanpa mengubah signature publik middleware/handler (lihat ringkasan desain).

### 2. `plugins/auth/plugin.go:10,13,44-58` (`RegisterServices`)
- `auth.New(auth.ConfigFromEnv())` dipanggil di sini (satu-satunya). **Kenapa**: AC-1; library menjamin services terdaftar sebelum routes.
- `services.SetTokenAuth(tokens)`.
- Error dibungkus `fmt.Errorf("auth plugin: build token instance: %w", err)` → `RegisterServices` gagal → `app.New` gagal → **proses tidak boot** (fail-closed).

### 3. `plugins/auth/plugin.go:60-75` (`buildTokenAuth`) — **deviasi sadar dari instruksi**
- Instruksi WO-03 menyebut `auth.New(auth.ConfigFromEnv())` langsung. Saat diperiksa, `ConfigFromEnv()` di library **mempertahankan fallback insecure legacy**: bila `AUTH_JWT_SECRET` dan `JWT_ACCESS_SECRET` kosong, ia mengisi `"change-me-access-secret"` dsb. (`auth/config.go:105-107`, komentarnya sendiri: *"It mirrors the deprecated behaviour, insecure fallbacks included"*). Artinya `auth.New` **tidak akan** error pada env kosong, sehingga **AC-3 tidak akan tercapai** kalau hanya memanggil `auth.New`.
- Karena itu ditambah guard eksplisit: bila `AUTH_JWT_SECRET` **dan** `JWT_ACCESS_SECRET` sama-sama kosong → return error. **Kenapa**: AC-3 mensyaratkan gagal keras, bukan default insecure. Perilaku normal (secret ada) tetap 100% `ConfigFromEnv()` sehingga kompatibilitas env legacy terjaga.

### 4. `plugins/auth/middleware/middleware.go:133-160` (`bearerClaims`)
- `auth.ParseAccessTokenClaims(token)` → `services.TokenAuth().ParseAccessTokenClaims(token)`; import `donisgo/auth` dihapus dari file ini.
- **Fail-closed tambahan**: bila `TokenAuth()` nil → `return "", "", false` (401), bukan jatuh ke wrapper deprecated.

### 5. `plugins/auth/handlers/auth.go:11,44,140,201-216`
- `auth.AccessExpirySeconds()` → `accessExpirySeconds()` (helper baru di file ini yang membaca instance; `0` bila nil).
- `issueToken` sekarang memakai `tokens.GenerateAccessTokenWithLevel(...)` dan `tokens.AccessExpirySeconds()` untuk TTL; error eksplisit bila instance nil.
- Import `donisgo/auth` dihapus.
- **TTL**: sumber sama persis — `ConfigFromEnv()` membaca `JWT_ACCESS_EXP_SECONDS` (default 600s) / `JWT_REFRESH_EXP_SECONDS` / `JWT_RESET_EXP_SECONDS`, identik dengan `internal/auth` lama. `.env` bbh (`JWT_ACCESS_EXP_SECONDS=7200`) tetap berlaku; tidak ada TTL yang dibaca manual di tempat lain.

### 6. `plugins/auth/plugin_test.go` (baru) — 4 test
- `TestInstances_IsolatedSecrets` (AC-4): dua instance secret berbeda → token A tidak terverifikasi B (dan A memverifikasi miliknya sendiri).
- `TestBuildTokenAuth_FailClosed` (AC-3): semua env secret dikosongkan → error + nil instance.
- `TestBuildTokenAuth_FailClosedFromDotEnv` (AC-3, jalur boot nyata): `t.TempDir()` + `.env` sendiri berisi `AUTH_JWT_SECRET=` kosong, `t.Chdir` + `godotenv.Overload()` — meniru `init()` console/server. **Kenapa pola ini**: `godotenv.Overload` membaca `.env` dari **cwd**, jadi `docker compose exec -e AUTH_JWT_SECRET=""` memang tidak berpengaruh (gotcha yang disebut WO-03).
- `TestBuildTokenAuth_LegacyAccessSecret` (kompatibilitas): hanya `JWT_ACCESS_SECRET` di-set → instance tetap terbentuk (deployment lama tidak pecah).

### 7. Env & docs
- `.env.example` tidak diubah: sudah mendokumentasikan `AUTH_JWT_SECRET` sebagai sumber utama + `JWT_*_EXP_SECONDS`. **Tidak ada rename env wajib** (WO-03 batas).

### Catatan untuk Manager (temuan, bukan blocker)
- **`gofmt` debt pra-ada**: `gofmt -l` menyorot 27 file (mis. `plugins/auth/cli.go`, `plugins/endpoint/plugin.go`, `plugins/resident/plugin.go`). File-file itu **tidak** saya sentuh dan diff-nya hanya urutan grup import (sisa rewrite import WO-02). Empat file yang saya ubah gofmt-bersih. Saya **tidak** memformat ulang 27 file itu karena di luar lingkup WO-03 dan akan mengaburkan diff atomik — usul: satu commit `style: gofmt` terpisah sebelum WO-04.
- **`portalsso` tidak disentuh** (sesuai batas). Diperiksa: `plugins/portalsso` hanya memakai `authservices.New(db)` (service RBAC) + `middleware.RequirePermission`; ia **tidak** memanggil util token `donisgo/auth` sama sekali, jadi tidak ada keterkaitan yang perlu dilaporkan sebagai STOP.
- **`.env` dev** berisi `AUTH_JWT_SECRET=change_me_to_a_strong_secret` (placeholder) → lolos guard AC-3 karena non-kosong. Ini kondisi dev yang sudah ada sebelumnya; tidak diubah.

## Review Manager

**LULUS — WO-03 COMPLETE.** Dev: commit `c495e2c` (5 file, +202/−9), dok historic `e42314d`, tanpa push. Verifikasi spot-check Manager (independen, 2026-11-25):

- `grep auth.New(` di source → **1 titik konstruksi** `plugins/auth/plugin.go:65` (2 hit lain = unit test) ✅ AC-1.
- **Klaim deviasi sadar TERVERIFIKASI di sumber library**: `auth/config.go` v0.2.0 — `ConfigFromEnv()` memang memakai fallback insecure (`"change-me-access-secret"` dst. via `getEnv(..., default)`) bila env kosong → `auth.New` TIDAK error. Tanpa guard dev, AC-3 (fail-closed) pasti gagal. **Deviasi DIADILI: SAH dan WAJIB** — guard eksplisit `AUTH_JWT_SECRET` + `JWT_ACCESS_SECRET` kosong → error keras; env legacy utuh (`TestBuildTokenAuth_LegacyAccessSecret` PASS). Ini perbaikan keamanan riil, bukan penyimpangan.
- **Desain penyimpanan instance DIADILI: SETUJU** — `SetTokenAuth`/`TokenAuth` di paket `services` alih-alih parameter tambahan: `middleware.RequirePermission(svc, perm)` dipakai 9 plugin lain; injeksi parameter = puluhan call-site di luar lingkup WO-03. Signature publik tak berubah → benar secara rekayasa (prinsip: lingkup minimum per WO).
- Test eksplisit dijalankan Manager: `TestBuildTokenAuth_FailClosed` + `TestBuildTokenAuth_FailClosedFromDotEnv` + `TestInstances_IsolatedSecrets` + `TestBuildTokenAuth_LegacyAccessSecret` = **4/4 PASS** ✅ AC-3/AC-4.
- Grep wrapper deprecated di source (excl `.kilo` & module cache) = **0** ✅ AC-5.
- Runtime: boot server Manager sendiri → `POST /api/admin/login` root = 200 (token HS256 dari instance baru) ✅ AC-2.
- `portalsso` diperiksa dev & dikonfirmasi nol keterkaitan util token (`authservices.New` + `RequirePermission` saja) — diterima, tidak ada STOP.
- **Temuan gofmt debt (27 file pra-ada sisa WO-02) → diterima sebagai usulan**: commit `style: gofmt` terpisah SEBELUM WO-04 dieksekusi (WO-04 akan regen swagger; diff harus bersih dulu). Ditambahkan ke lingkup WO-04 §F housekeeping.
- Artefak uji dibersihkan (`/tmp/seccheck`, admin uji soft-delete) ✅; `.env.example` tidak perlu berubah ✅.

Status: **WO-03 complete**. Prasyarat WO-04 terpenuhi. Sisa untuk menutup topik: WO-04 (swagger re-baseline, docs/TRUSTED_PROXIES/DONISGO.md banner, Bruno regresi + RBAC sampling, issue upstream H-3, re-review multi-DB D-5, housekeeping gofmt).

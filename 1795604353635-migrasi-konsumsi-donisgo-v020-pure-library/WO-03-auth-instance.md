---
title: WO-03 — Migrasi auth ke API instans (D-4)
description: "plugins/auth memakai auth.New(auth.ConfigFromEnv()) satu titik; hapus seluruh pemakaian wrapper deprecated auth package-level dari kode bbh; test isolasi dua instance + fail-closed secret kosong."
status: pending
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

(dikosongkan)

## Review Manager

(dikosongkan)

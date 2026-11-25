---
title: WO-02 — Migrasi inti (module rename + hapus internal/ + cmd ala template)
description: "Satu commit atomik: go.mod module baru + require donisgo v0.2.0, rewrite ~104 import, hapus internal/ terganti, cmd/server + cmd/console pola donisgo-template, registrasi eksplisit 13 plugin + EnsurePluginLast(pages), hapus internal/pluginloader, swap plugin_registry."
status: pending
created: "2026-11-25"
updated: "2026-11-25"
tags: [donisgo, migrasi, wo-02, atomik]
---

# WO-02 — Migrasi inti (ATOMIK)

Ref: `spec.md` §Target, §Pemetaan, §Perubahan perilaku. Prasyarat: WO-01 lulus (tidak ada blocker).

## Lingkup (satu commit atomik)

1. `go.mod`: module → `github.com/AkasaTechDev/bbh-data-board`; `require github.com/rolldone/donisgo v0.2.0`; `go mod tidy`.
2. Rewrite import: semua `donisgo/internal/<paket>` → path baru (library `github.com/rolldone/donisgo/<paket>` atau app-side sesuai tabel WO-01).
3. Hapus paket `internal/` yang 1:1 terganti library (`plugins`, `db`, `auth`, `events`, `keydb`, `mail`, `storage`, `secrets`, `uuid`, `console` engine, `pluginloader`) — termasuk testnya yang duplikat milik library.
4. `internal/app/bootstrap.go` → logika wiring dipindah ke `cmd/server/main.go` (registrasi 13 plugin bbh urutan tertentu + `registry.EnsurePluginLast(pagesPlugin)` + `/healthz` tetap).
5. `cmd/console/main.go` pola template: `console.RegisterAdditionalPlugins(...)` + `console.Execute()`; generator `plugin new` diadopsi dari `donisgo-template/console/plugin.go` (import framework tetap donisgo).
6. `plugins/plugin_registry` lokal dihapus, import diganti paket library (D-3; asumsi diff WO-01 "aman").
7. `internal/admin` sesuai keputusan WO-01/owner (default: hapus bila stub).
8. Env port sesuai temuan AC-4 WO-01 (`.env`, `.env.example`, `docker-compose.yml`).
9. `migrations/` core: bila v0.2.0 menuntut core di `migrations/{db_type}` — sesuaikan layout & verifikasi path.

## AC

- **AC-0 — Regresi nol pasca-commit atomik**: satu commit tunggal; `go build ./...`, `go vet ./...`, `go test ./...` hijau (test pre-existing merah boleh dulu dgn bukti sama seperti baseline WO-01 AC-7c); `grep -r "rolldone/donisgo/internal" --include="*.go"` = 0 hasil; `grep -r "module github.com/rolldone/donisgo" go.mod` = tidak ada.
- **AC-1 — Server boot + serving**: `go run ./cmd/server` boot; `/healthz` 200; `/` homepage html (`href="/app/"`); `/app/` + aset 200; `/spa` redirect 301; `/api/*` tak dikenal tetap 404-JSON; API `/api/admin/residents` 401 tanpa token. Bukti: curl per endpoint.
- **AC-2 — Registrasi eksplisit**: tidak ada plugin yang diregistrasi library otomatis — daftar 13 plugin eksplisit di `cmd/server/main.go`, `pages` terakhir via `EnsurePluginLast` (bukti: baca kode + urutan route benar: wildcard NoRoute TIDAK menelan route spesifik — bukti AC-1 cukup).
- **AC-3 — CLI jalan**: `cmd/console` baru: `migrate list` tampil; `migrate down && migrate up && migrate up` idempotent (up ke-2 0 applied); command plugin jalan (sampling: `auth init-admin-root` dry, `endpoint list`, `hsh version`, `maintenance clean-data` tanpa `--yes`, `sso list`). Generator: `go run ./cmd/console plugin new --id tmp-scaffold` → direktori dibuat, build-tree tetap hijau, lalu scaffolding UJI dihapus + `migrate down` bila sempat up (jangan tinggalkan artefak).
- **AC-4 — Data tak berubah**: DB dev pasca-migrasi: jumlah tabel & baris per plugin TIDAK berubah vs sebelum (Bukti: hitungan SQL sebelum/sesudah). Migrasi bookkeeping (`migrations`/`migration_targets`) tetap dikenali engine library (versi yang sudah applied tidak dieksekusi ulang).
- **AC-5 — RBAC & login tetap**: login `POST /api/admin/login` 200; `/me` kirim `groups`+`permissions`; viewer read 200 / write 403. Bukti: curl atau Bruno smoke.
- **AC-6 — Env port**: `.env`/compose/webapp konsisten dengan env yang dipakai library (hasil AC-4 WO-01); host akses `localhost:8082` tetap berlaku. Bukti: curl dari host.

## Catatan Implementasi (Dev)

(dikosongkan — diisi dev bertahap)

## Review Manager

(dikosongkan)

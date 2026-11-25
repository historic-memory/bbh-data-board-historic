---
title: Temuan riset donisgo v0.2.0 pure library
description: "Hasil cek pkg.go.dev + CHANGELOG donisgo v0.2.0 (pure library, breaking) dan status monorepo bbh-data-board; dasar draft SPEC migrasi."
status: progress
created: "2026-11-25"
updated: "2026-11-25"
tags: [donisgo, migrasi, library, framework]
---

# Temuan Riset: donisgo v0.2.0 (Pure Library)

Tanggal riset: 2026-11-25. Sumber: `https://pkg.go.dev/github.com/rolldone/donisgo@v0.2.0` + `CHANGELOG.md` tag `v0.2.0`.

## Fakta rilis

- **Rilis hari ini (2026-11-25)**, tag `v0.2.0`, MIT. Tagline: *"A composable Go web-framework library — built to be imported, not to be run."*
- Tidak ada binary, tidak ada `internal/`, tidak ada scaffolding. **12 paket publik**: `app`, `plugins`, `plugins/plugin_registry`, `db`, `auth`, `events`, `keydb`, `storage`, `mail`, `secrets`, `uuid`, `console`.
- Rilis **BREAKING** (jalur v0.x, minor bump tetap boleh breaking).

## Isi CHANGELOG v0.2.0 (gabungan SPEC-004 D3 + SPEC-005 fase 4)

### Removed (BREAKING)
- `plugins/pages` — plugin konten (SPA `/app`, redirect `/spa`, proxy `HOMEPAGE_URL`, wildcard `NoRoute`) **pindah ke `github.com/rolldone/donisgo-template/plugins/pages`**.
- `cmd/server` dan `cmd/console` — donisgo kini nol binary; entrypoint aplikasi ada di repo starter `donisgo-template/cmd/*`.
- Command `plugin new` (generator scaffold) pindah ke starter (`donisgo-template/console/`).
- `internal/pluginloader` dihapus sepenuhnya (`RegisterCorePlugins*`, `RegisterLastPlugins*`). Registrasi plugin kini 100% tanggung jawab pemanggil via `app.Config.Plugins`; donisgo tidak mendaftarkan plugin apa pun.
- Dokumentasi app-level (quickstart, plugin dev, env, deploy) + `.env.example` pindah ke repo starter.

### Added
- `auth.Config` + `auth.New(cfg) (*Auth, error)` — API instans; token ditandatangani/diverifikasi per instance sehingga dua instance dengan secret berbeda sepenuhnya terisolasi. `New` menolak secret kosong (tanpa default insecure). TTL opsional `AccessTTL`/`RefreshTTL`/`ResetTTL` (nol → default 10m/14d/1h). `auth.ConfigFromEnv()` memetakan env legacy (`AUTH_JWT_SECRET` dulu, lalu `JWT_ACCESS_SECRET`/`JWT_REFRESH_SECRET`/`JWT_RESET_SECRET` + `JWT_*_EXP_SECONDS`). (SPEC-004 D3)

### Changed (BREAKING)
- Serving halaman/SPA TIDAK lagi otomatis: `app` tidak menyajikan `/`, `NoRoute` catch-all, maupun `StaticFS`. Aplikasi yang butuh serving halaman mendaftarkan plugin `pages` eksplisit dan menjaga urutan terakhir via `registry.EnsurePluginLast(...)` (idempoten, de-dup by ID).
- `app.attachPlugins` tidak menambahkan plugin framework; urutan registry ditentukan pemanggil.
- `console.RootCmd` memuat `migrate` & `seed` tanpa `plugin new`; `NewRootCmd()` tetap tersedia agar starter menempelkan command sendiri.

### Deprecated
- API package-level `auth` (`Sign*Token`, `Parse*Token`, `GenerateAccessTokenWithLevel`, `ParseAccessTokenClaims`, `AccessExpirySeconds`, dkk.) kini wrapper ke instance default-env, ditandai `// Deprecated:`; dipertahankan hingga v1.0. Perilaku lama tidak berubah. (SPEC-004 D4)

### Documentation
- Godoc lengkap 12 paket publik (semua Inggris, siap pkg.go.dev). README dirombak jadi README library + bagian Versioning & releases.

### Migration notes (v0.1.x → v0.2.0)
1. Aplikasi yang mengandalkan serving otomatis **WAJIB daftarkan `pages` manual** dari repo starter, urutan terakhir — tanpa itu `/app` dan fallback SPA 404.
2. Binary server & console: pindahkan `cmd/server`/`cmd/console` ke repo starter, sesuaikan import ke module path aplikasi.
3. Generator `plugin new` via CLI starter.
4. Registrasi plugin eksplisit: `app.Config.Plugins` (server) + `console.RegisterAdditionalPlugins` (CLI).
5. Referensi docs aplikasi pindah ke README starter.

## Konteks versi sebelumnya (penting untuk path migrasi)

- `v0.1.0`: 10 paket dipindah `internal/` → root (`app`, `plugins`, `db`, `storage`, `mail`, `auth`, `secrets`, `uuid`, `events`, `keydb`); wiring injektable (`app.Config`+`app.New`, `plugins.Registry` instance-scoped, `db.Open`, `events.NewBus`, `keydb.New`); fungsi package-level jadi wrapper Deprecated; contoh `examples/library/`.
- `v0.1.1`: env `TRUSTED_PROXIES` wired ke gin `SetTrustedProxies`; default **trust-none** — app di belakang reverse proxy WAJIB set `TRUSTED_PROXIES`.
- `v0.1.2`: `console` dipromosikan jadi paket publik `github.com/rolldone/donisgo/console`.
- `v0.1.3`: CHANGELOG + section versioning.
- `v0.1.4`: fix `plugin new` pisahkan path modul framework vs aplikasi (issue #13).

## Status workspace bbh-data-board saat ini

- Workspace **masih monorepo lama pra-v0.1.0**: `go.mod` = `module github.com/rolldone/donisgo` (repo bbh = repo framework yang sama, remote `donisgo` sudah tidak terpasang — hanya `origin` = `AkasaTechDev/bbh-data-board`).
- `internal/*` lengkap (app, pluginloader, console, db, auth, events, keydb, mail, storage, secrets, uuid, plugins, admin, ...), `cmd/server` + `cmd/console` ada.
- **104 file Go** meng-import `donisgo/internal/...`.
- `plugins/` 14 folder: auth, dashboard, endpoint, federation, hsh, ingest, maintenance, pages, plugin_registry (errors.go/registry.go/registry_test.go), resident, sec, visit, warehouse (+ plugin_registry juga di internal).
- `DONISGO.md` di repo = dokumentasi monorepo (usang terhadap v0.2.0).

## Dampak & poin migrasi yang harus masuk SPEC

1. **Module rename** — module baru (usul: `github.com/AkasaTechDev/bbh-data-board`) + `require github.com/rolldone/donisgo v0.2.0`; hapus `internal/` seluruhnya; perbaiki semua import plugin.
2. **`cmd/*` dari starter template** (`donisgo-template`) + registrasi plugin eksplisit: `app.Config.Plugins` (server), `console.RegisterAdditionalPlugins` (CLI).
3. **Plugin `pages`** — bbh punya pages sendiri (SPA `/app` + `homepage.html` di `plugins/pages/plugin.go`, pemegang `NoRoute`/wildcard; lihat topic `1795505104037` WO-01 homepage `/`). Opsi: (a) pertahankan lokal + `EnsurePluginLast`, atau (b) adopsi milik template. **Keputusan owner menunggu.**
4. **`plugins/plugin_registry` lokal** vs paket library `plugins/plugin_registry` — perlu diff dulu ada/tidaknya modifikasi lokal; kalau identik → ganti import ke library.
5. **`auth` package-level deprecated** — aman jangka pendek; migrasi ke instance API bisa bertahap/post-v0.2.0.
6. **Interaksi WO multi-DB** (topic `1792765876397`, SPEC complete, WO belum eksekusi) — target refactor `internal/console/migrate.go` (`applyUp/applyDown` hard-code `db.GetGormDB()`) & `internal/db/gorm.go` (singleton `sync.Once` baca env `DB_*`) **kini milik library donisgo**. Strategi WO multi-DB WAJIB diulas ulang: refactor tidak lagi bisa dilakukan di repo bbh lokal.
7. `DONISGO.md` perlu di-usangkan/di-repoint ke pkg.go.dev + README starter.
8. Git: repo ini dulunya di-push ke remote `donisgo` (insiden reset 2026-10-29 tercatat di memori); pasca-rename module repo ini murni aplikasi konsumen.

## Pertanyaan terbuka untuk owner (sebelum WO)

- O-1: Nama module baru untuk repo bbh? (usulan `github.com/AkasaTechDev/bbh-data-board`)
- O-2: Strategi plugin `pages`: pertahankan lokal atau adopsi milik `donisgo-template`?
- O-3: `plugins/plugin_registry` lokal: ganti ke paket library, atau ada divergensi lokal yang harus dipertahankan?
- O-4: Timing migrasi auth instance API ( Deprecated tetap aman) — kerjakan di WO ini atau tunda?
- O-5: Review ulang strategi WO multi-DB (topic `1792765876397`) — refactor kini di sisi library; apakah WO di-revisi/re-scoped?

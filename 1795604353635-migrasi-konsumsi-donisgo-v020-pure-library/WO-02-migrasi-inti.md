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

> **PEMBUKAAN (2026-11-25)** — WO-01 COMPLETE, review Manager LULUS (dok `38b9ee3`/`cabe109`). Keputusan **O-B DISETUJU owner**: swap `plugin_registry` → library. Temuan WO-01 yang mengikat eksekusi WO-02: **H-3** (jangan pakai pola `EnsurePluginLast` setelah `app.New`), **H-1** (default `migrate --plugin core` + repo tanpa dir `migrations/`), **H-2** (`/healthz` kehilangan rumah), **H-4** (generator `plugin new`), dan **tidak ada perubahan env port** (library hanya baca `PORT`; compose sudah benar).

## Lingkup (satu commit atomik)

1. `go.mod`: module → `github.com/AkasaTechDev/bbh-data-board`; `require github.com/rolldone/donisgo v0.2.0`; `go mod tidy`.
2. Rewrite import: semua `donisgo/internal/<paket>` → path baru (library `github.com/rolldone/donisgo/<paket>` atau app-side sesuai tabel WO-01).
3. Hapus paket `internal/` yang 1:1 terganti library (`plugins`, `db`, `auth`, `events`, `keydb`, `mail`, `storage`, `secrets`, `uuid`, `console` engine, `pluginloader`) — termasuk testnya yang duplikat milik library.
4. `internal/app/bootstrap.go` → logika wiring dipindah ke `cmd/server/main.go` (registrasi 13 plugin bbh urutan tertentu + `/healthz` tetap hidup).
   - **H-3 (mengikat)**: `EnsurePluginLast` SETELAH `app.New` TIDAK me-register route (bukti empiris WO-01: `pages.Routes=0`). WAJIB pakai **varian A** — `pages` sebagai elemen **terakhir** di `app.Config.Plugins` — atau **varian B** — pre-register ke default registry sebelum `New`. DILARANG pola README `donisgo-template`.
   - **H-2 (mengikat)**: `registerStoreRoutes` library kosong → `/healthz` WAJIB pindah rumah: plugin kecil `healthz` (baris `router.GET("/healthz")` di `RegisterRoutes` plugin pertama) ATAU didaftarkan di `plugins/pages` sebelum wildcard. Konsumen `/healthz` yang tidak boleh putus: probe `plugins/endpoint`, owner route federation, `vite.config.ts`, `client.ts`.
   - **H-1 (mengikat)**: default `migrate --plugin` berubah `all` → `core` dan repo TIDAK punya dir `migrations/` → `migrate up` polos akan gagal. Solusi (rekomendasi Manager, reversible): buat `migrations/postgres/` kosong (`.gitkeep`) sehingga target `core` valid dengan 0 migrasi, PLUS koreksi docs (`README.md:225,281,406`, `AGENT.md:77`, `DONISGO.md:51,527`) agar contoh memakai `migrate --plugin <id> up`.
5. `cmd/console/main.go` pola template: `console.RegisterAdditionalPlugins(...)` + `console.Execute()`; generator `plugin new` diadopsi dari `donisgo-template/console/plugin.go` (import framework tetap donisgo).
6. `plugins/plugin_registry` lokal dihapus, import diganti paket library — **O-B DISETUJU owner 2026-11-25** (divergensi tekstual saja, kode identik, 0 importir).
7. `internal/admin` — **tidak ada di repo** (WO-01 AC-3); tidak ada tindakan.
8. **Env port: TIDAK ADA perubahan yang dibutuhkan** (WO-01 AC-4: library hanya baca `PORT`; `docker-compose.yml` sudah menyuntik `PORT: "8080"`; `APP_PORT=8082` murni sisi host). Jangan menyentuh `.env`/compose.
9. `migrations/postgres/` kosong dibuat (`.gitkeep`) sesuai H-1 — lihat item 4.

## AC

- **AC-0 — Regresi nol pasca-commit atomik**: satu commit tunggal; `go build ./...`, `go vet ./...`, `go test ./...` hijau (test pre-existing merah boleh dulu dgn bukti sama seperti baseline WO-01 AC-7c); `grep -r "rolldone/donisgo/internal" --include="*.go"` = 0 hasil; `grep -r "module github.com/rolldone/donisgo" go.mod` = tidak ada.
- **AC-1 — Server boot + serving (dengan H-2)**: `go run ./cmd/server` boot; `/healthz` 200 (rumah baru ditemukan — bukan 404); `/` homepage html (`href="/app/"`); `/app/` + aset 200; `/spa` redirect 301; `/api/*` tak dikenal tetap 404-JSON; API `/api/admin/residents` 401 tanpa token. Bukti: curl per endpoint.
- **AC-2 — Registrasi eksplisit + urutan benar (dengan H-3)**: tidak ada plugin diregistrasi library otomatis — daftar plugin eksplisit di `cmd/server/main.go`; `pages` terakhir via **varian A atau B** (BUKAN `EnsurePluginLast` setelah `New`). Bukti: baca kode + urutan route benar — wildcard NoRoute TIDAK menelan route spesifik (bukti AC-1 cukup); tulis varian mana yang dipakai di Catatan Implementasi.
- **AC-3 — CLI jalan (dengan H-1)**: `cmd/console` baru: `migrate list` tampil; **`migrate up` POLOS (tanpa flag) SUKSES** — tidak gagal `no migration paths found for target "core"`; `migrate down && migrate up && migrate up` idempotent (up ke-2 0 applied); command plugin jalan (sampling: `auth init-admin-root` dry, `endpoint list`, `hsh version`, `maintenance clean-data` tanpa `--yes`, `sso list`). Generator (H-4): `go run ./cmd/console plugin new --id tmp-scaffold` → direktori dibuat, build-tree tetap hijau, lalu scaffolding UJI dihapus (jangan tinggalkan artefak).
- **AC-4 — Data tak berubah**: DB dev pasca-migrasi: jumlah tabel & baris per plugin TIDAK berubah vs sebelum (Bukti: hitungan SQL sebelum/sesudah). Migrasi bookkeeping (`migrations`/`migration_targets`) tetap dikenali engine library (versi yang sudah applied tidak dieksekusi ulang).
- **AC-5 — RBAC & login tetap**: login `POST /api/admin/login` 200; `/me` kirim `groups`+`permissions`; viewer read 200 / write 403. Bukti: curl atau Bruno smoke.
- **AC-6 — Env port**: `.env`/compose/webapp konsisten dengan env yang dipakai library (hasil AC-4 WO-01); host akses `localhost:8082` tetap berlaku. Bukti: curl dari host.

## Catatan Implementasi (Dev)

(dikosongkan — diisi dev bertahap)

## Review Manager

(dikosongkan)

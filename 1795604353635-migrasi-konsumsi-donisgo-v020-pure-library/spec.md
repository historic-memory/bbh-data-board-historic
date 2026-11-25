---
title: spec
description: "SPEC migrasi bbh-data-board dari monorepo donisgo (pra-v0.1.0, module github.com/rolldone/donisgo + internal/) menjadi aplikasi konsumen donisgo v0.2.0 pure library, bentuk sejajar donisgo-template."
status: draft
created: "2026-11-25"
updated: "2026-11-25"
tags: [donisgo, migrasi, module-rename, plugin, spec]
related: ["1795604353635"]
---

# SPEC — Migrasi bbh-data-board ke donisgo v0.2.0 (pure library)

## Status
DRAFT — menunggu review owner. `work-order.md` dibuat SETELAH review.

Konteks: donisgo rilis `v0.2.0` (2026-11-25) sebagai **pure library BREAKING** — lihat riset lengkap di `notes.md`. Prinsip owner: *"bbh sebenarnya sama seperti donisgo-template, cuma internal/ tidak ada"* — jadi target akhir = bentuk aplikasi ala `donisgo-template`, mengimpor `github.com/rolldone/donisgo` yang di-pin.

## Baseline (saat ini)
- `go.mod` = `module github.com/rolldone/donisgo` (repo bbh = repo framework, monorepo pra-v0.1.0).
- `internal/*` lengkap; **104 file Go** meng-import `donisgo/internal/...`.
- `plugins/` 14 folder: auth, dashboard, endpoint, federation, hsh, ingest, maintenance, pages, plugin_registry, resident, sec, visit, warehouse (plugin_registry = errors.go/registry.go/registry_test.go).
- `cmd/server` + `cmd/console` milik repo; bootstrap di `internal/app/bootstrap.go` (registrasi plugin + `/healthz`).
- Migrasi hanya di `plugins/<id>/migrations/postgres/` (auth memegang tabel framework: admins/RBAC/endpoints).

## Target (bentuk donisgo-template)
- `go.mod`: `module github.com/AkasaTechDev/bbh-data-board` + `require github.com/rolldone/donisgo v0.2.0`.
- `cmd/server/main.go` tipis: `app.New(app.Config{Plugins: [...]})` — daftar plugin bbh eksplisit; `registry.EnsurePluginLast(pagesPlugin)` (pages memegang `router.NoRoute`).
- `cmd/console/main.go`: `console.RegisterAdditionalPlugins([...])` + `console.Execute()`; generator `plugin new` diadopsi dari `donisgo-template/console/plugin.go`.
- `plugins/<id>/` 13 plugin bisnis TIDAK berubah perilaku — hanya import di-rewrite.
- **`internal/` dihapus seluruhnya**; paket yang sudah ada di library diganti import library.
- `plugins/pages` TETAP milik bbh (SPA `/app`, redirect `/spa`, `homepage.html` exact-`/`, `HOMEPAGE_URL` proxy) — tidak diadopsi dari template.

## Keputusan terkunci (owner, 2026-11-25)
| ID | Keputusan |
|---|---|
| D-1 | Module baru = `github.com/AkasaTechDev/bbh-data-board` (usulan Manager disetujui). |
| D-2 | Plugin `pages` TETAP di bbh ("itu kan emang bukan bagian dari internal") — plugin konten aplikasi, daftar paling akhir. |
| D-3 | `plugins/plugin_registry` lokal → ganti paket library `github.com/rolldone/donisgo/plugins/plugin_registry`. **Fail-loud**: bila diff menemukan divergensi/ekstensi lokal, berhenti & lapor owner (jangan tukar diam-diam). |
| D-4 | Migrasi `auth` ke API instans (`auth.New(auth.ConfigFromEnv())`) DIKERJAKAN dalam migrasi ini (WO terpisah ringan); tidak menunggu v1.0. |
| D-5 | Re-review strategi WO multi-DB (topic `1792765876397`) DIKERJAKAN di topic ini — target refactor lama (`internal/console/migrate.go`, `internal/db/gorm.go`) kini milik library. |
| D-6 | Bentuk akhir bbh sejajar `donisgo-template`: tanpa `internal/`, cmd tipis, registrasi eksplisit. |

## Pemetaan `internal/*` → target (verifikasi rinci = WO-01)
| Paket bbh saat ini | Target |
|---|---|
| `internal/app` (bootstrap.go = wiring bbh) | Logika bootstrap pindah ke `cmd/server/main.go` (registrasi plugin, `/healthz`, urutan) — bukan library mentah |
| `internal/plugins` (types/ServiceDeps/registry) | `github.com/rolldone/donisgo/plugins` |
| `plugins/plugin_registry` (lokal) | `github.com/rolldone/donisgo/plugins/plugin_registry` (D-3) |
| `internal/db` | `github.com/rolldone/donisgo/db` |
| `internal/auth` (JWT utils) | `github.com/rolldone/donisgo/auth` (instans API, D-4) |
| `internal/events` | `github.com/rolldone/donisgo/events` |
| `internal/keydb` | `github.com/rolldone/donisgo/keydb` |
| `internal/mail` | `github.com/rolldone/donisgo/mail` |
| `internal/storage` | `github.com/rolldone/donisgo/storage` |
| `internal/secrets` | `github.com/rolldone/donisgo/secrets` |
| `internal/uuid` | `github.com/rolldone/donisgo/uuid` |
| `internal/console` (root + engine migrate) | `github.com/rolldone/donisgo/console` (migrate/seed) + generator `plugin new` dari template |
| `internal/pluginloader` | DIHAPUS — registrasi eksplisit (v0.2.0 Removed) |
| `internal/admin` (business services) | **Perlu keputusan WO-01** — jika masih ada kode bisnis → per aturan user (fitur = plugin) pindah/diapus; lapor owner |

Catatan mekanis penting: module rename mengubah SEMUA import (`github.com/rolldone/donisgo/internal/X` tidak lagi valid — milik library baru). Langkah module-rename + rewrite-import + hapus `internal/` HARUS satu commit atomik agar build tetap valid.

## Perubahan perilaku yang WAJIB diantisipasi
1. **Serving SPA tidak otomatis lagi** — tanpa `EnsurePluginLast(pages)`: `/app` & fallback SPA 404.
2. **`TRUSTED_PROXIES` default trust-none** (sejak donisgo v0.1.1) — bbh di Docker tanpa reverse proxy → aman default; tambahkan ke `.env.example` dengan penjelasan.
3. **Env port**: README template menyebut `PORT` (default 8080) sedangkan bbh memakai `APP_HOST`/`APP_PORT` (`.env` lokal `APP_PORT=8082`). WO-01 WAJIB cek sumber `app` v0.2.0 yang mana yang dibaca — jangan menebak; bila `PORT`, .env bbh + compose disesuaikan.
4. **Swagger**: nama definisi memuat module path (`github_com_rolldone_donisgo_plugins_resident_model.*` → `github_com_akasatechdev_bbh_data_board_plugins_...`). Baseline manifest `build.sh --verify-docs` HARUS di-re-pin setelah rename.
5. **Migrate**: layout engine library = core `migrations/{db_type}` + plugin `plugins/{id}/migrations/{db_type}` — sama dengan bbh; verifikasi idempotensi (up/down/up) di DB dev.
6. **Console commands plugin** (`auth`, `endpoint`, `sso`, `maintenance`, `hsh version` dll.) — memakai `ConsoleCommands()` kontrak plugin; verifikasi semua CLI jalan dari `cmd/console` baru.

## Rencana Work Order (draf — 4 WO, partial principle)
- **WO-01 — Inventarisasi & pemetaan (read-only)**: daftar file `internal/*` lengkap + mapping tabel di atas diverifikasi satu per satu; diff `plugins/plugin_registry` lokal vs library v0.2.0; daftar pemakaian API `auth` package-level per plugin; cek env port (`APP_PORT` vs `PORT`) di sumber `app` v0.2.0; cek go directive library vs bbh (go 1.25.3); baseline swagger + `go version -m bin/*` (stamp VCS). Output = tabel bukti di dok WO + laporan divergensi (D-3, `internal/admin`) ke owner.
- **WO-02 — Migrasi inti (atomik)**: module rename (D-1) + `require donisgo v0.2.0` + rewrite 104 import + hapus `internal/` yang terganti library + bootstrap ke `cmd/server` + `cmd/console` pola template (+ generator `plugin new`) + swap plugin_registry (D-3) + registrasi eksplisit 13 plugin + `EnsurePluginLast(pages)`. Gates: `go build/vet/test ./...` hijau; server boot + `/healthz` + `/app/` + `/` homepage + API 401/404 tetap; `migrate up/down/up` idempotent; CLI `auth`/`endpoint`/`sso`/`maintenance` jalan; Bruno smoke PASS.
- **WO-03 — Migrasi auth ke API instans (D-4)**: `plugins/auth` memakai `auth.New(auth.ConfigFromEnv())` satu titik, inject ke service; hapus semua pemakaian wrapper deprecated di kode bbh; test isolasi dua instance; fail-closed secret kosong tetap tertegak.
- **WO-04 — Verifikasi & dokumentasi + re-review multi-DB (D-5)**: `build.sh --verify-docs` re-baseline swagger (module path baru); Bruno smoke + RBAC e2e (viewer/limited); Docker build + compose (APP_PORT/PORT sesuai temuan WO-01); `.env.example` (TRUSTED_PROXIES, port); README repoint (dokumentasi app-level ala template) + `DONISGO.md` di-usangkan (repoint ke pkg.go.dev + README template); tulis re-review WO multi-DB (posisi refactor kini upstream library — rekomendasi disimpan di dok WO); housekeeping remote git.

Non-tujuan: perubahan perilaku bisnis plugin apa pun; refactor `migrate` di library (upstream, terpisah); penambahan fitur.

## Pertanyaan terbuka
- O-A: Nasib `internal/admin` (bila berisi kode bisnis lama) — diputuskan saat laporan WO-01.
- O-B: Konfirmasi owner atas hasil diff plugin_registry (hanya jika divergensi ditemukan, D-3 fail-loud).

Lainnya sudah dijawab owner (O-1..O-5 lama → D-1..D-5).

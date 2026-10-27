---
title: "WO-01: Plugin Skeleton — kerangka plugins/resident"
description: "Membuat kerangka plugin `resident` (plugin.go, cli stub, route stub, folder migrations/model/handlers/services), mendaftarkannya di cmd/server & cmd/console, memastikan aplikasi tetap build & boot tanpa error. Belum ada tabel maupun logika domain."
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-01: Plugin Skeleton — kerangka `plugins/resident`

**SPEC acuan:** `SPEC.md` topic ini (§14 Batas Plugin, P-11, D-13)
**Prasyarat:** —

## Tujuan
Menyediakan **kerangka plugin kosong** yang sudah terdaftar & ter-boot, sehingga WO berikutnya hanya menambah migrasi/model/handler di dalam folder yang sudah siap.

## Cakupan
1. Buat struktur folder sesuai §14 SPEC:
   ```
   plugins/resident/
   ├── plugin.go
   ├── cli.go
   ├── handlers/handlers.go
   ├── migrations/postgres/        (kosong dulu)
   ├── model/models.go
   ├── model/view.go
   └── services/service.go
   ```
2. `plugin.go` mengimplementasikan interface `plugins.Plugin` seperti `plugins/endpoint`:
   - `ID() string` → `"resident"`
   - `RegisterServices(deps plugins.ServiceDeps) error` → simpan `deps.DB` (tanpa DDL)
   - `RegisterMiddleware()` → `nil`
   - `RegisterRoutes(_, _, api *gin.RouterGroup) error` → daftarkan grup **stub** `/admin/residents` (handler mengembalikan `501 Not Implemented` — belum ada guard permission di WO ini)
   - `Seed()` → no-op
   - `ConsoleCommands()` → kembalikan `resident` command group (stub)
3. Daftarkan plugin di:
   - `cmd/server/main.go` → `plugins.RegisterPlugins([]plugins.Plugin{..., residentplugin.New()})`
   - `cmd/console/main.go` (atau lokasi registrasi console yang setara)
4. `cli.go` → command `resident` dengan subcommand stub `version` yang mencetak `resident plugin 0.1.0`.
5. Test dasar: `go build ./...`, plugin `ID()` == `"resident"`.

## Non-Cakupan
- Tabel, migrasi, model, service, handler nyata (WO-02+).
- Permission (WO-03).

## Langkah
1. Buat file & folder.
2. Implementasi `plugin.go` meniru pola `plugins/endpoint/plugin.go`.
3. Registrasi di `server` & `console`.
4. Stub CLI & route.
5. `go build ./...` + `go vet ./...` + `gofmt -l`.
6. Jalankan server; pastikan boot tanpa error dan `GET /healthz` tetap 200.

## Bukti yang Harus Dilaporkan
- `go build ./...` lulus (di container `app`).
- `gofmt -l plugins/resident` → kosong.
- Server boot OK; `healthz` 200.
- `./bin/console resident version` → `resident plugin 0.1.0`.
- Output `git status --short`.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | Folder `plugins/resident/` ada dengan 7 berkas sesuai §14 SPEC. |
| AC-2 | Plugin terdaftar di `server` dan `console`; `ID()` = `"resident"`. |
| AC-3 | `go build ./...` & `go vet ./...` lulus; `gofmt -l` kosong. |
| AC-4 | Server boot tanpa error; `/healthz` 200. |
| AC-5 | `resident version` mencetak `resident plugin 0.1.0`. |
| AC-6 | `RegisterServices` **tidak** melakukan DDL apa pun. |
| AC-7 | Tidak ada perubahan pada plugin lain kecuali baris registrasi. |
| AC-8 | Tidak ada kode fitur di `internal/`. |

## Status Eksekusi
**Status: complete** — commit kode `6da7740` (`feat(resident): WO-01 plugin skeleton`). Tidak ada push.

| # | AC | Hasil verifikasi (container `app`) |
|---|---|---|
| AC-1 | 7 entri plugin | `plugin.go`, `cli.go`, `handlers/handlers.go`, `migrations/postgres/.gitkeep`, `model/models.go`, `model/view.go`, `services/service.go` |
| AC-2 | Terdaftar di server/console; `ID()="resident"` | impor + entri slice di `cmd/server/main.go` & `cmd/console/main.go` |
| AC-3 | build/vet/gofmt | `go build ./...` OK, `go vet ./...` OK, `gofmt -l` kosong |
| AC-4 | Boot + `/healthz` | server listen `:8080`; `GET /healthz` = `HTTP 200` |
| AC-5 | `resident version` | `resident plugin 0.1.0` |
| AC-6 | Tanpa DDL | `RegisterServices` hanya `p.db = deps.DB` |
| AC-7 | Hanya baris registrasi | diff terbatas pada `cmd/server/main.go` & `cmd/console/main.go` |
| AC-8 | Tanpa kode di `internal/` | tidak ada perubahan `internal/` |

Catatan: route stub `GET/POST /api/admin/residents` mengembalikan `501` (sengaja belum ber-guard; guard menyusul WO-03/WO-04 untuk menghindari 403 fail-closed).

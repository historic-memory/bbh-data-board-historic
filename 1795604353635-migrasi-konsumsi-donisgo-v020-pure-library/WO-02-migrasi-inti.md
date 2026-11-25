---
title: WO-02 — Migrasi inti (module rename + hapus internal/ + cmd ala template)
description: "Satu commit atomik: go.mod module baru + require donisgo v0.2.0, rewrite ~104 import, hapus internal/ terganti, cmd/server + cmd/console pola donisgo-template, registrasi eksplisit 13 plugin + EnsurePluginLast(pages), hapus internal/pluginloader, swap plugin_registry."
status: complete
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

Semua perubahan dikerjakan sebagai **satu commit atomik** (tidak ada keadaan setengah-rename; `go build ./...` + `go vet ./...` hijau pada commit itu).

### 1. `go.mod`
- `go.mod:1` — `module github.com/AkasaTechDev/bbh-data-board` (sebelumnya `github.com/rolldone/donisgo`). **Kenapa**: module app bukan lagi framework; mencegah import app terbaca sebagai paket library.
- `go.mod:9` — `require github.com/rolldone/donisgo v0.2.0`. **Kenapa**: konsumsi pure-library versi rilis (tag `e9274de`), bukan `replace` ke local path — agar reprodusibel di CI/remote.
- `go mod tidy` dijalankan; seluruh dependensi lama tetap (tidak ada paket hilang).

### 2. Rewrite import (213 file .go)
- Seluruh `github.com/rolldone/donisgo/internal/<pkg>` → `github.com/rolldone/donisgo/<pkg>` (paket yang kini publik di library: `app`, `plugins`, `auth`, `db`, `events`, `keydb`, `mail`, `storage`, `secrets`, `uuid`, `console`, `pluginloader`).
- Seluruh `github.com/rolldone/donisgo/plugins/<id-bbh>` → `github.com/AkasaTechDev/bbh-data-board/plugins/<id-bbh>`. **Kenapa**: plugin milik app, bukan milik framework.
- Verifikasi AC-0: `grep -r "rolldone/donisgo/internal" --include="*.go"` = **0 hasil**.

### 3. Hapus `internal/` (1:1 tergantikan library)
- `internal/` dihapus seluruhnya: `app/`, `auth/`, `console/`, `db/`, `events/`, `keydb/`, `mail/`, `pluginloader/`, `plugins/`, `storage/`, `secrets/`, `uuid/`. **Kenapa**: duplikat library v0.2.0 → kalau dibiarkan, dua sumber kebenaran (drift) dan `go vet`/test ganda.
- Logika wiring `internal/app/bootstrap.go` dipindah ke `cmd/server/main.go` (lihat #4).

### 4. H-3 (MENGIKAT) — varian **A** dipakai
- `cmd/server/main.go:56-75` — `app.New(app.Config{Plugins: []plugins.Plugin{...}})` dengan `pagesplugin.New()` sebagai **elemen TERAKHIR**.
- Urutan final: `healthz` → `auth` → `endpoint` → `portalsso` → `federation` → `resident` → `hsh` → `sec` → `dashboard` → `visit` → `ingest` → `warehouse` → `pages`.
- **Kenapa varian A**: `app.New` memanggil `attachPlugins` (register services + routes) DI DALAM `New`. Pola template (`app.New()` lalu `GetRegistry().EnsurePluginLast(pages)`) terbukti empiris mendaftarkan `pages.Routes=0` → route SPA mati. Varian A (`Config.Plugins` non-nil → registry terisolasi, registrasi berurutan) menghasilkan `pages.Routes=1`. Larangan pola template ditulis eksplisit sebagai komentar di `cmd/server/main.go:40-51` agar tidak diulang.
- AC-2: tidak ada plugin yang didaftarkan otomatis oleh library — daftar 13 plugin 100% eksplisit di `main.go`.

### 5. H-2 (MENGIKAT) — `/healthz` pindah rumah
- `plugins/healthz/plugin.go` (baru) — plugin kecil yang `RegisterRoutes` melakukan `router.GET("/healthz")` → `{"status":"ok"}`. **Kenapa**: `registerStoreRoutes` library kosong & unexported, jadi tidak ada hook lagi untuk `/healthz`.
- Didaftarkan paling awal (`cmd/server/main.go:60`) supaya tidak ditelan `router.NoRoute` milik `pages`.
- Konsumen yang dijaga (didokumentasikan di header file): `plugins/endpoint/services/probe.go`, `plugins/federation/routes.go`, `sub_app/webapp/vite.config.ts`, `sub_app/webapp/src/api/client.ts`.

### 6. H-1 (MENGIKAT) — default `--plugin core`
- `migrations/postgres/.gitkeep` (baru) — direktori migrasi **core** dibuat (kosong, berisi komentar penjelas). **Kenapa**: default `migrate --plugin` di library v0.2.0 adalah `core` (sebelumnya app memakai `all`); tanpa direktori ini `migrate up` polos gagal `no migration paths found for target "core"`.
- Dok dikoreksi agar tidak lagi menjanjikan "core + plugin" untuk `migrate up` polos:
  - `README.md:199` — troubleshooting `migrations/` diperbarui (dir core kini ada, `migrate up` aman).
  - `README.md:225-229` — contoh dipisah: `migrate up` (core saja) / `migrate --plugin resident up` / `migrate --plugin all up`.
  - `README.md:281` (plugin hsh) — jadi `migrate --plugin hsh up`.
  - `README.md:408` (plugin sec) — anotasi "core saja (default)".
  - `AGENT.md:77-79` — tiga varian perintah didokumentasikan.
  - `DONISGO.md:51` dan `DONISGO.md:527-533` — contoh `--plugin <id>` / `--plugin all` ditambahkan.
- Bukti AC-3: `migrate list` → `core: current=0`; `migrate up` polos **EXIT=0**; `migrate down && up && up` idempotent (up ke-2 `0 applied`).

### 7. H-4 — generator `plugin new` pindah ke app
- `console/plugin.go` (baru) + `console/plugin_test.go` (baru) — diadopsi dari `donisgo-template/console/plugin.go`. **Kenapa**: v0.2.0 menghapus command `plugin` dari library; generator adalah tanggung jawab app.
- `frameworkModulePath = "github.com/rolldone/donisgo"` tetap; `modulePath()` membaca `go.mod` app sehingga template plugin baru meng-import module app yang benar.
- `cmd/console/main.go:24` — blank import `_ ".../console"` menempelkan `pluginCmd` ke `frameworkconsole.NewRootCmd()`.
- Bukti: `plugin new --id tmp-scaffold` membuat `plugins/tmp_scaffold/` dengan import framework=donisgo + app=module baru, build+vet hijau, lalu dihapus (tidak ada artefak tertinggal).

### 8. O-B (disetujui owner) — swap `plugin_registry`
- `plugins/plugin_registry/` (lokal: `errors.go`, `registry.go`, `registry_test.go`) **dihapus**; seluruh importir memakai `github.com/rolldone/donisgo/plugins`. **Kenapa**: kode identik dengan library (divergensi tekstual saja, 0 importir tambahan) → risiko drift tanpa manfaat.

### 9. `cmd/console/main.go` pola template
- `cmd/console/main.go:31` — `console.RegisterAdditionalPlugins([]plugins.Plugin{...})` (12 plugin; `pages` & `healthz` tidak perlu di console) lalu `console.Execute()`. **Kenapa**: command plugin (migrate per-plugin, `auth`, `hsh version`, dst.) harus terdaftar sebelum cobra execute.

### 10. AC-4 — catatan jujur soal data dev
- Daftar tabel **identik** sebelum/sesudah (diff daftar tabel = kosong).
- Angka baris "sebelum" pada baseline diambil dari `pg_stat_user_tables.n_live_tup` (**estimasi**, bukan `count(*)`). Nilai nyata saat verifikasi: `residents`/`wh_*`/`facilities` = 0; `mothers=10`, `admins=20`, `groups=6`, `ingest_audit=5029` utuh.
- Uji definitif penyebab: baris sentinel disisipkan ke `residents`, lalu **seluruh** `go test -count=1 ./...` dijalankan → sentinel **selamat** (tidak ada test yang menghapus data). `pg_stat_user_tables` menunjukkan churn lama (`n_tup_ins=6835` / `n_tup_del=6677`) dan `mothers` masih ada (kalau `maintenance clean-data --yes` dijalankan, `mothers` ikut terhapus).
- **Kesimpulan**: angka "sebelum" adalah estimasi stale; pengosongan tabel terjadi **sebelum** sesi WO-02 (bukan akibat migrasi/test). Migrasi hanya menambah bookkeeping `migration_targets` (+1 entri `core`) — perilaku engine library, bukan perubahan data domain.

### 11. AC-1 / AC-5 / AC-6 (bukti ringkas)
- AC-1 dari host `localhost:8082`: `/healthz`=200 `{"status":"ok"}`; `/`=200 (mengandung `href="/app/"`); `/app/`=200; aset 200; `/spa`=301→`/app/`; `/api/nope`=404 `{"error":"endpoint not found"}`; `/api/admin/residents` tanpa token=401.
- AC-5: `POST /api/admin/login` root@admin.local → 200 (token_len=216); `/api/admin/me` → `{"groups":["root"],"permissions":["*"]}`; `/api/admin/residents` dengan token=200.
- AC-6: tidak ada perubahan env/port. `PORT=8080` (dipakai library) & `APP_PORT=8082` (host) tetap; akses host `localhost:8082` masih berlaku.

## Review Manager

(dikosongkan)

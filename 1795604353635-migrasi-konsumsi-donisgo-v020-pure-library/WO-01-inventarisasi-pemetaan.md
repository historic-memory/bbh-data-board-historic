---
title: WO-01 — Inventarisasi & pemetaan (read-only)
description: "Inventaris lengkap internal/* + pemetaan ke library donisgo v0.2.0, diff plugin_registry, cek env port, baseline swagger/binari. TANPA perubahan kode."
status: progress
created: "2026-11-25"
updated: "2026-11-25"
tags: [donisgo, migrasi, wo-01, read-only]
---

# WO-01 — Inventarisasi & pemetaan (read-only)

Ref: `spec.md` §Pemetaan internal/* → target, §Perubahan perilaku. Prasyarat: —.

## Lingkup

WO ini **TIDAK mengubah kode sumber apa pun**. Outputnya adalah data untuk WO-02 (dan keputusan owner bila ada temuan divergensi).

## AC

- **AC-1 — Inventaris paket `internal/`**: daftar lengkap paket di `internal/` (nama, jumlah file, satu kalimat fungsi). Tabel pemetaan di SPEC diperiksa satu per satu; setiap baris mendapat status ✅ cocok-library / ⚠️ ada-divergensi / ❌ milik-app. Bukti: tabel di section `## Hasil Inventaris (Dev)` dokumen ini.
- **AC-2 — Diff `plugins/plugin_registry`**: bandingkan `plugins/plugin_registry/{errors,registry}.go` + test dengan `github.com/rolldone/donisgo/plugins/plugin_registry@v0.2.0` (fetch via `go mod download` ke module cache atau raw.githubusercontent). Bila identik → catat "swap aman". Bila ada perbedaan (termasuk komentar/konstanta ekstra) → **BERHENTI, lapor owner** (D-3 fail-loud); jangan tukar diam-diam. Bukti: output diff (atau kalimat "tidak ada perbedaan") + hash kedua file.
- **AC-3 — Nasib `internal/admin`**: telusuri isi `internal/admin/services` (direferensikan DONISGO.md sebagai "business services"). Bila kosong/stub → catat untuk dihapus di WO-02. Bila ada kode bisnis → daftar dependensinya (plugin mana yang memakai) + usulan penempatan, **tunggu keputusan owner sebelum lanjut** (O-A di SPEC).
- **AC-4 — Env port**: dari sumber `github.com/rolldone/donisgo/app@v0.2.0`, temukan env yang dibaca untuk host/port (`APP_HOST`/`APP_PORT` vs `PORT`). Jawab tegas: env apa yang dipakai v0.2.0, apakah nama lama masih didukung, dan apa yang harus diubah di `.env` bbh / `docker-compose.yml` / webapp `VITE_API_TARGET`. Bukti: kutipan kode sumber (file:baris) di dok WO.
- **AC-5 — Pemakaian API `auth` package-level**: grep semua pemanggil `auth.Sign*`/`auth.Parse*`/`auth.GenerateAccessTokenWithLevel`/`auth.ParseAccessTokenClaims`/`auth.AccessExpirySeconds`/`auth.RefreshExpirySeconds` di kode bbh. Output: daftar call-site per plugin (untuk dijadikan bahan WO-03). Bukti: tabel call-site.
- **AC-6 — Pemakaian API lain yang berubah**: cek call-site `events.*` (package-level vs `NewBus`), `keydb.*`, `db.GetGormDB()`, `storage.*` package-level, `console.*` (root/migrate). Klasifikasi: tetap pakai library package-level (deprecated tapi aman) vs harus di-inject di WO-02/03. Bukti: tabel.
- **AC-7 — Baseline pra-migrasi**: simpan di dok WO — (a) `go version -m bin/server bin/console` (stamp VCS saat ini), (b) jumlah path/ops swagger saat ini dari `docs/swagger.json`, (c) hasil `go build ./... && go vet ./... && go test ./...` sebagai kondisi awal (catat test yang memang merah pre-existing, mis. test resident vs data dev — topic `1793687545424` T-1), (d) `docker compose version` di host.
- **AC-8 — Laporan keputusan**: rangkum semua temuan ⚠️/❌ + O-A/O-B ke chat; bila tidak ada yang perlu keputusan, tulis "tidak ada blocker" eksplisit.

## Non-tujuan
Edit file Go apa pun, ubah go.mod, jalankan migrasi DB, commit kode.

## Hasil Inventaris (Dev)

Semua perintah dijalankan dari root repo `/home/donny/workspaces/bbh-data-board`, kecuali yang ditandai
container `app` (`docker compose exec -T app ...`). Pembanding library diambil dari klon tag
`donisgo@v0.2.0` (commit `e9274de`) di `/tmp/donisgo-v020/dl`.

### AC-1 — Inventaris paket `internal/`

31 file `.go` di 12 paket. Kolom **Status**: ✅ = file library identik (setelah normalisasi
komentar+spasi), ⚠️ = ada divergensi tapi ada padanan library, ❌ = tidak ada padanan library.

| Paket | File | Fungsi (1 kalimat) | Padanan library | Status |
|---|---|---|---|---|
| `internal/app` | 1 (`bootstrap.go`) | Wiring app bbh: DB, storage, keydb, gin, CORS, group, `attachPlugins`, `/healthz` | `app/bootstrap.go` | ⚠️ wiring bbh-spesifik |
| `internal/auth` | 2 (`jwt.go`, `claims_tokens.go`) | JWT sign/parse + klaim level akses | `auth/*` | ⚠️ library menambah instance API (D-4) |
| `internal/console` | 4 (`root.go`, `migrate.go`, `seed.go`, `plugin.go`) | Root cmd, migrator, seeder, generator `plugin new` | `console/{root,migrate,seed}.go` | ⚠️ `plugin.go` ❌ tidak ada di library |
| `internal/db` | 3 (`db.go`, `gorm.go`, `tx.go`) | Koneksi multi-DB, `GetGormDB`, helper transaksi | `db/*` | ⚠️ `tx.go` ✅ identik |
| `internal/events` | 4 (`events.go`, `request_reply.go`, +2 test) | Bus event in-proc (publish/subscribe, request-reply) | `events/*` | ⚠️ library menambah `NewBus` + wrapper deprecated |
| `internal/keydb` | 2 (`client.go`, `flash.go`) | Klien KeyDB/Redis + flash message | `keydb/*` | ⚠️ ada tambahan api instance |
| `internal/mail` | 2 (`mailer.go`, `mail.go`) | Mailer SMTP | `mail/*` | ⚠️ `mail.go` ✅ identik — **0 importir di bbh** |
| `internal/pluginloader` | 2 (`loader.go`, +test) | Loader plugin bbh-specific (`RegisterCorePlugins`/`RegisterLastPlugins`) | — | ❌ milik-app, harus dihapus |
| `internal/plugins` | 4 (`types.go`, `registry.go`, `registry_test.go`, `middleware_priorities.go`) | Interface Plugin, registry, prioritas middleware | `plugins/*` | ⚠️ `middleware_priorities.go` ✅ identik; sisanya ditambah API instance |
| `internal/secrets` | 2 | Master key / kripto | `secrets/*` | ✅ identik keduanya |
| `internal/storage` | 4 (`config.go`, `factory.go`, `local.go`, `types.go`) | Abstraksi storage + backend lokal | `storage/*` | ⚠️ `factory.go` ✅ identik; sisanya divergen |
| `internal/uuid` | 1 (`uuidv7.go`) | Generator UUIDv7 | `uuid/uuidv7.go` | ✅ identik (`New`, `NewString`) |

**Bukti jumlah importir** (`grep -rln 'donisgo/internal/<pkg>"' --include='*.go' .` di luar `.kilo`):

```
internal/app           total=1   luar_internal=1  dalam_internal=0
internal/auth          total=2   luar_internal=2  dalam_internal=0
internal/console       total=1   luar_internal=1  dalam_internal=0
internal/db            total=14  luar_internal=11 dalam_internal=3
internal/events        total=4   luar_internal=1  dalam_internal=3
internal/keydb         total=1   luar_internal=0  dalam_internal=1
internal/mail          total=0   luar_internal=0  dalam_internal=0   <-- TIDAK DIPAKAI
internal/pluginloader  total=2   luar_internal=0  dalam_internal=2
internal/plugins       total=22  luar_internal=15 dalam_internal=7
internal/secrets       total=3   luar_internal=3  dalam_internal=0
internal/storage       total=5   luar_internal=1  dalam_internal=4
internal/uuid          total=13  luar_internal=13 dalam_internal=0
```

**Koreksi terhadap SPEC**: angka "104 file meng-import `internal`" di SPEC terlalu besar karena ikut
menghitung salinan repo usang `.kilo/worktrees/laced-surprise/` (di-exclude via `.git/info/exclude:9`,
bukan bagian repo). Angka riil: **55 file** di luar `.kilo`. Perbaikan angka ini dicatat untuk WO-04.

**Bukti per-file** (gofmt + buang komentar + normalisasi spasi, lalu bandingkan dengan library):

```
SAMA      internal/db/tx.go
SAMA      internal/mail/mail.go
SAMA      internal/plugins/middleware_priorities.go
SAMA      internal/secrets/secrets.go
SAMA      internal/secrets/secrets_test.go
SAMA      internal/storage/factory.go
SAMA      internal/uuid/uuidv7.go
BEDA      internal/auth/{claims_tokens,jwt}.go
BEDA      internal/console/{migrate,root,seed}.go
BEDA      internal/db/{db,gorm}.go
BEDA      internal/events/{events,request_reply,+2_test}.go
BEDA      internal/keydb/{client,flash}.go
BEDA      internal/plugins/{registry,registry_test,types}.go
BEDA      internal/storage/{config,local,types}.go
TIDAK-ADA internal/console/plugin.go
```

Kesimpulan AC-1: tidak ada paket `internal` yang berisi logika bisnis bbh. Semuanya salinan
framework + 2 pengecualian: `internal/pluginloader` (bbh-specific, hapus) dan
`internal/console/plugin.go` (generator `plugin new`, pindah ke `console/` milik app).

### AC-2 — Diff `plugins/plugin_registry` vs library (D-3 fail-loud)

**Hash**:

```
LOKAL                                             LIBRARY v0.2.0
4a5ee501...c8fe9c50  errors.go                    799ebb95...92ca517  errors.go
5da65f58...f20f95f  registry.go                  982d17c5...26c539e8  registry.go
57e11daa...69c9c8ab  registry_test.go             57e11daa...69c9c8ab  registry_test.go  <-- IDENTIK
(tidak ada)                                        e34efb13...140b4429  doc.go            <-- HANYA di library
```

**Perbandingan setelah normalisasi** (gofmt, buang baris `//`, buang baris kosong, rapatkan spasi):

```
errors.go        : IDENTIK setelah komentar+spasi dinormalisasi
registry.go      : IDENTIK setelah komentar+spasi dinormalisasi
registry_test.go : IDENTIK setelah komentar+spasi dinormalisasi
```

**Isi perbedaan** (`diff -u`) — semuanya komentar/dokumentasi:
- `errors.go`: header blok `═══ ERROR BERSAMA ANTAR PLUGIN ═══` (versi lokal, bahasa Indonesia) diganti
  komentar bahasa Inggris; ditambah doc-comment per field `Code`/`Message`.
- `registry.go`: doc-comment panjang bahasa Indonesia di atas `package` + panduan "CARA MENAMBAH
  CONTRACT BARU" dipindah/dipadatkan ke bahasa Inggris; isi `EchoProvider`, `echoProvider`,
  `RegisterEchoProvider`, `Echo` (termasuk nil-guard) **identik**.
- Library menambah `doc.go` (1630 B) — paket dokumentasi, tidak mengubah API.
- Kode executable: **tidak ada satu pun perbedaan**.

**Pemakaian**: `grep -rln 'plugin_registry' --include='*.go'` di luar `.kilo` → **0 importir**.
Paket ini scaffold mati (hanya didaftarkan sebagai direktori di root `plugins/`, bukan plugin
terdaftar di `cmd/server/main.go`).

**Status AC-2**: ada divergensi tekstual → sesuai D-3 **dilaporkan ke owner (O-B)**, tidak ditukar
diam-diam. Rekomendasi dev: swap aman (kode identik, 0 importir, library hanya menambah `doc.go`),
tapi eksekusi menunggu persetujuan owner di WO-02.

### AC-3 — Nasib `internal/admin`

```
$ ls internal/
app  auth  console  db  events  keydb  mail  pluginloader  plugins  secrets  storage  uuid
$ find . -type d -name admin -not -path './.kilo/*'
(hasil kosong)
```

`internal/admin` **tidak ada** di repo. Referensi di `DONISGO.md:98` dan `DONISGO.md:921`
("business services") adalah dokumentasi usang dari versi scaffold lama. **O-A selesai: tidak ada
kode bisnis yang perlu diselamatkan, tidak ada yang perlu dihapus.** Yang perlu dilakukan di WO-04:
hapus/koreksi 2 baris DONISGO.md tersebut.

### AC-4 — Env port di `app@v0.2.0`

Jawaban tegas: **v0.2.0 hanya membaca `PORT` (via gin), `APP_HOST` dan `APP_PORT` tidak pernah
dibaca oleh kode Go — baik di versi lama maupun baru.**

Bukti kutipan:

1. Library, `app/bootstrap.go:438-442` — `Run()` hanya memanggil router:
   ```go
   func (a *App) Run() error {
           ...
           return a.router.Run()   // :442 — tanpa argumen alamat
   }
   ```
2. `app/bootstrap.go:115` — `r := gin.Default()` (sama dengan bbh `internal/app/bootstrap.go:91`).
3. gin v1.9.1 `utils.go:143-147` (sumber alamat listen sebenarnya):
   ```go
   if port := os.Getenv("PORT"); port != "" { ... }
   debugPrint("Environment variable PORT is undefined. Using port :8080 by default")
   ```
4. `app/bootstrap.go:201-203` hanya membaca `KEYDB_PORT` — bukan port HTTP.
5. `grep -rn 'APP_HOST\|APP_PORT' --include='*.go'` di repo bbh → **0 hasil**. Nilai itu hanya dipakai
   `docker-compose.yml` sebagai nama variabel untuk port-mapping host.

**Kesimpulan AC-4**: tidak ada perubahan env yang dibutuhkan untuk port. `docker-compose.yml` sudah
menyuntik `PORT: "8080"` ke container, dan `APP_PORT=8082` murni sisi host. Yang perlu diperbaiki
hanya dokumentasi (`DONISGO.md`/`README.md` masih menyiratkan `APP_PORT`/`APP_HOST` dibaca app).
Catatan tambahan: `gin.Run(":8080")` bind ke `0.0.0.0`, jadi `APP_HOST` memang tidak diperlukan.

### AC-5 — Call-site `auth` package-level (bahan WO-03)

| # | File:baris | API | Konteks |
|---|---|---|---|
| 1 | `plugins/auth/middleware/middleware.go:146` | `auth.ParseAccessTokenClaims(token)` | middleware validasi bearer token |
| 2 | `plugins/auth/handlers/auth.go:44` | `auth.AccessExpirySeconds()` | respons login (field `ExpiresIn`) |
| 3 | `plugins/auth/handlers/auth.go:140` | `auth.AccessExpirySeconds()` | respons refresh token |
| 4 | `plugins/auth/handlers/auth.go:210` | `auth.AccessExpirySeconds()` | hitung TTL saat generate token |
| 5 | `plugins/auth/handlers/auth.go:211` | `auth.GenerateAccessTokenWithLevel(subjectID, level, ttl)` | generate access token ber-level |

Total: **5 call-site di 2 file**, semuanya di plugin `auth`. Tidak ada pemakaian `auth.Sign*`,
`auth.Parse*` lain, atau `auth.RefreshExpirySeconds()` di luar paket itu. Library v0.2.0 **masih
menyediakan** semua fungsi package-level ini (dipertahankan sampai v1.0), jadi migrasi ke instance
API (D-4) bersifat opsional-teknis, bukan wajib-kompilasi.

### AC-6 — Call-site API lain & klasifikasi

| API | Call-site | Klasifikasi |
|---|---|---|
| `db.GetGormDB()` | `plugins/endpoint/cli.go:38`, `plugins/portalsso/cli.go:31`, `plugins/auth/cli.go:70`, `plugins/maintenance/cli.go:48`, `internal/app/bootstrap.go:56`, `internal/console/{seed.go:43, migrate.go:157,440,501,560,580,603}`, + 6 `*_db_test.go` | **tetap** — library masih menyediakan `db.GetGormDB()`; alternatif inject `deps.DB` dicatat untuk WO-02 (khusus non-CLI) |
| `events.Subscribe` | `plugins/auth/events.go:13` | **tetap** — wrapper deprecated masih ada; instance `Bus` disarankan di WO-02/03 |
| `events.Publish` | hanya di dalam `internal/events/events.go:20` (logging) | ikut pindah ke library, tidak ada call-site plugin |
| `keydb.Init(...)` | `internal/app/bootstrap.go:84` | **inject** — di library, `app.Config.KeyDB` menggantikan; blok manual ini hilang bersama `internal/app` |
| `storage.LoadConfig()` / `storage.NewStore()` | `internal/app/bootstrap.go:62,67`, `internal/console/seed.go:76-77` | **tetap** untuk `console/seed.go` (CLI di luar app); **inject** untuk bootstrap (`app.Config.Store`) |
| `console.RegisterAdditionalPlugins()` / `console.Execute()` | `cmd/console/main.go:26,27` | **tetap** — API library sama |
| `plugins.RegisterPlugins()` (package-level) | `cmd/server/main.go` (via hook lama) | **ganti** ke `app.Config.Plugins` (lihat hazard H-3) |

Tidak ditemukan pemakaian `mail.*` dari plugin (0 importir `internal/mail`), jadi `app.Config.Mail`
bisa dibiarkan default.

### AC-7 — Baseline pra-migrasi

**(a) Stamp binari** (`go version -m`):

```
bin/server : go1.27.3  path=github.com/rolldone/donisgo/cmd/server
             mod github.com/rolldone/donisgo v0.0.0-20261124073332-bbe63762223c
bin/console: go1.27.3  path=github.com/rolldone/donisgo/cmd/console
             mod github.com/rolldone/donisgo v0.0.0-20261124082238-132fdb086e38+dirty
```

Catatan: binari dibangun dengan go1.27.3 (host snap), sedangkan container `app` memakai
go1.27.4 → baseline build kanonik = container.

**(b) Swagger** `docs/swagger/swagger.json`: **paths=141, operations=209, definitions=248**.

**(c) Build/vet/test** (container `app`, go1.27.4):

```
go build ./...   -> OK
go vet ./...     -> OK
go test ./...    -> 1 FAIL pre-existing: plugins/pages TestProxyPath
```

`plugins/pages TestProxyPath` merah **sebelum** migrasi: route `/` (homepage embedded) menangkap
ecact-`/` sehingga proxy `HOMEPAGE_URL` tidak pernah terpanggil. Ini bukan akibat migrasi; dicatat
sebagai kondisi awal, perbaikannya di luar lingkup topik ini.

**(d) Environment**: `docker compose version --short` → `2.32.4`; container `app` → `go1.27.4 linux/amd64`.

### AC-8 — Temuan ⚠️/❌ + hazard yang harus ditangani WO-02

**Blocker/keputusan owner:**

- **O-B (perlu keputusan)** — `plugins/plugin_registry` berbeda dari library secara teks (komentar
  bahasa Indonesia vs Inggris) + library punya `doc.go`. Kode executable identik, 0 importir.
  Pertanyaan: setuju swap ke library di WO-02 (menghapus 3 file lokal)?
- **O-A (selesai)** — `internal/admin` tidak ada; tidak ada yang perlu diputuskan.

**Hazard wajib ditangani WO-02 (di luar AC, ditemukan saat inventaris):**

- **H-1 — `console migrate` default berubah `all` → `core`.** Bukti: bbh
  `internal/console/migrate.go:211` → `StringVar(&migratePluginFlag, "plugin", "all", ...)`;
  library `console/migrate.go:211` → `StringVar(&migratePluginFlag, "plugin", "core", ...)`.
  Repo bbh **tidak punya** direktori `migrations/` (hanya `plugins/*/migrations/`), sehingga
  `migrate up` tanpa flag akan gagal `no migration paths found for target "core"`
  (`console/migrate.go:250-255`). Dampak: `README.md:225,281,406`, `AGENT.md:77`,
  `DONISGO.md:51,527` harus diubah menjadi `--plugin <id>` / eksplisit, atau bbh menyediakan
  `migrations/` kosong-bertanda. Ini regresi operasional nyata untuk operator.
- **H-2 — `/healthz` kehilangan rumah.** bbh mendaftarkannya di `internal/app/bootstrap.go`
  (`registerStoreRoutes`); di library fungsi itu kosong dan unexported, `app.Config` tidak punya hook
  route tambahan. Konsumen `/healthz`: probe `plugins/endpoint`, tabel owner route federation,
  proxy `sub_app/webapp/vite.config`, `client.ts`. WO-02 harus memindahkannya ke plugin kecil / ke
  `plugins/pages` — bukan menghapusnya.
- **H-3 — `EnsurePluginLast` SETELAH `app.New` tidak mendaftarkan route (bug pola template).**
  Hasil uji empiris (program di `/tmp/proof`, library v0.2.0, container `app`):
  ```
  A Config.Plugins[.., pages]   : mine.Routes=1 pages.Routes=1 order=[mine pages]
  B default registry + New({})  : mine.Routes=1 pages.Routes=1 order=[mine pages]
  C New lalu EnsurePluginLast   : mine.Routes=1 pages.Routes=0 order=[mine pages]   <-- pola donisgo-template
  ```
  Artinya pola yang didokumentasikan `donisgo-template` (`app.New` dulu, baru
  `GetRegistry().EnsurePluginLast(pages)`) membuat `pages` **tidak pernah** me-register route
  (catch-all `/app`, `/`, `/spa 404`). WO-02 **wajib** memakai varian A (pages sebagai elemen
  terakhir `app.Config.Plugins`) atau B (pre-register ke default registry sebelum `New`).
- **H-4 — Generator `plugin new` pindah tangan.** `internal/console/plugin.go` (452 baris, termasuk
  `events.Subscribe` contoh di :452) tidak ada di library. Harus menjadi `console/plugin.go` milik app
  (pola `donisgo-template/console/plugin.go`), didaftarkan ke root cmd bbh.
- **H-5 — `TRUSTED_PROXIES` jadi default aman.** bbh: `gin.Default()` tanpa `SetTrustedProxies` →
  gin mempercayai semua proxy. Library: `app/bootstrap.go:294-303` memanggil
  `SetTrustedProxies(nil)` bila `TRUSTED_PROXIES` kosong (= trust none). Audit bbh **tidak** memakai
  `ClientIP()`/`RemoteAddr` (0 hasil grep), jadi tidak ada perubahan data; tapi `TRUSTED_PROXIES`
  perlu didokumentasikan bila nanti ditaruh di belakang reverse proxy.
- **H-6 — Swagger perlu re-baseline.** `build.sh:98` memakai `swag init -g cmd/server/main.go`; setelah
  `internal/` hilang dan `cmd/server` berganti isi, 141/209/248 akan bergeser dan harus dibangun ulang
  di container (host tidak bisa: GOROOT snap).
- **H-7 — Angka "104 file" di SPEC** harus dikoreksi menjadi **55** (lihat AC-1) di WO-04.

Kesimpulan: **ada blocker keputusan (O-B)** dan **7 hazard**, semuanya sudah dipetakan dengan bukti;
tidak ada yang menghalangi WO-02 secara teknis begitu O-B diputuskan.

## Catatan Implementasi (Dev)

1. `internal/` (inventaris) — `for d in $(find internal -type d)` + `ls $d/*.go | wc -l`: 12 paket,
   31 file. Dipakai untuk tabel AC-1. Tidak ada sub-paket tersembunyi (mis. `internal/admin`).
2. `internal/<pkg>` (hitung importir) — `grep -rln "donisgo/internal/$p\"" --include='*.go'`, hasil
   dipisah luar/dalam `internal`. Menemukan `internal/mail` = **0 importir** (kandidat hapus) dan
   `internal/pluginloader` hanya dipakai `internal/app` (bbh-specific).
3. `plugins/plugin_registry/*.go` (AC-2) — `sha256sum` lokal vs `/tmp/donisgo-v020/dl`: 2 dari 3 hash
   berbeda, `registry_test.go` identik. Divergensi terbukti tekstual.
4. `plugins/plugin_registry/*.go` (verifikasi kode) — normalisasi `gofmt | grep -v '^\s*//' | tr -s`,
   ketiga file IDENTIK. Juga dicoba AST-diff (`/tmp/astcmp`): `registry.go`/`registry_test.go`
   AST-identical, `errors.go` berbeda hanya karena pergeseran alignment struct akibat doc-comment baru
   — karena itu dipakai normalisasi teks sebagai bukti utama.
5. `plugins/plugin_registry` (blast radius) — `grep -rln plugin_registry --include='*.go'` → 0 importir
   → swap tidak berdampak ke kode mana pun; tetap dilaporkan (D-3 fail-loud).
6. `DONISGO.md:98,921` + `find -type d -name admin` — membuktikan `internal/admin` fiktif; O-A ditutup.
7. `internal/app/bootstrap.go:91` + `app/bootstrap.go:115` + `gin@v1.9.1/utils.go:143-147` — membaca
   rantai alamat listen: `gin.Default()` → `router.Run()` tanpa argumen → `PORT` (default `:8080`).
   Menutup AC-4 tanpa perlu mengubah `.env`/compose.
8. `grep -rn 'APP_HOST\|APP_PORT' --include='*.go'` → 0 hasil di seluruh repo; membuktikan nama env
   itu hanya dipakai compose (host), bukan kode.
9. `docker-compose.yml` — `PORT: "8080"` sudah disuntik ke service `app`; jadi tidak ada perubahan
   compose untuk migrasi.
10. `plugins/auth/**` (AC-5) — `grep -rn 'auth\.(Sign|Parse|GenerateAccessTokenWithLevel|...)'` →
    5 call-site / 2 file, semuanya di plugin auth. Jadi WO-03 cukup menyentuh 2 file.
11. `plugins/**`, `internal/console/**` (AC-6) — grep terpisah untuk `db.GetGormDB`,
    `events.*`, `keydb.Init`, `storage.*`, `console.*`; hasil diklasifikasi tetap-vs-inject. Semua
    API package-level yang dipakai bbh **masih ada** di library (deprecated, dipertahankan s/d v1.0).
12. `bin/server`, `bin/console` — `go version -m` mencatat stamp modul+VCS pra-migrasi sebagai
    pembanding pasca-migrasi (AC-7a).
13. `docs/swagger/swagger.json` — dihitung via python json: 141 paths / 209 ops / 248 definitions
    (AC-7b). Patokan untuk mendeteksi regresi dokumentasi setelah `internal/` dihapus.
14. Container `app` — `go build ./... && go vet ./... && go test ./...` sebagai baseline. Satu merah:
    `plugins/pages TestProxyPath`. Diverifikasi bukan disebabkan migrasi (route `/` homepage memang
    menangkap exact `/` pada kode saat ini) → dicatat sebagai kondisi awal, bukan regresi.
15. `/tmp/proof/main.go` (baru, di luar repo) — program bukti 3 pola registrasi plugin terhadap
    library v0.2.0; dijalankan di container via `docker cp` + `go run`. Output membuktikan
    `EnsurePluginLast` setelah `New` tidak me-register route (hazard H-3).
16. `console/migrate.go:211` (lokal vs library) — membandingkan nilai default flag `--plugin`:
    `all` → `core`. Dikombinasikan dengan `ls -d migrations` (tidak ada) → terbukti `migrate up` tanpa
    flag akan gagal pasca-migrasi (hazard H-1).
17. `grep -rn 'migrate up' --include='*.md' --include='*.bru' --include='*.sh'` — memetakan dokumen
    yang harus diperbarui karena H-1: `README.md:225,281,406`, `AGENT.md:77`, `DONISGO.md:51,527`.
18. `app/config.go:39-44` + `app/bootstrap.go:294-303` — membaca field `TrustedProxies` dan
    perilaku default `SetTrustedProxies(nil)`; dibandingkan dengan bbh yang tidak menyetel apa pun
    → dicatat sebagai perubahan perilaku (H-5), dampak nol karena tidak ada pemakaian `ClientIP()`.
19. `internal/console/plugin.go` — `diff` terhadap library: file tidak ada di library → dipetakan ke
    `console/plugin.go` milik app mengikuti `donisgo-template` (H-4).
20. `.kilo/worktrees/laced-surprise/` — ditemukan ikut terhitung pada `grep -r` awal (angka 104 di
    SPEC); setelah dikecualikan hasilnya 55 file. Semua perintah inventaris di atas dijalankan dengan
    filter `grep -v '^./\.kilo'`.

## Review Manager

**LULUS dengan 1 blocker keputusan (O-B) — 2026-11-25.** Dev: commit dok historic `38b9ee3`, working tree kode bersih, tanpa perubahan kode ✅ (sesuai lingkup read-only).

Verifikasi spot-check Manager (independen, hari yang sama):
- `grep -rln "donisgo/internal/" --include="*.go" . | grep -v .kilo` → **55 file** ✅ (koreksi 104→55 sah; 104 terinflasi worktree `.kilo/laced-surprise`).
- `grep APP_HOST|APP_PORT --include="*.go"` → **0 hasil** ✅ (AC-4 tegas: hanya `PORT` via gin; compose sudah suntik `PORT: "8080"`; tidak ada perubahan env).
- `plugin_registry` importir nyata di luar paketnya → **0** ✅ (3 match grep = package declaration milik paket itu sendiri; klaim "scaffold mati" benar).
- `ls -d migrations` → **tidak ada** ✅ (H-1 valid: `migrate up` tanpa flag akan gagal karena default library `core`).
- `go.mod` masih `module github.com/rolldone/donisgo` ✅ (baseline konsisten).

Penilaian AC: AC-1 ✅ (tabel 12 paket/31 file berbukti; temuan `mail` 0 importir + `pluginloader` bbh-specific + `console/plugin.go` pindah tangan = input langsung WO-02). AC-2 ✅ prosedur fail-loud ditegakkan dengan benar — divergensi tekstual dilaporkan, tidak ditukar diam-diam; bukti normalisasi menunjukkan kode executable identik. AC-3 ✅ O-A ditutup (`internal/admin` fiktif, hanya koreksi 2 baris `DONISGO.md` di WO-04). AC-4 ✅ rantai bukti bootstrap→gin→PORT meyakinkan. AC-5 ✅ 5 call-site / 2 file — lingkup WO-03 kecil dan jelas. AC-6 ✅ klasifikasi tetap-vs-inject rapi (`keydb.Init` & `storage.NewStore` bootstrap → `app.Config.KeyDB`/`.Store`). AC-7 ✅ baseline lengkap (binari, swagger 141/209/248, 1 test merah pre-existing `TestProxyPath` — dicatat bukan regresi migrasi). AC-8 ✅ blocker+hazard dipetakan.

**Temuan paling berharga: H-3** — pola `app.New` dulu → `EnsurePluginLast` TIDAK me-register route `pages` (bukti empiris A/B/C). Konsekuensi:
1. WO-02 **wajib** pakai varian A (pages sebagai elemen terakhir `app.Config.Plugins`) atau varian B (pre-register ke default registry sebelum `New`) — bukan pola README `donisgo-template`.
2. Ini bug dokumentasi **upstream `donisgo-template`** — Manager usulkan issue/PR kecil ke repo template setelah migrasi bbh selesai (terpisah dari topic ini; catat di WO-04 housekeeping).

Catatan review tambahan: laporan menyebut "route `/` homepage menangkap exact `/`" menyebabkan `TestProxyPath` merah — koreksi kecil: itu perilaku `homepage.html` WO-01 topic `1795505104037` yang **disengaja** (exact-`/` dilayani homepage, bukan proxy). Konsisten dengan H-7 (angka 104→55 dikoreksi WO-04).

Status WO-01: **selesai dijalankan, lulus review; WO-02 tertahan keputusan O-B owner** (swap `plugin_registry` → library: kode identik, 0 importir, rekomendasi Manager = SETUJU swap).

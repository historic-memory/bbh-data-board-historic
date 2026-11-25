---
title: WO-04 — Verifikasi & dokumentasi + re-review WO multi-DB (D-5)
description: "Swagger re-baseline pasca module rename, Docker/compose, .env.example, README repoint + DONISGO.md di-usangkan, Bruno regresi RBAC, re-review strategi WO multi-DB (1792765876397), housekeeping."
status: complete
created: "2026-11-25"
updated: "2026-11-25"
tags: [donisgo, migrasi, wo-04, verifikasi]
---

# WO-04 — Verifikasi & dokumentasi + re-review WO multi-DB (D-5)

Ref: `spec.md` §Perubahan perilaku 4–6, D-5, D-6. Prasyarat: WO-03 complete.

## Lingkup

**A — Swagger re-baseline**
- Regenerasi `docs/swagger` via `build.sh` di container `app` (lingkungan kanonik; jangan host) — anotasi mengutip `model.*` jadi `github_com_akasatechdev_bbh_data_board_plugins_...`.
- Angka paritas WAJIB diukur ulang (gotcha terdokumentasi: angka handoff sering usang; filter substring ikut menangkap path lain).
- `build.sh --verify-docs` exit 0, byte-identik setelah re-baseline; commit hash manifest baru tercatat.
- JEBAKAN swag berulang: file yang mengutip `model.X` di godoc wajib import paket model + blank-var; perbaiki semua yang kena (pola resident/warehouse/ingest).

**B — Docker & compose**
- Dockerfile/Dockerfile.runtime/compose ditinjau: bila ada path `internal/` yang di-COPY/referenced → disesuaikan; env port sesuai temuan WO-01; `TRUSTED_PROXIES` (default trust-none) masuk `.env.example` + dokumentasi (bbh di-compose TANPA reverse proxy → default aman, catat implikasi bila nanti di-deploy di belakang proxy).

**C — Dokumentasi**
- README root: repoint dokumentasi aplikasi ala template (quickstart, env, migrate, plugin dev) + catatan ekosistem (donisgo library di-pin; template = referensi struktur).
- `DONISGO.md`: tambahkan banner USANG di atas (konten monorepo tidak lagi valid; arahkan ke pkg.go.dev + README donisgo-template). Jangan dihapus total (masih ada jejak sejarah) — cukup header.
- `.env.example`: sinkron dengan env library v0.2.0 (port/TRUSTED_PROXIES/JWT legacy note).

**D — Bruno regresi**
- `bru run --env Local --tags=smoke` PASS; sampling folder domain (dashboard/secondary/warehouse visit) sesuai konvensi read-only; RBAC viewer/limited sampling (200/403); token di-refresh dulu (gotcha kedaluwarsa).

**E — Re-review WO multi-DB (D-5)**
- Baca ulang topic `1792765876397` (SPEC complete, WO draft) — target lama `internal/console/migrate.go` & `internal/db/gorm.go` kini milik library donisgo.
- Tulis **rekomendasi tertulis** di dok ini: opsi (1) up-stream request/kontribusi ke donisgo; (2) fork kecil di sisi app (mis. engine migrate bbh sendiri di `cmd/console`); (3) tunda hingga library menyediakan multi-conn. Plus daftar gap spesifik bbh (per-branch DB `bbh_<key>`).
- **KEPUTUSAN FINAL diambil owner** — Manager hanya menyajikan opsi + usulan.

**F — Housekeeping**
- `.env.example` JWT 900→7200 (sisa WO-04 topic `1793687545424`) — ikut commit di sini bila masih uncommitted.
- Cek binari `bin/` (gitignored) tidak ikut commit; `git add` path spesifik saja (gotcha insiden 80MB).

## AC

- **AC-1**: `build.sh --verify-docs` exit 0; manifest baru; angka path/ops terukur & tercatat.
- **AC-2**: build image `app` sukses; server di container boot; `/healthz` 200 dari host.
- **AC-3**: `.env.example` berisi TRUSTED_PROXIES + port yang benar; README & DONISGO.md sesuai C.
- **AC-4**: Bruno smoke PASS (angka req/test tercatat); RBAC sampling lolos.
- **AC-5**: re-review multi-DB tertulis lengkap (3 opsi + gap + usulan); status di dok ini "menunggu keputusan owner".
- **AC-6**: housekeeping F selesai (JWT env & hygiene git).
- **AC-7**: data uji & scaffolding bersih; tidak ada token/PII di-commit.

## Catatan Implementasi (Dev)

### §F — Housekeeping (dikerjakan PALING AWAL, sesuai instruksi)

**F.1 — commit `style: gofmt` terpisah** → `fafafd8`
- 28 file disentuh, +28/−29 (murni whitespace & urutan grup import). **Kenapa**: rewrite import WO-02 mengubah urutan grup import tanpa reformat, meninggalkan 27 file tidak gofmt-bersih; ini akan mengaburkan diff commit fungsional berikutnya.
- Verifikasi: `gofmt -l . | grep -v '^\.kilo'` = **0**; `go build ./...` + `go vet ./...` hijau.
- Termasuk `plugins/auth/plugin_test.go` (hanya hapus trailing newline). Tidak ada perubahan logika.

**F.2 — `.env.example` JWT 900→7200**: sudah bernilai `7200` dan **tidak ada diff** (`git diff .env.example` kosong) — sisa WO-04 topic `1793687545424` ternyata sudah ter-commit sebelumnya. Tidak ada aksi.

**F.3 — hygiene git**: seluruh `git add` memakai **path spesifik** (tidak `-A`/`.`) agar `.historic/` (repositori git terpisah + insiden 80MB) tidak ikut; `bin/` tetap gitignored dan tidak masuk commit.

### §A — Swagger re-baseline (H-6)

- Dijalankan di **container `app`** (lingkungan kanonik): `docker compose exec -T app sh -c 'export PATH=$PATH:/usr/local/go/bin:/go/bin && bash build.sh --verify-docs'`. ⚠️ Gotcha: `bash -lc` me-reset PATH → `go: command not found`; gunakan `sh -c` + export PATH eksplisit.
- **Jebakan swag ditemukan & diperbaiki (2 file sumber)**:
  - `plugins/ingest/handlers/analysis_apply.go:11-18` — tambah import `plugins/ingest/model` + `var _ = model.AnalysisChatBlock{}` (blank-var) **dan** ganti nama tipe di anotasi menjadi `github_com_AkasaTechDev_bbh-data-board_plugins_ingest_model.AnalysisChatBlock`. **Kenapa**: `--parseDependency` mewajibkan paket yang disebut di godoc di-import oleh file beranotasi; nama tipe juga harus mengikuti module path baru.
  - `plugins/ingest/handlers/record_write.go:181` — nama tipe `github_com_rolldone_donisgo_...RecordChatBlock` → `github_com_AkasaTechDev_bbh-data-board_...RecordChatBlock` (paket `model` sudah di-import file ini). **Kenapa**: sisa module path lama di anotasi → `cannot find type definition`.
  - Catatan: 444 kemunculan `github_com_rolldone_donisgo` lain semuanya berada di **artefak generated** (`docs/swagger/*`) dan hilang sendiri setelah regenerasi.
- **Hasil regenerasi** (`BUILD OK: biner + dokumentasi swagger diperbarui.`):

  | Metrik | Baseline lama | **Hasil terukur** | Selisih |
  |---|---|---|---|
  | paths | 141 | **141** | 0 |
  | operations | 209 | **209** | 0 |
  | definitions | 248 | **248** | 0 |

  Definisi ber-prefix baru: **170** `github_com_AkasaTechDev_bbh-data-board_*`; sisa `rolldone` di definisi = **0**.
- **Idempotensi terbukti**: run kedua melaporkan `Swagger docs: tidak ada perubahan isi (byte-identik).`; md5 ketiga artefak identik sebelum/sesudah (`IDEMPOTENT_OK`), exit code 0.
- **Manifest baru (SHA-256)** — dicatat per AC-1:
  - `docs/swagger/docs.go` → `f5033294ec9fd599e10c02b85abb24dd67ddcf97187cc5f920e43983b40c2d2c`
  - `docs/swagger/swagger.json` → `86a6dc3f3616fd2e62c692feb9f4d723d249276e0e1e82788b8ddd7523dc77c0`
  - `docs/swagger/swagger.yaml` → `298c32d07e27eeae00f308c688293e15399d3ca63fbfdc9af4941e68adeb5cfb`

> ⚠️ `docs/swagger/*` **gitignored** (artefak generated, lihat README), jadi manifest hanya tercatat sebagai bukti; tidak ikut commit.

### §B — Docker & compose

- `grep -n "internal/" Dockerfile Dockerfile.runtime docker-compose.yml` → **0 hasil**: tidak ada COPY/referensi `internal/` yang perlu disesuaikan (Dockerfile memakai bind-mount `WORKDIR /src`, runtime image hanya COPY `sub_app` manifests).
- `docker compose build app` → **sukses** (`naming to docker.io/library/bbb-data-board-app`, `app  Built`).
- Server di container boot (async `go run ./cmd/server`), `curl localhost:8082/healthz` → **200** `{"status":"ok"}` (dari host).
- Catatan operasional: container `app` IDLE (`sleep infinity`, `restart: no`) — `docker compose up -d --force-recreate app` **mematikan** server yang sedang jalan; harus di-start manual.

### §C — Dokumentasi

- **`.env.example:17-21`** — komentar `APP_PORT` diperjelas: default `8080` (sesuai `${APP_PORT:-8080}` di compose), dan mesin dev memakai `8082` yang sudah tercantum di `CORS_ALLOWED_ORIGINS`. Nilai tidak diubah agar tidak menyimpang dari default compose.
- **`.env.example:72-83`** — blok baru `# === Trusted proxies ===` + `TRUSTED_PROXIES=` (kosong). **Kenapa**: library memetakan `TRUSTED_PROXIES` → `gin.SetTrustedProxies`; kosong = trust-none (`SetTrustedProxies(nil)`), aman untuk stack compose **tanpa** reverse proxy karena `ClientIP` diambil dari `RemoteAddr` sehingga header `X-Forwarded-For` tidak bisa dipalsukan. Didokumentasikan pula: isi hanya bila di belakang nginx/caddy/traefik/NPM, dan nilai tak valid → warning + fallback trust-none (tidak bisa memperluas rentang secara diam-diam).
- **`README.md:1-40`** — ditulis ulang bagian atas: catatan **ekosistem donisgo** (library di-pin di `go.mod`, `donisgo-template` = referensi struktur saja, `DONISGO.md` USANG), tabel struktur project **tanpa `internal/`** (tambah `console/`, `migrations/postgres/`, `healthz`, `maintenance`), plus blok **peringatan urutan registrasi plugin** (pages terakhir, jangan tambah setelah `app.New`).
- **`README.md` (akhir file)** — seksi baru **"Pengembangan plugin"**: scaffold `plugin new`, registrasi di `cmd/server`/`cmd/console`, migrasi `--plugin <id>`, build/test di container, daftar method interface `plugins.Plugin`, dan rujukan `plugins/auth/plugin.go` sebagai contoh satu-instance token.
- **`DONISGO.md:1-24`** — banner **⚠️ DOKUMEN INI USANG (arsip sejarah)** di paling atas: menjelaskan ekstraksi ke pure library v0.2.0, seluruh path `internal/...` tidak ada lagi, dan mengarahkan ke pkg.go.dev + README `donisgo-template` + `README.md` repo ini. File **tidak dihapus** (jejak sejarah).
- **`DONISGO.md` — koreksi `internal/admin` (2 lokasi)**: `Main components` (baris ~99-105) dan `Quick file references` (baris ~926-931) diperbarui: `internal/admin/services` → `plugins/auth/services` (paket `internal/admin` tidak pernah ada di repo ini), dan seluruh path `internal/*` lain dipetakan ke paket library yang menggantikannya.
- **H-7 (angka 104→55)**: tidak ada di `README.md`/`AGENT.md`/`DONISGO.md` — angka itu hanya hidup di dok SPEC/notes topic. Dikoreksi di sumbernya: `spec.md:25`, `spec.md:78`, `notes.md:62` → **55 file** (dengan penjelasan bahwa 104 ikut menghitung salinan `.kilo/worktrees/` di luar repo). `spec.md:78` sekaligus diperbarui agar tidak lagi menyebut `EnsurePluginLast(pages)` (varian yang terbukti tidak bekerja) melainkan varian A.

### §D — Bruno regresi

- **Gotcha token**: `environments/Local.bru` memakai `{{process.env.BBH_TOKEN}}` — tidak terbaca pada bru 4.2.1 → seluruh smoke 401. Solusi: berikan nilai lewat `--env-var token=...` (pola yang juga dipakai README untuk folder domain).
- Token di-refresh lebih dulu via `POST /api/admin/login` (root@admin.local).
- **`bru run --env Local --tags=smoke`** → **✓ PASS**: **13 request / 36 test** (13 passed).
- **Sampling RBAC viewer** (`ops@bbh.dev`, grup `viewer`): `GET /api/admin/residents`=**200**, `POST /api/admin/residents`=**403**, `GET /api/admin/wh/summary`=**200**, `POST /api/admin/wh/items`=**403**.
- **Sampling limited** (grup sementara `wo04limited`: `dashboard.read,resident.read,sec.read` tanpa `hsh.read`): `bru run --env Local dashboard` → **✓ PASS 12/12** (7 request: 5 passed + 2 skipped by pre-request script) — membuktikan blok `hsh` menjadi `null` sementara `residents`/`sec` tetap terisi.
- **Pembersihan**: grup `wo04limited` + admin `wo04-limited@bbh.dev` dihapus setelah sampling (diverifikasi: count = 0). Tidak ada data uji tertinggal.

### §E — Re-review WO multi-DB (D-5) — **STATUS: MENUNGGU KEPUTUSAN OWNER**

> ⚠️ **Temuan yang mengubah kerangka pertanyaan.** Instruksi menyebut topic `1792765876397` (SPEC complete, WO draft). Saat diperiksa, **topic itu sudah tidak ada** di repo — dihapus sebagai bagian dari pembatalan arah. Bukti langsung dari `SPEC.md` topic federasi (`1793013271539`), baris 14:
>
> *"**Arah lama (dibatalkan):** multi-database terpusat — pusat mengakses DB cabang secara langsung via `db_registry`. Seluruh artefaknya sudah dihancurkan (WO-15 destroy: rollback migrasi, potong commit ke `1f5a9b2`, hapus topic `1792765876397`)."*
>
> Arah pengganti = **federasi instance** (pusat & cabang menjalankan aplikasi yang sama, DB lokal masing-masing, saling bicara lewat HTTP + token; tidak ada akses DB lintas jaringan). Jadi "re-review" di sini bukan memilih cara mengakses banyak DB, melainkan **memastikan tidak ada sisa kebutuhan tersembunyi** yang membuat keputusan itu perlu dibuka ulang.

**Fakta teknis terkini (terukur, bukan kutipan lama)**

| Aspek | Kondisi sekarang |
|---|---|
| Engine migrate | `github.com/rolldone/donisgo/console` — target berbasis **`--plugin <id>`**, bukan multi-koneksi |
| Tabel bookkeeping | `migrations(id, target, version, name, applied_at)` UNIQUE(target,version) + `migration_targets(target PK, dirty, updated_at)` |
| Target terdaftar di DB dev | 10: `auth, business_generator, core, endpoint, hsh, ingest, resident, sec, visit, warehouse` — semua `dirty=false` |
| DB per cabang `bbh_<key>` | **nol jejak** di kode/config/migrations repo (`grep "bbh_<key>"` = 0 di luar `.historic`/`.kilo`) |
| `db_registry` / akses DB lintas instance | tidak ada |
| Target refactor lama (`internal/console/migrate.go`, `internal/db/gorm.go`) | **sudah bukan milik repo** — pindah ke library saat WO-02 |

**Tiga opsi (untuk diputuskan owner)**

1. **Kontribusi upstream ke donisgo.** Ajukan dukungan multi-koneksi (mis. `migrate --dsn <name>` + registry koneksi bernama) sebagai PR/issue ke `github.com/rolldone/donisgo`. *Plus*: satu engine, semua app dapat manfaatnya, tidak ada duplikasi. *Minus*: siklus rilis library di luar kendali; butuh diskusi desain (bagaimana `migration_targets` dipisah per DSN); kemungkinan besar **tidak diperlukan** bila federasi instance tetap jadi arah.
2. **Engine migrate milik app di `console/`.** Tulis runner sendiri yang membaca daftar DSN lalu menerapkan migrasi ke tiap DB per cabang. *Plus*: penuh kendali, bisa langsung mendukung `bbh_<key>`. *Minus*: menduplikasi logika yang barusan kita hapus (ironi besar pasca-WO-02: kita menghapus 4.218 baris `internal/`), risiko drift, dan **bertentangan dengan arah federasi** yang sudah diputuskan.
3. **Tunda (rekomendasi Dev).** Tidak mengerjakan apa pun sampai ada kebutuhan bisnis nyata yang tidak bisa dipenuhi federasi instance (mis. laporan konsolidasi lintas cabang). *Plus*: nol biaya, tidak melawan keputusan owner, menjaga hasil WO-02 tetap bersih. *Minus*: bila kebutuhan konsolidasi muncul, keputusan harus dibuka lagi.

**Usulan Dev: opsi 3 (tunda)**, karena:
- Arah multi-DB **sudah dibatalkan owner** dan artefaknya dihancurkan; membangunnya kembali sekarang berarti melawan keputusan itu tanpa kebutuhan baru.
- Federasi instance sudah menyelesaikan masalah yang sama (pusat melihat data cabang) lewat HTTP, tanpa akses DB lintas jaringan — pola yang lebih aman.
- Repo baru saja keluar dari refactor besar; menambah engine migrate sendiri akan mengembalikan duplikasi yang baru saja dihapus.

**Bila owner memilih opsi 1 atau 2**, gap konkret yang harus ditangani (supaya estimasi tidak salah):
- **Gap A — penamaan DSN/target**: `migration_targets.target` saat ini hanya berisi nama plugin (`auth`, `hsh`, …). Multi-DB menuntut dimensi kedua, mis. `target = '<dsn>:<plugin>'` atau tabel target terpisah per DSN — perubahan skema bookkeeping (migrasi data, bukan sekadar kode).
- **Gap B — `--plugin all` vs per-DB**: `collectTargets` library membangun daftar target dari registry plugin dalam satu proses; menerapkannya ke N DSN berarti loop luar per DSN + ordering dependensi intra-DSN tetap sama.
- **Gap C — per-branch DB `bbh_<key>`**: belum ada konvensi di repo. Butuh keputusan penamaan, provisioning (create DB bila belum ada), hak akses, dan bagaimana `endpoints.key`/`INSTANCE_KEY` (yang sekarang dipakai sebagai penanda cabang) dipetakan ke DSN.
- **Gap D — fail-closed & dirty tracking**: `migration_targets.dirty` harus per (DSN,target); kegagalan di satu cabang tidak boleh membuat cabang lain dianggap sukses.
- **Gap E — CLI ergonomi**: `migrate --dsn bbh_cabangA --plugin all up`, `list`, `status`, `down`; plus cara aman menyimpan kredensial DSN (saat ini hanya `.env` tunggal).

**Status: MENUNGGU KEPUTUSAN OWNER. Tidak ada satu baris pun dari rekomendasi ini yang dieksekusi.**

### §E-2 — Draft issue upstream H-3 untuk `donisgo-template` (siap kirim, pengiriman oleh owner)

> **Judul:** `EnsurePluginLast` after `app.New()` registers no routes (catch-all plugin silently missing)
>
> **Body:**
>
> `cmd/server/main.go` di template memakai pola:
>
> ```go
> application, err := app.New(app.Config{Plugins: []plugins.Plugin{}})
> // ...
> application.GetRegistry().EnsurePluginLast(pagesplugin.New())
> ```
>
> Ini **tidak me-register route apa pun dari `pages`**. Bukti empiris (donisgo v0.2.0, commit `e9274de`), dengan plugin uji yang menghitung route terdaftar:
>
> | Varian | `mine.Routes` | `pages.Routes` |
> |---|---|---|
> | A — `pages` sebagai elemen **terakhir** `app.Config.Plugins` | 1 | **1** |
> | B — pre-register ke default registry **sebelum** `app.New` | 1 | **1** |
> | C — `EnsurePluginLast` **setelah** `app.New` (pola template) | 1 | **0** |
>
> **Penyebab:** `app.New` memanggil `attachPlugins()` (`app/bootstrap.go:474-492`) yang menjalankan `RegisterAllServices` lalu `RegisterAllRoutes` **di dalam** `New`. Plugin yang ditambahkan setelahnya tidak pernah melewati fase registrasi route.
>
> **Dampak di aplikasi nyata:** SPA/`/`, `/app/`, `/spa`, dan catch-all `router.NoRoute` hilang tanpa error saat boot — hanya ketahuan saat request 404. Kami menemukan ini saat migrasi ke v0.2.0 (hazard dicatat sebagai H-3).
>
> **Usulan perbaikan:** ubah template ke varian A (plugin catch-all sebagai elemen terakhir `Config.Plugins`) dan/atau buat `EnsurePluginLast` mengembalikan error/panic bila dipanggil setelah `New` (fail-loud), atau dokumentasikan batasan ini di doc comment `EnsurePluginLast`.

## Review Manager

**LULUS — WO-04 COMPLETE. TOPIK MIGRASI TUNTAS.** Dev: `fafafd8` (gofmt terpisah) + `af73b4f` (WO-04), dok historic `3490295`, tanpa push. Verifikasi spot-check Manager (independen, 2026-11-25):

- **AC-1**: `docs/swagger/swagger.json` dihitung sendiri = **141 paths / 209 ops / 248 defs** — identik baseline WO-01 (nol regresi dokumentasi); `rolldone` di definitions = **0** ✅. Regenerasi di container (lingkungan kanonik) ✅ H-6.
- **AC-6**: `gofmt -l` di container = **kosong** (0 file) ✅; commit `style: gofmt` terpisah `fafafd8` ✅.
- **AC-2**: `/healthz` 200 dari host ✅; Dockerfile/compose bersih dari `internal/`.
- **Sisa 50 kemunculan `rolldone` di kode = SAH**: itu import framework ke library yang di-pin (`github.com/rolldone/donisgo/{plugins,db,secrets,...}`) — bukan sisa monorepo. Tidak perlu dibersihkan; berbeda dengan `internal/` yang salah.
- **AC-4**: Bruno smoke 13/36 PASS + RBAC sampling (viewer 200/403, limited group dashboard 12/12) — diterima.
- **AC-7**: data uji dibersihkan (verified count=0) ✅.

**Adili temuan besar (temuan #1): `1792765876397` TIDAK ADA — koreksi terhadap dokumen topic ini.**
Manager verifikasi langsung: `ls .historic/` + `.historic/.database/` = **tidak ada** topic itu; bukti di `SPEC.md` topic federasi `1793013271539:14` — arah multi-DB terpusat **dibatalkan owner** (WO-15 destroy: rollback migrasi, potong commit `1f5a9b2`, hapus topic). Konsekuensi:
1. D-5 dalam topic ini **terbentur realita**: target re-review (WO multi-DB) sudah dihancurkan. Bagian §E kini bukan memilih cara akses banyak DB, melainkan **menutup kebutuhan yang sudah tidak ada**.
2. **Keputusan §E: usulan dev Opsi 3 (tunda) diterima Manager** — federasi instance menyelesaikan kebutuhan via HTTP (plugin `endpoint` + `federation`, WO-01..02 topic itu complete); membangun engine migrate sendiri = mengembalikan duplikasi ±4.218 baris yang baru dihapus WO-02. Gap A–E tetap terdokumentasi di dok WO bila arah berubah kelak.
3. **Dokumen yang perlu disesuaikan** (perbaikan kecil): D-5 di `spec.md` & ringkasan master index diberi catatan penutup — "target re-review sudah di-destroy owner; keputusan akhir = tunda (Opsi 3), arah aktif = federasi instance".

**Adili temuan #2 (jebakan swag)**: 2 jebakan nyata (`analysis_apply.go` blank-var + nama tipe baru; `record_write.go:181` path lama) diterima — pola swag berulang (3x terdokumentasi) dan sudah ditangani benar.

**Adili temuan #3 (gotcha Bruno)**: diterima — catatan untuk keputusan kecil owner (bisa jadi patch kecil tanpa WO per preferensi fix kecil: ubah `{{process.env.BBH_TOKEN}}` ke var biasa di `environments/Local.bru`).

**Catatan `go.mod` godotenv indirect→direct**: diterima (konsekuensi sah test WO-03 meng-import langsung).

Status akhir: **WO-01..04 semua complete → migrasi ke donisgo v0.2.0 pure library SELESAI**. Rekap kode: `e99477d` (inti, 244 file), `c495e2c` (auth instans), `fafafd8` (gofmt), `af73b4f` (verifikasi/docs). Bahan keputusan owner pasca-topik: (a) push 4 commit ke `origin/master`; (b) kirim draft issue H-3 ke `donisgo-template`; (c) konfirmasi penutupan (close) topic ini; (d) patch kecil Bruno env (opsional).

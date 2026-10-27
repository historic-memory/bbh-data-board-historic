---
title: 'WO-07: Paritas swagger (route tanpa anotasi), koreksi angka bukti, dan rebuild bin/console'
description: 'Verifikasi Manager atas WO-06 menemukan 4 sisa: (1) PATCH /api/admin/endpoints/{key} berjalan tetapi tidak ada di swagger karena handler Update hanya dianotasi [put]; (2) GET /api/business/reports berjalan tetapi tidak ada di swagger karena handler WhoAmI hanya dianotasi [woami]; (3) angka route di WO-05 §6a tidak dapat direproduksi (klaim endpoints 15 / sso-enrollments 2, kenyataan tag endpoints 8 / sso 4); (4) DoD rebuild binary belum dipenuhi untuk bin/console (sha256 beda dari build HEAD).'
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 3
---

# WO-07: Paritas swagger, koreksi bukti, rebuild `bin/console`

**SPEC acuan:** `SPEC.md` (D-8, D-19), **WO-05 §3 T-1**, **WO-06 §7.3**
**Prasyarat:** WO-06 (guard & env sudah tertutup — jangan diubah lagi)

## 1. Latar Belakang

WO-06 menutup celah keamanan guard byte-level dan memperbaiki komentar `.env.example`. Namun verifikasi Manager menemukan bahwa **T-1 (swagger) belum tuntas**: ada route yang benar-benar melayani request tetapi tidak muncul di dokumentasi.

Ini kelas cacat yang sama dengan yang WO-05 dimaksudkan untuk menghapus — dokumentasi yang membuat pembaca berikutnya salah paham tentang API yang tersedia.

## 2. Temuan

### S-1 — `PATCH /api/admin/endpoints/{key}` tidak terdokumentasi

Route terdaftar di kode:

```go
// plugins/endpoint/plugin.go
group.PUT("/:key",   middleware.RequirePermission(authSvc, "endpoint.write"), h.Update)
group.PATCH("/:key", middleware.RequirePermission(authSvc, "endpoint.write"), h.Update)
```

Handler-nya hanya punya **satu** anotasi (`plugins/endpoint/handlers/*.go`):

```go
// @Router /api/admin/endpoints/{key} [put]
func (h *Handler) Update(c *gin.Context) {
```

Akibatnya `swagger.yaml` untuk path `/api/admin/endpoints/{key}` hanya memuat `delete`, `get`, `put`.

**Dampak:** klien yang mengandalkan swagger tidak tahu ada PATCH. `PUT` dan `PATCH` berbagi handler yang sama tetapi semantiknya berbeda (partial update), sehingga keduanya perlu didokumentasikan terpisah.

### S-2 — `GET /api/business/reports` tidak terdokumentasi

Route terdaftar di kode:

```go
// plugins/portalsso/plugin.go:61-64
business := api.Group("/business", ssomw.RequirePortalIdentity(svc, resolver))
business.GET("/whoami",  h.WhoAmI)
business.GET("/reports", ssomw.RequireLocalPermission(authSvc, "admin.read"), h.WhoAmI)
```

Handler `WhoAmI` hanya dianotasi `// @Router /api/business/whoami [get]`.

`grep business/reports docs/swagger/swagger.yaml docs/swagger/swagger.json` → **0 hasil**.

**Dampak:** route uji otorisasi lintas instance (`admin.read`) tidak terlihat di dokumentasi. Ini route yang dipakai membuktikan AC-13, jadi justru perlu tercatat.

### S-3 — Angka route di WO-05 §6a tidak dapat direproduksi

Klaim di `WO-05-housekeeping-docs-env.md` §6a:

> route `endpoints` 15, `sso-enrollments` 2, `portal-token` 1, `whoami` 1; `paths: 26`

Hitungan Manager langsung dari `swagger.yaml`:

| Tag | Operasi |
|---|---|
| `admins` | 5 |
| `audit` | 1 |
| `auth-admin` | 4 |
| `auth-member` | 4 |
| `endpoints` | **8** |
| `members` | 5 |
| `rbac` | 8 |
| `sso` | **4** |
| **total** | **39** |

`paths` = **26** ✅ (satu-satunya angka yang cocok).

Angka `15` dan `2` tidak sesuai metode hitung mana pun. Kemungkinan besar diambil dari log `swag` saat generasi, bukan dari isi berkas hasil. **Bukti harus bisa direproduksi dari artefak, bukan dari log proses.**

### S-4 — `bin/console` basi

| Item | Nilai |
|---|---|
| `bin/console` mtime | `2026-10-27 16:12:41` |
| `plugins/portalsso/services/sso.go` mtime | `2026-10-27 16:15:06` |
| `sha256sum bin/console` vs `go build -o /tmp/console-check ./cmd/console` | **berbeda** |

`bin/console` memuat simbol `looksLikeReencodedPair`, tetapi hasil build-nya bukan HEAD — jadi versinya adalah `5152ae6`, belum `ee5b35a`.

**Akar masalah:** `go build ./...` **tidak** menulis ke `bin/`. Rebuild harus eksplisit (`go build -o bin/console ./cmd/console`). Ini sudah pernah tercatat sebagai pelajaran dan terulang.

## 3. Pekerjaan

| # | Pekerjaan | File |
|---|---|---|
| 1 | Tambah anotasi kedua pada handler `Update`: `// @Router /api/admin/endpoints/{key} [patch]` (blok `@Summary`/`@Param`/`@Success` dilengkapi agar berbeda dari PUT) | `plugins/endpoint/handlers/*.go` |
| 2 | Tambah anotasi untuk `business.GET("/reports")` — tidak boleh memakai dokumentasi `WhoAmI` milik `/whoami`; beri ringkasan sendiri (mis. route uji otorisasi `admin.read`) | `plugins/portalsso/handlers/*.go` |
| 3 | Regenerasi swagger (`/go/bin/swag init` sesuai `build.sh:24`) lalu **hitung ulang** angka bukti dari berkas hasil | `docs/swagger/*` |
| 4 | Koreksi baris "T-1 swagger" di WO-05 §6a dengan angka hasil hitung baru + sebutkan metode hitungnya | `WO-05-housekeeping-docs-env.md` |
| 5 | Rebuild **eksplisit**: `go build -o bin/console ./cmd/console` dan `go build -o bin/server ./cmd/server` | `bin/` |

## 4. Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | `swagger.yaml` memuat `patch` untuk `/api/admin/endpoints/{key}` | parse yaml, cetak daftar method |
| 2 | `swagger.yaml` memuat `/api/business/reports` | `grep business/reports` ≥ 1 |
| 3 | Jumlah operasi per tag di WO-05 §6a **sama** dengan hasil parse `swagger.yaml` | jalankan skrip hitung, bandingkan tabel |
| 4 | `paths` tidak berkurang dari 26 dan `is_root` tetap 0 di 3 berkas swagger | `grep -c is_root` = `0 0 0` |
| 5 | `sha256 bin/console` == hasil build ulang HEAD, dan mtime-nya lebih baru dari semua source `.go` | `sha256sum` + `find -newer` |
| 6 | `go build ./... && go vet ./... && go test ./...` lulus; `gofmt -l .` kosong | CI |

## 5. Definition of Done

- [ ] Semua AC lulus dengan **output perintah** yang ditempel di laporan, bukan ringkasan
- [ ] Angka bukti berasal dari **berkas hasil**, bukan dari log proses
- [ ] `bin/console` **dan** `bin/server` di-rebuild eksplisit
- [ ] Commit dokumen diverifikasi dengan `historic git log`, bukan `git log`
- [ ] Tidak ada push ke remote

## 6. Out of Scope

- Mengubah guard `SSO_SHARED_SECRET` (sudah selesai di WO-06 — jangan disentuh)
- Menambah endpoint baru
- Memperbaiki ketidakcocokan `Update` dipakai untuk PUT **dan** PATCH (itu keputusan desain, catat sebagai temuan bila mengganggu)
- Perubahan frontend

## 7. Catatan Metode Uji (dari WO-06 §7.2)

Jangan menguji konfigurasi lewat `docker compose exec -e VAR=...` selama file `.env` ada: `godotenv.Overload()` menimpa environment proses dengan isi `.env`, sehingga hasil uji tidak mencerminkan nilai yang dimaksud. Pakai direktori kerja dengan `.env` sendiri.

## 8. Hasil Verifikasi Manager (independen, 2026-10-27)

Commit yang diperiksa: `c3dc6f4` (4 file: 2 handler + 2 plugin).

### 8.1 Status AC

| # | AC | Verifikasi Manager | Hasil |
|---|---|---|---|
| 1 | `swagger.yaml` memuat `patch` untuk `/api/admin/endpoints/{key}` | parse `docs/swagger/swagger.json` → methods = `delete, get, patch, put`; route di `plugins/endpoint/plugin.go` kini `h.Patch` (bukan `h.Update`) | ✅ |
| 2 | `swagger.yaml` memuat `/api/business/reports` | `'/api/business/reports' in paths` → `True`; route memakai `h.BusinessReports` (bukan `h.WhoAmI`) | ✅ |
| 3 | Angka §6a sama dengan hasil parse | hitungan Manager: `paths` **27**, operasi **41**, `endpoints` **9**, `sso` **5**, `admins` 5, `audit` 1, `auth-admin` 4, `auth-member` 4, `members` 5, `rbac` 8 — **cocok persis** dengan laporan Dev | ✅ |
| 4 | `paths` ≥ 26, `is_root` = 0 di 3 berkas | `paths` 27; `grep -c is_root` → `0 / 0 / 0` | ✅ |
| 5 | `sha256 bin/console` == build ulang HEAD, mtime lebih baru dari source | **GAGAL pada laporan Dev** — lihat §8.2. Setelah rebuild pasca-commit oleh Manager: `CONSOLE_IDENTIK`, `SERVER_IDENTIK`, `find -newer` kosong | ⚠️→✅ |
| 6 | build/vet/test/gofmt | `BUILD_OK`, `VET_OK`, 10 paket `ok`, `GOFMT_CLEAN` | ✅ |

### 8.2 AC-5 gagal pada laporan awal — sebab dan bukti

Dev melaporkan "`sha256 bin/console` == build ulang → `console MATCH`". Saat diperiksa Manager, **tidak dapat direproduksi**:

```
sha bin/console  : 1e7da33ccc7db37d...
build dari HEAD  : 24357b83b259107e...   -> CONSOLE_BEDA, SERVER_BEDA
find plugins internal cmd -name "*.go" -newer bin/console
  -> plugins/endpoint/plugin.go, plugins/endpoint/handlers/handlers.go,
     plugins/portalsso/plugin.go, plugins/portalsso/handlers/handlers.go
```

Akar masalah, dibuktikan dengan `go version -m`:

```
bin/console: build vcs.revision=ee5b35a1e93d...  vcs.modified=true
```

Kedua binari di filesystem dibuild dari working tree **sebelum** commit `c3dc6f4`, sehingga Go menyematkan revisi lama + status dirty. Pembanding yang dipakai Dev (`/tmp/console-check`) dibuat pada kondisi pra-commit yang sama — jadi keduanya cocok satu sama lain dan menghasilkan `MATCH` yang menyesatkan. Begitu commit dibuat, build baru menghasilkan stamp `c3dc6f4` → hash berbeda.

**Pelajaran wajib:** rebuild binari harus dilakukan **setelah** commit, karena Go menyematkan `vcs.revision`/`vcs.modified` ke dalam binari. Membandingkan dua binari yang sama-sama dibuat sebelum commit tidak membuktikan apa pun. `vcs.modified=true` akan selalu muncul selama `bin/` untracked — itu wajar, jadi kriteria yang benar adalah kesamaan isi binari, bukan flag dirty.

**Tindakan:** Manager menjalankan rebuild eksplisit dari `c3dc6f4` (bukan penulisan kode) → stamp menjadi `vcs.revision=c3dc6f4`, dan `cmp` terhadap build ulang HEAD → `CONSOLE_IDENTIK` / `SERVER_IDENTIK`, `find -newer` kosong. AC-5 kini terpenuhi.

### 8.3 Temuan baru → S-5

**`build.sh:24` tidak reproducible terhadap artefak yang ada.**

| Aspek | `swag init` seperti di `build.sh` | `swag init --parseDependency --parseInternal` (= isi `docs/swagger` sekarang) |
|---|---|---|
| `paths` | 27 | 27 |
| total operasi | 41 | 41 |
| path dengan `patch` | 3 | 3 |
| `/api/business/reports` | ada | ada |
| jumlah `definitions` | 29 | 29 |
| **nama definisi** | `handlers.ChangePasswordRequest` | `plugins_auth_handlers.ChangePasswordRequest` |

Struktur API identik, jadi klaim Dev bahwa hasil "deterministik (paths 27, patch 3, reports ada)" **benar**. Yang tidak sama adalah **penamaan definisi**: tanpa `--parseDependency`, `swag` menamai tipe berdasarkan nama paket terakhir saja, sehingga beberapa paket `handlers` saling bertabrakan dan diberi prefix berbeda.

Konsekuensi: siapa pun menjalankan `build.sh` akan **menimpa** `docs/swagger` dengan penamaan yang berbeda dari artefak yang diverifikasi di sini — SDK/klien yang di-generate dari swagger akan melihat nama tipe berubah. `docs/` memang gitignored, jadi ini tidak merusak commit, tetapi membuat dokumentasi tidak dapat direproduksi.

**Perbaikan yang diminta:** selaraskan perintah di `build.sh:29` dengan flag yang menghasilkan artefak final (`swag init -g cmd/server/main.go -o ./docs/swagger --parseDependency --parseInternal`), atau buang flag tersebut dan regenerasi artefak final dengan perintah `build.sh`. Pilih satu sebagai sumber kebenaran.

### 8.4 Verifikasi S-5 (build.sh) — Manager, 2026-10-27

Commit `b6c425c` (perintah swag) + `790acf6` (yaml.v3 direct).

| Aspek | Hasil verifikasi |
|---|---|
| Perintah di `build.sh` memuat kedua flag | ✅ `swag init -g cmd/server/main.go -o ./docs/swagger --parseDependency --parseInternal` |
| Alasan tabrakan nyata | ✅ `ErrorResponse` didefinisikan di **3** paket (`auth`, `endpoint`, `portalsso`). Plain → nama campur (`handlers.MessageResponse` + `plugins_auth_handlers.ErrorResponse`); final → konsisten (`plugins_auth_handlers.*`) |
| Struktur API identik plain vs final | ✅ `paths` 27, operasi 41, `definitions` 29 di keduanya |
| Byte-identity saat regenerate | ✅ di container: `docs.go`, `swagger.json`, `swagger.yaml` **IDENTIK** (`docs.go` hanya beda nama `package` bila `-o` memakai nama direktori lain) |
| Idempotensi | ✅ `go.mod` tidak berubah setelah `bash build.sh`; `git status` bersih kecuali `bin/` |

**S-5 selesai dan terbukti.**

### 8.5 Temuan baru → S-6: `build.sh` tidak efektif di host

Saat menjalankan `bash build.sh` di **host**, generasi swagger **gagal diam-diam**:

```
Found swag CLI - regenerating docs...
failed to parse file /snap/go/11382/src/math/rand/v2/rand.go, error: method must have no type parameters
swag init returned non-zero status (continuing)
```

| Uji | Hasil |
|---|---|
| `swag init ... --parseDependency --parseInternal` di host | `exit=1` (GOROOT host = `/snap/go/11382`, swag gagal parse stdlib generik) |
| `swag init` **tanpa** flag di host | berhasil, tetapi penamaan definisi campur (yang justru ingin dihindari) |
| `swag init ... --parseDependency --parseInternal` di container | `exit=0`, byte-identik |
| Dampak ke `docs/swagger` setelah `bash build.sh` host | **sha256 tidak berubah** — no-op, karena `|| echo "... continuing"` menelan kegagalan |

Artinya: dengan `build.sh` versi sekarang, di host **tidak ada** yang memberi tahu bahwa swagger tidak di-regenerate. Komentar non-fatal memang disengaja, tetapi konsekuensinya dokumentasi bisa tertinggal tanpa peringatan.

**Caveat tambahan (reproducibility binari antar-lingkungan):** `go build` di host (GOROOT `/snap/go/11382`) dan di container (`/usr/local/go`) menghasilkan binari **berbeda** meski `go version` sama:

```
container: 667448a34a6e7746  (stamp vcs.revision=790acf6)
host     : 6bd87df46fe58933  (stamp vcs.revision=790acf6)
```

Jadi `bin/` hanya reproducible bila dibangun di **satu** lingkungan kanonik (container). `bin/` untracked, jadi tidak memengaruhi commit — tetapi klaim "sha256 cocok" hanya bermakna bila kedua sisi dibangun di lingkungan yang sama **dan** setelah commit.

### 8.6 Status

WO-07 **`complete`** — S-1..S-5 terverifikasi; AC-6 (build/vet/test/gofmt) lulus. Temuan S-6 (host no-op + binari antar-lingkungan) dipindah ke `WO-08`.

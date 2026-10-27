---
title: 'WO-07: Paritas swagger (route tanpa anotasi), koreksi angka bukti, dan rebuild bin/console'
description: 'Verifikasi Manager atas WO-06 menemukan 4 sisa: (1) PATCH /api/admin/endpoints/{key} berjalan tetapi tidak ada di swagger karena handler Update hanya dianotasi [put]; (2) GET /api/business/reports berjalan tetapi tidak ada di swagger karena handler WhoAmI hanya dianotasi [woami]; (3) angka route di WO-05 §6a tidak dapat direproduksi (klaim endpoints 15 / sso-enrollments 2, kenyataan tag endpoints 8 / sso 4); (4) DoD rebuild binary belum dipenuhi untuk bin/console (sha256 beda dari build HEAD).'
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
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

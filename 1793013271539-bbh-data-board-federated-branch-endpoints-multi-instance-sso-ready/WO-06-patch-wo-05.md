---
title: 'WO-06: Penambal WO-05 — normalisasi base64 guard, komentar env, dan arsip bukti AC-13'
description: 'Verifikasi Manager atas WO-05 menemukan 3 cacat: (1) normalizeKey hanya strip prefix base64: tanpa decode, padahal MasterKey() men-decode — byte kunci sama bisa lolos guard (AC 5c belum terpenuhi); (2) komentar .env.example salah menyebut "server menolak start" saat kosong, padahal tetap start; (3) bukti AC-13 dan hasil T-1/T-2/T-4 tidak tersimpan di dokumen WO-05 (status complete tapi §3 masih rencana, §5 tanpa blok bukti).'
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 3
---

# WO-06: Penambal WO-05

**SPEC acuan:** `SPEC.md` (D-17), **WO-05 §3 & §5**
**Prasyarat:** WO-05 (selesai) — ini menambal cacat yang ditemukan saat verifikasi.

## 1. Latar Belakang

WO-05 sudah dinyatakan `complete`. Verifikasi Manager atas laporan menemukan **klaim benar sebagian**:

| Klaim WO-05 | Hasil verifikasi |
|---|---|
| Commit `3fc0b5e` ada, 11 file | ✅ benar |
| `go build`/`vet`/`test`/`gofmt` lulus, 10 paket | ✅ benar |
| T-1 swagger: `is_root`=0, route baru ada | ✅ benar (`endpoints` 15, `sso-enrollments` 2, `portal-token` 1, `whoami` 1) |
| T-2 `.env.example` 111 baris, 5 var baru | ✅ benar |
| T-3 AC-13 terbukti di UI | ✅ **perilaku** benar (laporan detail) |
| T-4 fail-closed, 6/6 test | ⚠️ **sebagian** — lihat Cacat 1 |
| DB 3 buah di-drop | ✅ benar (sisa `bbh_data_board`) |
| Tidak ada push | ✅ benar (origin tetap `22ce050`) |

## 2. Cacat yang Ditemukan

### Cacat 1 — 🔴 **Guard bisa diloloskan** (AC 5c belum terpenuhi)

WO-05 §3 meminta normalisasi **tiga langkah**: buang prefix → **base64-decode** → bandingkan byte.

Implementasi `normalizeKey` (`plugins/portalsso/services/sso.go:224`):

```go
func normalizeKey(v string) string {
    return strings.TrimPrefix(strings.TrimSpace(v), "base64:")
}
```

**Hanya langkah 1.** Tidak ada langkah 2 (decode).

Padahal `MasterKey()` (`internal/secrets/secrets.go:42-45`) **men-decode** `APP_KEY`:
```go
if strings.HasPrefix(keyB64, "base64:") { keyB64 = strings.TrimPrefix(keyB64, "base64:") }
key, err := base64.StdEncoding.DecodeString(keyB64)
```

**Bukti celah (dijalankan):**

```
APP_KEY=base64:QUJD  -> MasterKey decode = ABC      (3 byte)
SSO_SHARED_SECRET=ABC -> normalizeKey = ABC  (string)
normalizeKey("base64:QUJD") = "QUJD"
"QUJD" != "ABC"  ->  GUARD LOLOS
tapi byte kunci: ABC == ABC  ->  RISIKO NYATA
```

**Akibat:** admin bisa memasukkan `SSO_SHARED_SECRET` yang **byte-nya identik** dengan `APP_KEY` (hanya berbeda penulisan) dan guard tidak menangkapnya — persis keadaan yang WO-05 ingin cegah.

### Cacat 2 — Komentar `.env.example` bertentangan dengan perilaku

`.env.example:84` menyatakan:

```
#   1. Diisi bila SSO dipakai, kalau tidak server menolak start.
```

Padahal kode (`ValidateSecretConfig`):
```go
shared := normalizeKey(os.Getenv("SSO_SHARED_SECRET"))
if shared == "" { return nil }     // ← KOSONG = AMAN, server TETAP start
```

Komentar menyatakan **kebalikan** dari perilaku. Cabang tanpa SSO seharusnya tetap jalan (itu keputusan eksplisit WO-05 §0/§3).

### Cacat 3 — Bukti tidak terarsip

Dokumen `WO-05-housekeeping-docs-env.md` ditandai `status: complete` dan AC-13 `[x]`, tetapi:

| Seharusnya | Kenyataan |
|---|---|
| §3 berisi **hasil** (kode + bukti) | §3 masih berbunyi *"Desain yang wajib diimplementasikan"* — masih rencana |
| §5 (atau lampiran) memuat **output AC-13** | Tidak ada blok bukti (`afterLogin`/`afterExitPortal` tidak ditemukan) |
| Bukti T-1/T-2/T-4 terekam | Hanya ada di laporan chat, tidak di dokumen |

**Dampak:** dokumen tidak bisa dijadikan sumber tunggal. Pembaca berikutnya harus menebak atau bertanya.

## 3. Pekerjaan

| # | Pekerjaan | File |
|---|---|---|
| 1 | `normalizeKey` juga **base64-decode** (fallback ke string mentah bila decode gagal); bandingkan **byte** | `plugins/portalsso/services/sso.go` |
| 2 | Unit test baru: `SSO_SHARED_SECRET=ABC` vs `APP_KEY=base64:QUJD` → **ditolak**; dan variasi padding (`AAAA` vs `AAAA==`) | `plugins/portalsso/services/secret_config_test.go` |
| 3 | Perbaiki komentar `.env.example`: **kosong = SSO nonaktif, server TETAP start**; identik `APP_KEY` = tolak start | `.env.example` |
| 4 | Arsipkan bukti ke dokumen: §3 diubah dari rencana → hasil; tambah blok bukti AC-13 + T-1/T-2/T-4 | `WO-05-housekeeping-docs-env.md` |

## 4. Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | `SSO_SHARED_SECRET=ABC` + `APP_KEY=base64:QUJD` → **start ditolak** | unit test + jalankan |
| 2 | `SSO_SHARED_SECRET=AAAA` + `APP_KEY=base64:AAAA==` (padding beda) → **ditolak** | unit test |
| 3 | `SSO_SHARED_SECRET` kosong → server **tetap start**, SSO nonaktif | jalankan |
| 4 | Nilai kunci **tidak pernah** tercetak di log/error | grep log = 0 |
| 5 | Komentar `.env.example` sesuai perilaku nyata | baca + jalankan dua skenario |
| 6 | Dokumen WO-05 §3 = hasil, dan memuat bukti nyata T-1/T-2/T-3/T-4 | baca dokumen |
| 7 | `go build ./... && go vet ./... && go test ./...` lulus | CI |

## 5. Definition of Done

- [ ] Semua AC lulus dengan bukti output nyata
- [ ] Test lama (6 kasus) **tetap lulus** — tidak ada regresi
- [ ] Bukti disimpan **di dokumen**, bukan hanya di chat
- [ ] Rebuild binary eksplisit
- [ ] Verifikasi commit dokumen dengan `historic git log`
- [ ] Tidak ada push ke remote

## 6. Out of Scope

- Mengubah desain guard (hanya melengkapi normalisasi)
- RS256/JWKS
- Perubahan frontend

## 7. Hasil Verifikasi Manager (independen, 2026-10-27)

Semua angka di bawah berasal dari perintah yang dijalankan Manager sendiri, bukan dari laporan Dev Agent.

### 7.1 Status AC

| # | AC | Verifikasi Manager | Hasil |
|---|---|---|---|
| 1 | `APP_KEY=base64:QUJD` + `SSO_SHARED_SECRET=ABC` → start ditolak | `.env` uji di `/tmp/seccheck` (bukan `.env` repo), `timeout 15 ./server` → `rc=1`, `failed to run server: portalsso: SSO_SHARED_SECRET tidak boleh sama dengan APP_KEY; buat nilai terpisah dengan: openssl rand -base64 32` | ✅ |
| 2 | `APP_KEY=base64:AAAA==` + `SSO_SHARED_SECRET=AAAA` (padding beda) → ditolak | `rc=1`, pesan identik | ✅ |
| 3 | Tanpa `SSO_SHARED_SECRET` → server tetap start, SSO nonaktif | `Listening and serving HTTP on :8080`; `curl /healthz` → **200** | ✅ |
| 4 | Kunci tidak tercetak di log/error | `grep -cE "QUJD\|AAAA" out.log out2.log` → `0`, `0` | ✅ |
| 5 | Komentar `.env.example` sesuai perilaku | `.env.example` baris ~83-89: "Bila KOSONG, SSO nonaktif tetapi server TETAP start" | ✅ |
| 6 | WO-05 §3 = hasil + bukti T-1..T-4 terarsip | §3 berganti menjadi "Desain yang diimplementasikan (hasil)"; §6a memuat T-1, T-2, T-4, dan blok AC-13 (`afterLogin` / `afterExitPortal` / `whoami`) | ✅ |
| 7 | `go build`/`vet`/`test`/`gofmt` lulus | `BUILD_OK`, `VET_OK`, 10 paket `ok`, `GOFMT_CLEAN` | ✅ |

Unit test: `go test ./plugins/portalsso/... -run 'TestValidateSecretConfig|TestSecretReuseErrorDoesNotLeakKey' -v` → **13 sub-tes RUN, PASS**; `TestSecretReuseErrorDoesNotLeakKey` PASS.

**Ketiga cacat WO-05 terbukti tertutup:** Cacat 1 lewat AC-1 & AC-2, Cacat 2 lewat AC-5, Cacat 3 lewat AC-6.

### 7.2 Catatan metode — jebakan saat verifikasi

Uji pertama Manager **tidak memvalidasi apa pun**: `docker compose exec -e APP_KEY=...` tidak berpengaruh karena `godotenv.Overload()` (dipakai `internal/console`) **menimpa** environment proses dengan isi `.env`. Akibatnya server start normal dan sempat disalahartikan sebagai "guard lolos". Verifikasi ulang dilakukan dengan `.env` terpisah di `/tmp/seccheck`.

→ Aturan untuk WO berikutnya: **jangan** menguji konfigurasi lewat `-e` selama file `.env` ada; buat direktori kerja dengan `.env` sendiri.

### 7.3 Sisa yang belum selesai → `WO-07`

| # | Sisa | Bukti |
|---|---|---|
| S-1 | 🔴 `PATCH /api/admin/endpoints/{key}` terdaftar di kode tetapi **tidak ada di swagger** | `plugins/endpoint/plugin.go` mendaftarkan `PUT /:key` **dan** `PATCH /:key`; handler `Update` hanya punya satu anotasi `// @Router /api/admin/endpoints/{key} [put]`. `swagger.yaml` hanya memuat `delete/get/put` untuk path itu |
| S-2 | 🔴 `GET /api/business/reports` terdaftar di kode tetapi **tidak ada di swagger** | `plugins/portalsso/plugin.go:64`; handler `WhoAmI` hanya dianotasi `// @Router /api/business/whoami [get]`; `grep business/reports` di `swagger.yaml` / `swagger.json` = 0 |
| S-3 | Angka di WO-05 §6a **tidak dapat direproduksi** | Klaim "`endpoints` 15, `sso-enrollments` 2, `portal-token` 1, `whoami` 1". Hitungan Manager per tag: `endpoints` **8**, `sso` **4**, `audit` **1**, total operasi **39**, `paths`: **26** (angka `paths: 26` cocok) |
| S-4 | DoD "rebuild binary eksplisit" belum dipenuhi untuk `console` | `bin/console` mtime `16:12:41`, `sso.go` mtime `16:15:06`; `sha256` `bin/console` **berbeda** dari hasil `go build -o /tmp/console-check ./cmd/console` pada HEAD |

S-1 dan S-2 adalah route yang **benar-benar berjalan tetapi tidak terdokumentasi** — persis kelas cacat yang WO-05 dimaksudkan untuk menghapus. Karena itu WO-06 **belum** dapat dinyatakan `complete`.

**Status WO-06: `progress` (revision 2).** Sisa pekerjaan dipindahkan ke `WO-07`.

### 7.4 Penutupan (revision 3)

Keempat sisa (S-1..S-4) ditutup lewat `WO-07` (commit `c3dc6f4` + rebuild binari oleh Manager). Verifikasi Manager atas WO-07 mencatat: S-1, S-2, S-3 ✅; S-4 awalnya gagal (binari di-stamp `vcs.revision=ee5b35a` karena dibuild sebelum commit) lalu terpenuhi setelah rebuild pasca-commit. WO-06 sendiri tidak punya pekerjaan terbuka lagi → `complete`.

---
title: 'WO-06: Penambal WO-05 — normalisasi base64 guard, komentar env, dan arsip bukti AC-13'
description: 'Verifikasi Manager atas WO-05 menemukan 3 cacat: (1) normalizeKey hanya strip prefix base64: tanpa decode, padahal MasterKey() men-decode — byte kunci sama bisa lolos guard (AC 5c belum terpenuhi); (2) komentar .env.example salah menyebut "server menolak start" saat kosong, padahal tetap start; (3) bukti AC-13 dan hasil T-1/T-2/T-4 tidak tersimpan di dokumen WO-05 (status complete tapi §3 masih rencana, §5 tanpa blok bukti).'
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
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

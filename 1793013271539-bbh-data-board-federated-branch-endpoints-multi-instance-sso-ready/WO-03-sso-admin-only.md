---
title: 'WO-03: SSO Lintas Instance (admin-only) — portal token + keanggotaan lokal'
description: 'Pusat menerbitkan identity token (portal token) ber-aud cabang; cabang memverifikasi LOKAL (tanpa callback) dan hanya menerima admin yang SUDAH terdaftar di admins cabang (403 bila belum). Enrollment admin + double-token session.'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
revision: 1
---

# WO-03: SSO Lintas Instance (admin-only)

**SPEC acuan:** `SPEC.md` (D-8, D-14, D-15; AC-11..AC-15, AC-21, AC-22)
**Prasyarat:** WO-01 (registry) + WO-02 (routing). **Jangan mulai sebelum keduanya stabil.**
**Fase:** 3
**Scope:** **ADMIN saja** (D-8). `members` di luar ruang lingkup.

## 1. Tujuan

Admin login **sekali** di pusat, lalu dapat memakai portal cabang **tanpa login ulang** — dengan syarat ia **sudah terdaftar** di `admins` cabang tersebut.

## 2. Alur

```
1. Admin login di PUSAT                → main token (sudah ada)
2. Admin pilih portal "cabang-jkt"      → pusat minta portal token
3. PUSAT menerbitkan PORTAL TOKEN       → aud=cabang-jkt, iss=pusat, exp pendek
4. Token dibawa ke CABANG (WO-02)
5. CABANG verifikasi LOKAL:
     a. tanda tangan sah?          → tidak → 401
     b. aud cocok dengan key dirinya? → tidak → 403 (token sah, salah tujuan)
     c. exp belum lewat (toleransi skew) → lewat → 401
     d. admin terdaftar di admins cabang? → TIDAK → 403 jelas
6. Lolos semua → peran lokal (level) dipakai; sesi punya main token + portal token
```

**Tidak ada callback ke pusat pada langkah 5.** Cabang harus bisa melayani admin yang sudah terdaftar **walau pusat sedang mati** (AC-13).

## 3. Klaim Portal Token

Memakai jalur yang **sudah ada** (`internal/auth`, HS256) dengan tambahan `aud`/`iss`.

```json
{
  "iss": "bbh-pusat",
  "aud": ["cabang-jkt"],
  "sub": "<admin_id di PUSAT>",
  "adm": "admin@example.com",
  "lvl": "admin",
  "typ": "portal",
  "iat": 1730000000,
  "exp": 1730000900
}
```

| Klaim | Fungsi |
|---|---|
| `iss` | Instance penerbit — cabang memverifikasi ini penerbit yang dikenali |
| `aud` | **Guard tujuan** — token cabang lain ditolak (AC-14) |
| `adm` | **Kunci pencocokan enrollment** di `admins` cabang (email = identitas stabil) |
| `lvl` | Peran yang diminta; cabang **boleh menurunkan**, **tidak boleh menaikkan** |
| `typ` | Membedakan `portal` dari `access`/`refresh` yang sudah ada |
| `exp` | **Pendek** (usul: 15 menit) |

> **Keputusan yang perlu dikonfirmasi:** `adm` = **email**. Alternatifnya pasangan `iss` + `sub` dari pusat (tanpa perlu email cocok). Email lebih mudah dibaca admin; `iss+sub` lebih tahan kalau email berubah. **Default usulan: email**, dengan `iss+sub` disimpan sebagai jejak di kolom tambahan bila diperlukan.

## 4. Verifikasi Lokal — Pilihan Kunci

| Opsi | Cara | Kelebihan | Kekurangan |
|---|---|---|---|
| **A. HS256 shared secret** | Satu secret di pusat + cabang (env) | Paling sederhana; `internal/auth` sudah HS256 | Secret bocor di **satu** cabang → bisa memalsukan token untuk **semua** |
| **B. RS256/EdDSA** | Pusat pegang private key; cabang pegang public key (JWKS) | Bocor di cabang tidak bisa memalsukan; rotasi lebih rapi | Lebih banyak kode; perlu distribusi kunci |

**Rekomendasi: mulai A**, tetapi **bungkus di balik interface verifier** agar pindah ke B tidak membongkar pemanggil:

```go
type IdentityVerifier interface {
    Verify(token string, expectAud string) (*Identity, error)
}
```

**Nama "verifier" harus jelas** — jangan menamainya `RotateDBMSSecret`-style ambigu. Fungsi ini **hanya memverifikasi**, tidak menerbitkan.

## 5. Guard Keanggotaan Lokal (inti WO ini)

Setelah token sah, cabang **wajib** mencari admin:

```sql
SELECT id, level, status FROM admins WHERE email = $1 LIMIT 1;
```

| Hasil | Aksi |
|---|---|
| Ditemukan & aktif | ✅ lanjut; pakai `level` **lokal** (boleh ≤ `lvl` di token) |
| Ditemukan tapi nonaktif | ❌ **403** — "akun dinonaktifkan di cabang ini" |
| Tidak ditemukan | ❌ **403** — "admin belum terdaftar di cabang ini" |

**Kode 403, bukan 401.** 401 = "siapa kamu?" (token tidak sah). 403 = "kami tahu siapa kamu, tapi kamu tidak berhak di sini". Membedakan keduanya membuat SPA bisa memberi pesan yang benar (AC-12).

## 6. Enrollment Admin di Cabang

> **KEPUTUSAN BELUM DIKONFIRMASI USER — wajib diputuskan sebelum implementasi.**

| Opsi | Cara | Kelebihan | Kekurangan |
|---|---|---|---|
| **A. Manual oleh admin cabang** | Admin cabang menambah baris di `admins` cabang (email + level) | Cabang berdaulat penuh; tidak ada jalur data baru | Repot bila admin banyak × cabang banyak |
| **B. Auto-provision saat login pertama** | Cabang membuat baris `pending`, admin cabang menyetujui | Praktis, tetap ada kendali | Perlu UI approval + tabel/kolom status |
| **C. Push dari pusat** | Pusat mendorong daftar admin ke cabang via API | Terpusat, audit rapi | Butuh endpoint baru + otorisasi; menambah jalur data |
| **D. Import berkala** | Admin cabang impor daftar dari pusat | Sederhana | Tidak real-time; proses rutin |

**Rekomendasi: A dulu** (paling kecil risiko, mempertahankan prinsip "cabang berdaulat atas user-nya"). Bisa dinaikkan ke B/C nanti **tanpa mengubah** verifikasi token — enrollment hanyalah cara mengisi tabel, bukan bagian dari verifikasi.

**Yang tidak boleh terjadi:** auto-approve tanpa kendali. Kalau opsi B dipilih, user **tetap ditolak** sampai admin cabang menyetujui — kalau tidak, siapa pun yang punya token sah pusat bisa masuk cabang mana pun.

## 7. Sesi Dua Token (D-14/D-15)

| Kondisi | `main token` | `portal token` |
|---|---|---|
| Login pusat | ✅ ada | — |
| Masuk portal cabang | ✅ tetap | ✅ ditambahkan |
| Keluar portal | ✅ tetap | ❌ **dibuang** |
| Logout total | ❌ dibuang | ❌ dibuang |

Route memakai token sesuai **pemiliknya** (WO-02 bagian 3). Token **tidak boleh** saling menggantikan (AC-21).

## 8. Kasus Tepi yang Wajib Ditangani

| Kasus | Perilaku |
|---|---|
| Jam cabang vs pusat berbeda | Toleransi skew (usul ±2 menit); di luar itu → 401 dengan pesan jelas |
| `aud` bukan key cabang ini | **403** — "token ditujukan untuk cabang lain" |
| `portal token` kedaluwarsa | **401**; SPA minta token baru ke pusat **tanpa** memaksa login ulang (bila `main token` masih sah) |
| Pusat mati setelah masuk portal | Cabang **tetap melayani** sampai `portal token` kedaluwarsa (AC-13) |
| Admin pusat dinonaktifkan **setelah** token terbit | Tidak bisa dicabut sebelum `exp` → **alasan `exp` pendek**. Untuk pencabutan instan, butuh daftar cabut (di luar scope) |
| `members` mencoba | Di luar scope — ditolak/tegas tidak ditangani fase ini |

## 9. Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | Admin terdaftar di cabang masuk portal **tanpa login ulang** (AC-11) | test 2 instance |
| 2 | Admin **tidak terdaftar** ditolak **403 + pesan jelas**, meski token sah (AC-12) | test |
| 3 | **Pusat dimatikan** → admin terdaftar tetap bisa memakai cabang selama token sah (AC-13) | test matikan pusat |
| 4 | Token ber-`aud` cabang lain **ditolak** (AC-14) | test |
| 5 | Password admin **tidak pernah** dikirim/disimpan di cabang (AC-15) | grep + query |
| 6 | `main token` tidak pernah ke cabang; `portal token` tidak pernah ke pusat (AC-21) | grep/trace |
| 7 | Keluar portal membuang `portal token`, `main token` utuh (AC-22) | test |
| 8 | Level lokal ≤ `lvl` token; cabang tidak bisa menaikkan peran | test |
| 9 | Token palsu/tanda tangan salah → 401 | test |
| 10 | `go build ./... && go vet ./... && go test ./...` lulus | CI |

## 10. Definition of Done

- [ ] Semua AC lulus dengan bukti output nyata
- [ ] **AC-3 dibuktikan dengan benar-benar mematikan pusat** — bukan sekadar alasan "tidak ada callback di kode"
- [ ] Interface verifier terpisah jelas (verifikasi ≠ penerbitan)
- [ ] Tidak ada token/password di log (grep)
- [ ] Rebuild binary eksplisit
- [ ] Bagian yang belum terbukti **dinyatakan belum terbukti**
- [ ] Tidak ada push ke remote

## 11. Out of Scope

- **User `members`** (ditunda)
- Pencabutan token instan / blocklist
- RS256/JWKS (opsi B — bisa jadi WO lanjutan)
- Sinkronisasi data bisnis antar cabang
- UI approval enrollment (bila opsi B/C dipilih)

## 12. Ketergantungan Terbuka (harus diputuskan user)

1. **Mekanisme enrollment** (bagian 6): A / B / C / D — **rekomendasi A**.
2. **Identitas pencocokan** (bagian 3): email (rekomendasi) atau `iss`+`sub`.
3. **Kunci verifikasi** (bagian 4): HS256 (rekomendasi, mulai) atau RS256.

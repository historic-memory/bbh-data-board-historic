---
title: 'WO-03: SSO Lintas Instance (admin-only) — portal token + keanggotaan lokal'
description: 'Pusat menerbitkan identity token (portal token) ber-aud cabang; cabang memverifikasi LOKAL dengan HS256 (D-17) dan hanya menerima admin yang SUDAH terdaftar di admins cabang (403 bila belum/chuma disetujui). Pencocokan via iss+sub UUIDv7 (D-16). Enrollment hybrid A+B (D-18). Double-token session.'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
revision: 2
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

## 3. Klaim Portal Token (D-16)

Memakai jalur yang **sudah ada** (`internal/auth`, HS256) dengan klaim berikut:

```json
{
  "iss": "bbh-pusat",
  "aud": ["cabang-jkt"],
  "sub": "<uuid-v7-admin-id-di-pusat>",
  "atr": "admin@example.com",        // atribut tampilan saja (email tidak jadi kunci)
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
| `sub` | **Kunci pencocokan enrollment**: id admin di DB pusat berupa **UUIDv7** (§5) |
| `atr` | Email/atribut tampilan untuk UI saja — **bukan** kunci pencocokan |
| `lvl` | Peran yang diminta; SSO selalu `admin`; cabang boleh menolaknya jika belum disetujui |
| `typ` | Membedakan `portal` dari `access`/`refresh` yang sudah ada |
| `exp` | **Pendek** (usul: 15 menit) |

> ⚠️ **Tidak lagi menggunakan email sebagai kunci pencocokan.** Admin mungkin ganti domain email — itu tidak boleh memutus akses. Pencocokan memakai pasangan **`iss` + `sub`** yang stabil.

## 4. Verifikasi Lokal (D-17)

**Keputusan: HS256 shared secret** (fase awal), dibungkus di balik **interface verifier** agar upgrade ke RS256/JWKS nanti tidak membongkar pemanggil:

```go
type IdentityVerifier interface {
    Verify(token string, expectAud string) (*Identity, error)
}
// Implementasi: verifyHS256() saat ini, bisa diganti verifyRS256() nanti
```

Fungsi ini **hanya memverifikasi** tanda tangan, ekspirasi, dan `aud`. Tidak pernah menerbitkan atau menyentuh password.

## 5. Guard Keanggotaan Lokal (inti WO ini)

Setelah token sah, cabang **wajib** mencocokkan admin lewat **`iss` + `sub`** (D-16):

```sql
SELECT a.id, a.is_active, a.status, a.origin
FROM admins a
WHERE a.sso_issuer = <token.iss> AND a.sso_subject = <token.sub>
LIMIT 1;
```

| Hasil | Aksi |
|---|---|
| Ditemukan, status `active` | ✅ Lanjut; peran selalu `admin` (SSO **tidak pernah** memberi `is_root`) |
| Ditemukan, status `pending` | ❌ **403** — "autentikasi berhasil, tetapi pendaftaran Anda masih menunggu persetujuan admin cabang" |
| Ditemukan tapi `is_active = false` | ❌ **403** — "akun dinonaktifkan di cabang ini" |
| Tidak ditemukan (pertama kali) | 🔨 **Auto-provision baris `pending`** (D-18 opsi B) → lalu **tolak** sampai disetujui |

**Kode 403, bukan 401.** 401 = "siapa kamu?" (token tidak sah). 403 = "kami tahu siapa kamu, tetapi kamu belum/berhak masuk." Ini memungkinkan SPA menampilkan pesan yang tepat (AC-12).

> 📌 **SKEMA `admins` — PERTIMBANGAN KRITIS**
>
> Tabel `admins` saat ini **tidak punya** kolom `level`, `status`, `sso_issuer`, `sso_subject`, atau `origin`, dan `password_hash` bersifat **NOT NULL**.
>
> Enrollment D-18 (A+B) memerlukan **migrasi tambahan di setiap cabang**:
>
> ```sql
> ALTER TABLE admins ADD COLUMN IF NOT EXISTS status TEXT NOT NULL DEFAULT 'active'; -- active | pending
> ALTER TABLE admins ADD COLUMN IF NOT EXISTS origin TEXT NOT NULL DEFAULT 'local';  -- local | sso
> ALTER TABLE admins ADD COLUMN IF NOT EXISTS sso_issuer TEXT;                      -- instance penerbit
> ALTER TABLE admins ADD COLUMN IF NOT EXISTS sso_subject TEXT;                     -- sub (UUIDv7)
> ALTER TABLE admins ALTER COLUMN password_hash DROP NOT NULL;                      -- admin SSO-murni tidak punya password
> CREATE UNIQUE INDEX IF NOT EXISTS uq_admins_sso_identity ON admins (sso_issuer, sso_subject)
>     WHERE sso_issuer IS NOT NULL;
> ```
>
> Tanpa migrasi ini, pencocokan `iss`+`sub` tidak dapat dilakukan secara akurat.

## 6. Enrollment Admin di Cabang (D-18)

> **KEPUTUSAN KUNCI: Hybrid A + B — kedua mekanisme saling melengkapi.**

| Mekanisme | Cara Kerja | Kelebihan | Kekurangan |
|---|---|---|---|
| **A. Manual oleh admin cabang** | Admin cabang sendiri menambah baris `admins` (dengan `sso_issuer`+`sso_subject`) untuk admin pusat | Cabang berdaulat penuh; tanpa jalur data lintas jaringan | Repot bila admin banyak × cabang banyak |
| **B. Auto-provision saat login pertama** | Admin SSO pertama kali → cabang cek `iss+sub` → tidak ketemu → buat baris `admins` `status='pending'` + `origin='sso'`; admin tetap **ditolak** sampai disetujui | Praktis; tetap terkendali approval | Perlu kolom status di `admins` + cara approve (CLI/API) |

**Alur A+B:**
1. Admin login di pusat → main token.
2. Pilih portal cabang → minta portal token.
3. Buka halaman business → cabang verifikasi token **LOKAL** → cek `iss+sub`:
   - Belum terdaftar → **buat baris `pending`** lalu **tolak**.
   - Terdaftar + `pending` → tolak (menunggu approval).
   - Terdaftar + `active` → lanjut.
4. Admin cabang menyetujui baris tersebut (CLI/API lokal).
5. Akses normal diberikan.

> ✅ **Auto-approve TIDAK DIIZINKAN.** Tanpa persetujuan, sistem otomatis menolak — meski token sah.

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
| Pusat mati setelah masuk portal | Cabang **tetap melayani** sampai `portal token` kedaluwarsa (AC-13) — verifikasi HS256 lokal, tanpa callback |
| Admin SSO pertama kali belum terdaftar | Dibuat baris `pending` → **ditolak**; butuh approval admin cabang (AC-23) |
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
| 6 | **Auto-provision (D-18):** SSO pertama kali membuat baris `pending`, ditolak hingga disetujui (AC-23) | test |
| 7 | **Pencocokan `iss`+`sub` (D-16):** mengganti email tidak memutus akses (AC-24) | test |
| 8 | SSO tidak pernah memberi `is_root` (D-19/AC-25) | grep + query |
| 9 | **Verifikasi HS256 (D-17):** token palsu / tanda tangan rusak → 401 (AC-26) | test |
| 10 | `main token` tidak pernah ke cabang; `portal token` tidak pernah ke pusat (AC-21) | grep/trace |
| 11 | Keluar portal membuang `portal token`, `main token` utuh (AC-22) | test |
| 12 | `go build ./... && go vet ./... && go test ./...` lulus | CI |

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
- UI approval enrollment (opsi B memakai CLI/API lokal dulu; UI menyusul)

## 12. Ketergantungan Terbuka (SUDAH DIPUTUSKAN)

> **Catatan:** tiga pertanyaan di bawah ini sudah dijawab user pada sesi ini. Dev Agent dapat langsung memulai WO-03 tanpa menunggu lagi.

| # | Topik | Keputusan | Keterangan |
|---|---|---|---|
| 1 | Enrollment admin cabang | **D-18: Hybrid A + B** | Manual + auto-provision `pending` + wajib approval. |
| 2 | Identitas pencocokan | **D-16: `iss` + `sub` (UUIDv7)** | Bukan email. Stabil walau email berganti. |
| 3 | Kunci verifikasi SSO | **D-17: HS256 shared secret** | Di balik interface verifier agar bisa upgrade ke RS256. |

### Hal Teknis Tambahan untuk Dev Agent

1. **Skema `admins`** (⚠️ kritis): tabel `admins` tidak punya kolom `level`, `status`, `sso_issuer`, `sso_subject`, `origin`. Migrasi D-18 wajib menambahkan kolom-kolom ini (lihat §5). Tanpa migrasi ini, enrollment tidak bisa berjalan.
2. **Peran/Level**: `admins` hanya punya `is_root` + `is_active`. SSO selalu memetakan ke `admin` (`LevelAdmin`) dan **tidak pernah** memberi `is_root` (D-19). Tidak perlu kolom `federated_level` kecuali pemetaan lebih rumit benar-benar dibutuhkan.
3. **Interface verifier**:

```go
type IdentityVerifier interface {
    Verify(token string, expectAud string) (*Identity, error)
}
// Implementasi awal: verifyHS256(); dapat diganti verifyRS256() di kemudian hari.
```

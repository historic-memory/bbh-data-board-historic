---
title: 'WO-03: SSO Lintas Instance (admin-only) — portal token + keanggotaan lokal'
description: 'Pusat menerbitkan identity token (portal token) ber-aud cabang; cabang memverifikasi LOKAL dengan HS256 (D-17) dan hanya menerima admin yang SUDAH terdaftar di admins cabang (403 bila belum/chuma disetujui). Pencocokan via iss+sub UUIDv7 (D-16). Enrollment hybrid A+B (D-18). Double-token session.'
status: complete
created: "2026-10-26"
updated: "2026-10-27"
revision: 3
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
  "typ": "portal",
  "iat": 1730000000,
  "exp": 1730000900
}
```

> 🔴 **TIDAK ADA klaim peran/izin di token** (`lvl`, `role`, `groups`, `permissions`). Ini keputusan sadar (D-19): **identitas federatif, otorisasi lokal**. Kalau peran ditulis di token, pusat bisa memberi hak lebih di cabang — itu eskalasi yang harus dicegah.

| Klaim | Fungsi |
|---|---|
| `iss` | Instance penerbit — cabang memverifikasi ini penerbit yang dikenali |
| `aud` | **Guard tujuan** — token cabang lain ditolak (AC-14) |
| `sub` | **Kunci pencocokan enrollment**: id admin di DB pusat berupa **UUIDv7** (§5) |
| `atr` | Email/atribut tampilan untuk UI saja — **bukan** kunci pencocokan |
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
| Ditemukan, status `active` | ✅ Lanjut; **izin ditentukan RBAC lokal cabang** (WO-04), bukan dari token |
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

| # | AC | Verifikasi | Status |
|---|---|---|---|
| 1 | Admin terdaftar di cabang masuk portal **tanpa login ulang** (AC-11) | test 2 instance | ✅ |
| 2 | Admin **tidak terdaftar** ditolak **403 + pesan jelas**, meski token sah (AC-12) | test | ✅ |
| 3 | **Pusat dimatikan** → admin terdaftar tetap bisa memakai cabang selama token sah (AC-13) | test matikan pusat | ✅ |
| 4 | Token ber-`aud` cabang lain **ditolak** (AC-14) | test | ✅ |
| 5 | Password admin **tidak pernah** dikirim/disimpan di cabang (AC-15) | grep + query | ✅ |
| 6 | **Auto-provision (D-18):** SSO pertama kali membuat baris `pending`, ditolak hingga disetujui (AC-23) | test | ✅ |
| 7 | **Pencocokan `iss`+`sub` (D-16):** mengganti email tidak memutus akses (AC-24) | test | ✅ |
| 8 | **Otorisasi lokal (D-19):** token tidak membawa peran; hak akses = RBAC cabang (AC-25) | grep token claims + test | ✅ |
| 9 | Admin SSO tanpa grup di cabang → login boleh, tetapi **403** di route ber-izin (D-22) | test | ✅ |
| 10 | Izin cabang berubah → berlaku **tanpa** minta token baru (D-23) | test | ✅ |
| 11 | **Verifikasi HS256 (D-17):** token palsu / tanda tangan rusak → 401 (AC-26) | test | ✅ |
| 12 | `main token` tidak pernah ke cabang; `portal token` tidak pernah ke pusat (AC-21) | grep/trace | ✅ (log & perilaku) |
| 13 | Keluar portal membuang `portal token`, `main token` utuh (AC-22) | test | ⚠️ belum terbukti lewat UI (kode `clearPortalSession` ada) |
| 14 | `go build ./... && go vet ./... && go test ./...` lulus | CI | ✅ |

## 9b. Status Implementasi (Bukti Nyata)

**Lokasi kode (semua di `plugins/`, tidak ada di `internal/`):**

| Berkas | Isi |
|---|---|
| `plugins/portalsso/services/sso.go` | penerbitan & verifikasi portal token (HS256), `IdentityVerifier` |
| `plugins/portalsso/services/enrollment.go` | guard keanggotaan lokal, auto-provision `pending`, list/approve |
| `plugins/portalsso/middleware/middleware.go` | `RequirePortalIdentity` (401 vs 403), `RequireLocalPermission` |
| `plugins/portalsso/handlers/handlers.go` | `POST /api/admin/portal-token`, `GET/POST /api/admin/sso-enrollments[/:id/approve]`, `GET /api/business/whoami`, `GET /api/business/reports` |
| `plugins/portalsso/cli.go` | `console sso list\|approve\|add\|issue\|verify` |
| `plugins/auth/migrations/postgres/2026_10_27_103000_admins_sso_enrollment.*.sql` | `status`, `origin`, `sso_issuer`, `sso_subject`, `password_hash` NULL-able, unique `(sso_issuer, sso_subject)` |
| `plugins/auth/model/models.go`, `model/password.go` | `PasswordHash *string`, `CheckPassword(*string)`, `SetPassword` |
| `plugins/federation/proxy.go` | portal token diteruskan sebagai `Authorization: Bearer` ke cabang; anti-loop hanya untuk hop lanjutan; 401/403 cabang diteruskan apa adanya pada jalur SSO |

**Bukti uji 2 instance** (pusat `INSTANCE_KEY=local`, cabang `INSTANCE_KEY=cabang-jkt` di port 8091):

```text
# AC-6/AC-23 — SSO pertama kali → auto-provision pending, DITOLAK
$ curl .../api/business/whoami -H 'X-Branch-Endpoint: cabang-jkt' -H "X-Portal-Token: $PT"
status=403
{"enrollment_id":"01a17c1a-...","error":"autentikasi berhasil, tetapi pendaftaran Anda masih menunggu persetujuan admin cabang","status":"pending"}
-- branch admins: root@admin.local | pending | sso | iss=local | sub=01a168d4... | password_hash IS NULL = t

# AC-1/AC-2 — setelah approval admin cabang (POST /api/admin/sso-enrollments/:id/approve → 200)
$ curl .../api/business/whoami ...
status=200
{"admin_id":"01a17c1a-...","groups":["viewer"],"instance":"cabang-jkt","issuer":"local",
 "permissions":["admin.read","endpoint.read","group.read","member.read","permission.read"],"subject":"01a168d4-..."}

# AC-9/D-22 — grup dicabut di cabang, token SAMA (tanpa token baru)
$ PUT /api/admins/<id>/groups {"groups":[]}   → 200
$ .../api/business/whoami  → 200, "permissions":[]
$ .../api/business/reports → 403 {"error":"izin ditolak: butuh admin.read"}

# AC-4 — token aud=cabang-jkt dipakai di instance lain
status=403 {"error":"forbidden: token ditujukan untuk cabang lain"}

# AC-3 — pusat di-kill, cabang tetap melayani dengan token yang sama
killed pusat pid=12391
$ curl http://127.0.0.1:8091/api/business/whoami -H 'Authorization: Bearer <portal token>'
cabang_direct=200 { ... "instance":"cabang-jkt" ... }

# AC-5/AC-12 — password tidak ada; main token pusat ditolak cabang; tidak ada token di log
SELECT email, origin, password_hash IS NULL AS no_password FROM admins;  → sso | t
$ curl cabang dengan main token pusat → 401 "token signature is invalid"
grep -c 'eyJhbGci' /tmp/cabang.log /tmp/pusat.log → 0 / 0

# AC-7 — email admin pusat diubah (root@admin.local → root+baru@admin.local), token baru
status=200, display berubah, admin_id cabang TETAP sama (pencocokan iss+sub)

# AC-8 — klaim token tidak memuat peran/izin (unit test memeriksa daftar klaim)
klaim: iss, aud, sub, atr, typ, iat, exp — tanpa role/groups/permissions

# AC-11 — tanda tangan rusak / token asing → 401 (7 unit test + 1 uji integrasi)
```

**Catatan revisi implementasi:**

1. Anti-loop (D-12) semula menolak **setiap** request yang membawa `X-Forwarded-Instance`, sehingga request pusat→cabang ikut ditolak 409. Diperbaiki: penolakan hanya saat request **akan diteruskan lagi** (hop lanjutan). Ada unit test baru untuk kedua sisi.
2. Pemetaan 401/403 → 502 di federation hanya berlaku untuk jalur **kredensial service**; pada jalur SSO status asli diteruskan agar SPA bisa menampilkan alasan (pending/izin kurang).
3. Auto-provision memakai `atr` sebagai `email` tampilan bila valid dan belum terpakai; bila tidak, alamat sintetis `<sub>@<iss>.sso`.

## 10. Definition of Done

- [x] Semua AC lulus dengan bukti output nyata
- [x] **AC-3 dibuktikan dengan benar-benar mematikan pusat** — bukan sekadar alasan "tidak ada callback di kode"
- [x] Interface verifier terpisah jelas (verifikasi ≠ penerbitan)
- [x] Tidak ada token/password di log (grep)
- [x] Rebuild binary eksplisit (`go build -o bin/console ./cmd/console`, `go build -o bin/server ./cmd/server`)
- [x] Bagian yang belum terbukti **dinyatakan belum terbukti** (AC-13 lewat UI)
- [x] Tidak ada push ke remote

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
| 3 | Kunci verifikasi SSO | **D-17: HS256 shared secret** | Di balik interface verifier agar bisa naik ke RS256. |
| 4 | Otorisasi | **D-19: otorisasi LOKAL** | Token hanya identitas; hak akses = RBAC cabang (**WO-04 wajib selesai dulu**) |

### Hal Teknis Tambahan untuk Dev Agent

1. **Skema `admins`** (⚠️ kritis): tabel `admins` tidak punya kolom `level`, `status`, `sso_issuer`, `sso_subject`, `origin`. Migrasi D-18 wajib menambahkan kolom-kolom ini (lihat §5). Tanpa migrasi ini, enrollment tidak bisa berjalan.
2. **Otorisasi = RBAC lokal** (D-19..D-23): setelah identitas terbukti, cabang menghitung izin dari `admin_groups` → `group_permissions`. **Jangan** menyalin izin pusat. Lihat `WO-04-rbac-groups-permissions.md`.
3. **Tidak ada `is_root` dari SSO**: root adalah keanggotaan grup `root` **di cabang**, diberikan admin cabang sendiri.

```go
type IdentityVerifier interface {
    Verify(token string, expectAud string) (*Identity, error)
}
// Implementasi awal: verifyHS256(); dapat diganti verifyRS256() di kemudian hari.
```

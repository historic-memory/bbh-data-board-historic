---
title: 'WO-05: Housekeeping pasca-federasi — docs, env, dan pembuktian AC-13'
description: 'Temuan verifikasi Manager setelah WO-01..WO-04 complete: (1) docs/swagger basi (masih is_root, tanpa route endpoint/federation/portalsso), (2) INSTANCE_KEY/SSO_SHARED_SECRET/FEDERATION_FALLBACK_LOCAL tidak ada di .env.example, (3) AC-13 (keluar portal) belum diuji lewat UI, (4) SSO_SHARED_SECRET dipakai sama dengan APP_KEY — diputuskan FAIL-CLOSED. Tidak mengubah perilaku fungsional selain #4.'
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 3
---

# WO-05: Housekeeping pasca-federasi

**SPEC acuan:** `SPEC.md` (D-19..D-25, AC-13, AC-22)
**Fase:** penutup
**Sifat:** dokumentasi + konfigurasi + pembuktian tertunda, **plus satu perubahan perilaku yang disengaja** (T-4).

## 0. Keputusan Manager (2026-10-27) — DISETUJUI, LANJUT

| # | Keputusan |
|---|---|
| 1 | **WO-05 dikerjakan SEKARANG** dengan urutan **T-1 → T-2 → T-4 → T-3** (biaya & risiko naik bertahap) |
| 2 | **T-4: FAIL-CLOSED** — `SSO_SHARED_SECRET` sama dengan `APP_KEY` → **tolak start**. Bukan sekadar peringatan. |
| 3 | **Diizinkan menambah `SSO_SHARED_SECRET` baru ke `.env`** (gitignored) dengan nilai terpisah: `openssl rand -base64 32` |
| 4 | **Drop `bbh_branch`** (data uji) — **tetapi SETELAH T-3 selesai**, karena T-3 membutuhkan instance cabang hidup |
| 5 | **AC-13 WAJIB DIBUKTIKAN lewat UI** (bukan ditulis "belum terbukti"). Ini satu-satunya bagian federasi yang sisi penggunanya belum pernah diuji |
| 6 | **Drop juga `bbh_cbg_jktbaru` dan `bbh_cbg_smg`** — sisa topic multi-DB yang sudah di-destroy; tidak dipakai fitur federasi |

### Yang DIMAKSUD dan TIDAK dimaksud WO-05

| ✅ Termasuk WO-05 | ❌ Bukan WO-05 |
|---|---|
| Regenerasi swagger | Menguji ulang seluruh alur SSO |
| Lengkapi `.env.example` | Memperbaiki bug fungsional |
| Guard kunci `SSO_SHARED_SECRET` | Menambah fitur federasi |
| **Bukti AC-13 di UI** (satu-satunya uji nyata) | Uji beban / keamanan menyeluruh |

**T-1, T-2, T-4 = dokumentasi & kode — tidak butuh SSO menyala.** Hanya **T-3** yang butuh instance cabang + browser.

### Yang SUDAH terbukti dari uji Dev Agent (tidak perlu diulang)

403 pending · approve→200 · cabut grup berlaku seketika · `aud` salah→403 · pusat mati cabang tetap 200 · main token ditolak cabang · ganti email tidak memutus akses · akun nonaktif→403 · anti-loop 409 · token di log 0.

**Yang belum:** AC-13 — `clearPortalSession()` ada di `client.ts:91` tetapi belum pernah **dijalankan**; tidak ada bukti `localStorage` benar-benar berubah. Tidak bisa dibuktikan lewat `curl`/unit test Go.

### Urutan DB (penting)

```
T-3 (uji AC-13 lewat UI) ── butuh ──► instance cabang ── butuh ──► bbh_branch
        └── setelah T-3 selesai ──► DROP bbh_branch, bbh_cbg_jktbaru, bbh_cbg_smg
```

**Jangan drop lebih dulu** — T-3 akan terhalang.

### Koreksi catatan verifikasi

Commit `1cd6cd8` **ADA**, di **repo historic terpisah** (`.historic/.database/.git`) — bukan di repo kode. `.gitignore:18 .historic/` **memang disengaja** agar dua repo tidak tercampur.

| Verifikasi benar | Verifikasi salah |
|---|---|
| `historic git log` / `historic git show --stat 1cd6cd8` | `git log` / `git cat-file` di repo kode |

`historic git show --stat 1cd6cd8` = 7 files, 359 insertions, termasuk WO-05 (117 baris). **Tidak ada commit yang hilang.**

> ⚠️ Namun pelajaran umum tetap berlaku: `cmd | tail -N` mengembalikan **exit code `tail`**, bukan `cmd`. Status gagal bisa tertelan. Jangan pakai pola itu saat status penting.

## 1. Latar Belakang

Manager memverifikasi laporan akhir WO-01..WO-04. Klaim **terbukti benar** pada:

- Commit `a13645e`, `51b4f06`, `1831ec2`, `bbfeb4a` ada & sesuai judulnya
- `go build ./...` / `go vet ./...` / `go test ./...` lulus; `gofmt -l` bersih
- Paket baru berada di `plugins/` (`endpoint`, `federation`, `portalsso`) — **tidak ada** kode fitur baru di `internal/`
- `RequireRootAdmin` **0 hasil**; kolom `is_root` sudah dihapus dari model
- Migrasi `is_root` → grup `root` **dipisah** dari migrasi hapus kolom (sesuai DoD)
- Portal token memang **tanpa** klaim peran (D-19 dipatuhi di `sso.go`)

Namun ditemukan **3 celah** yang tidak dilaporkan.

## 2. Temuan

### T-1 — `docs/swagger` basi (dokumentasi menyesatkan)

```
docs/swagger/docs.go:391   "Update the provided admin fields. Root only; is_root can only be changed here by a root admin."
docs/swagger/docs.go:891/930/1061   "is_root": { ... }
docs/swagger/swagger.json  4 kemunculan is_root
```

Selain itu **tidak ada** satu pun route `endpoint`, `federation`, atau `portalsso` di swagger (`grep -c` = **0**).

**Dampak:** dokumentasi API publik menyebut konsep yang **sudah dihapus** dan menghilangkan **4 fitur baru**. Siapa pun yang membaca swagger akan salah paham.
**Tingkat:** menengah — tidak merusak runtime, tetapi menyesatkan.

### T-2 — Env baru tidak terdokumentasi

Kode memakai 3 variabel yang **tidak ada** di `.env.example`:

| Variabel | Akibat bila tidak di-set |
|---|---|
| `INSTANCE_KEY` | Identitas instance tidak jelas; audit & `X-Forwarded-Instance` kehilangan label |
| `SSO_SHARED_SECRET` | SSO gagal total: `SSO_SHARED_SECRET is not set` |
| `FEDERATION_FALLBACK_LOCAL` | Fallback tidak aktif (default aman, tetapi tidak terdokumentasi) |

**Dampak:** developer baru meng-clone repo → SSO langsung gagal tanpa petunjuk.
**Tingkat:** menengah — menghambat onboarding.

### T-3 — AC-13 belum diuji lewat UI

`clearPortalSession()` ada di `sub_app/webapp/src/api/client.ts:91`, tetapi **belum pernah dijalankan lewat UI**. Ini sudah dinyatakan jujur sebagai *belum terbukti* oleh Dev Agent — **tidak ada pelanggaran kejujuran**. Yang kurang hanya pembuktiannya.

## 3. T-4 — Kunci dipakai ulang → **DIPUTUSKAN: FAIL-CLOSED**

Contoh perintah reproduksi di laporan memakai nilai `APP_KEY` dari `.env` sebagai `SSO_SHARED_SECRET`:

```
SSO_SHARED_SECRET="base64:LInXfU+02wrG8LS8/1x4RHj6swsd+N0ORzcQ43i4Mn0="
```

Nilai itu **identik** dengan `APP_KEY` di `.env:89`. Artinya **kunci enkripsi aplikasi (AES-256-GCM) dipakai ulang sebagai kunci tanda tangan JWT (HS256)** — dua tujuan berbeda, satu nilai.

**Mengapa berisiko:** satu kunci bocor → dua fungsi ikut berisiko; rotasi salah satu memaksa rotasi keduanya.

### Desain yang diimplementasikan (hasil)

| Kondisi | Perilaku |
|---|---|
| `SSO_SHARED_SECRET` **kosong** | SSO **nonaktif** + **log jelas** (BUKAN crash). Cabang yang tidak memakai SSO harus tetap bisa start |
| `SSO_SHARED_SECRET` **sama dengan `APP_KEY`** | **Tolak start** — `RegisterServices` mengembalikan error |
| `SSO_SHARED_SECRET` berbeda & terisi | Jalan normal |

**Perbandingan harus setelah normalisasi, bukan sekadar string:**

1. Buang prefix `base64:` bila ada
2. Base64-decode bila memang base64 valid
3. Bandingkan **byte hasil decode**

Alasan: `base64:AAAA` dan `AAAA` adalah **nilai yang sama** — perbandingan string akan meloloskannya.

**Pesan error (contoh):**

```
SSO_SHARED_SECRET tidak boleh sama dengan APP_KEY (kunci enkripsi).
Buat nilai terpisah: openssl rand -base64 32
```

**Larangan keras:** JANGAN pernah mencetak nilai kunci (utuh maupun sebagian) ke error/log. Cukup nyatakan "sama dengan" atau "tidak diset".

### Konsekuensi operasional

Begitu guard aktif, **pusat gagal start** sampai `.env` diberi `SSO_SHARED_SECRET` bernilai baru. Karena itu langkah ini **wajib** disertai:

1. Tambah baris ke `.env` (gitignored) dengan nilai baru
2. Tambah baris ke `.env.example` dengan **contoh placeholder**, bukan nilai nyata
3. Catat di changelog/commit message sebagai **perubahan perilaku yang disengaja**

## 4. Pekerjaan

| # | Pekerjaan | File |
|---|---|---|
| 1 | Regenerasi swagger (`swag init`) sehingga `is_root` hilang dan route baru muncul | `docs/swagger/*` |
| 2 | Bila regenerasi tidak memungkinkan, **hapus manual** sisa `is_root` dan tambahkan route baru | `docs/swagger/docs.go`, `swagger.json`, `swagger.yaml` |
| 3 | Tambahkan `INSTANCE_KEY`, `SSO_SHARED_SECRET`, `FEDERATION_FALLBACK_LOCAL` ke `.env.example` + komentar | `.env.example` |
| 4 | Tambahkan peringatan "jangan samakan `SSO_SHARED_SECRET` dengan `APP_KEY`" | `.env.example` |
| 5 | **T-4 (WAJIB, bukan opsional):** tolak start bila `SSO_SHARED_SECRET == APP_KEY` (setelah normalisasi base64); SSO nonaktif + log bila kosong | `plugins/portalsso/services/sso.go` |
| 5b | Tambah `SSO_SHARED_SECRET` baru ke `.env` (nilai terpisah) agar pusat bisa start | `.env` (gitignored) |
| 6 | Uji AC-13 lewat UI: masuk portal → keluar → pastikan `portal_token` hilang, `main_token` utuh | browser + `localStorage` |
| 7 | Perbarui `WO-02`/`WO-03`: ubah AC-13 dari *belum terbukti* menjadi terbukti + bukti | dokumen WO |

## 5. Acceptance Criteria — HASIL (rev 3)

| # | AC | Verifikasi | Status |
|---|---|---|
| 1 | `grep -rn "is_root" docs/swagger/` = **0 hasil** | grep |
| 2 | Swagger memuat route `endpoint`, `federation`, `portalsso` | grep ≥ 1 per fitur |
| 3 | `.env.example` memuat 3 variabel baru dengan komentar jelas | grep |
| 4 | Ada peringatan eksplisit agar `SSO_SHARED_SECRET` ≠ `APP_KEY` | baca file |
| 5 | **Guard aktif:** `SSO_SHARED_SECRET == APP_KEY` → **start ditolak** dengan pesan jelas; **tidak ada nilai kunci tercetak** | jalankan + grep log |
| 5b | `SSO_SHARED_SECRET` kosong → SSO **nonaktif** + log jelas, server **tetap start** (tidak crash) | jalankan tanpa var |
| 5c | Normalisasi diuji: `base64:<X>` vs `<X>` (byte sama) → **ditolak** | unit test |
| 6 | AC-13 terbukti: setelah keluar portal, `localStorage.portal_token` kosong & `main_token` masih ada | uji UI + screenshot |
| 7 | `go build ./... && go vet ./... && go test ./...` tetap lulus | CI |

## 6. Definition of Done

- [x] Semua AC lulus dengan bukti nyata
- [x] **Perubahan perilaku T-4 dicatat eksplisit** di commit message & WO ini
- [x] Guard tidak pernah mencetak nilai kunci (grep log: 0 di semua log)
- [x] Branch yang tidak memakai SSO tetap bisa start (tanpa `SSO_SHARED_SECRET` → healthz 200)
- [x] AC-13 **terbukti** lewat UI
- [x] Rebuild binary eksplisit (`go build -o bin/console ./cmd/console`)
- [x] Verifikasi commit dokumen dengan **`historic git log`**, bukan `git log`
- [x] Tidak ada push ke remote

## 6a. Arsip Bukti (sumber tunggal — jangan hanya di chat)

### Status implementasi T-1..T-4

| Item | Hasil nyata |
|---|---|
| T-1 swagger | `swag init` OK; `grep -c is_root` = **0** di `docs.go`, `swagger.json`, `swagger.yaml`; `paths: 26` ✅. ⚠️ Angka route lama (`endpoints` 15, `sso-enrollments` 2, `portal-token` 1, `whoami` 1) **DICABUT** — diukur ulang Manager per tag: `endpoints` **8**, `sso` **4**, `audit` **1**, total operasi **39**; dua route (`PATCH /api/admin/endpoints/{key}`, `GET /api/business/reports`) berjalan tetapi tidak terdokumentasi → `WO-07`. Lihat WO-06 §7.3 S-1/S-2/S-3 |
| T-1 pendukung | 3 anotasi `@Success {object} gin.H` → tipe eksplisit (`EnrollmentListResponse`, `ApproveResponse`, `WhoAmIResponse`) |
| T-2 env | `.env.example` (111 baris) memuat `INSTANCE_KEY`, `INSTANCE_LABEL`, `SSO_SHARED_SECRET` (+peringatan), `PORTAL_TOKEN_TTL`, `FEDERATION_FALLBACK_LOCAL`; origin `:8082` masuk `CORS_ALLOWED_ORIGINS` |
| T-4 guard | `ValidateSecretConfig()` dipanggil dari `RegisterServices`; menolak start bila material kunci sama (lihat WO-06 untuk perbaikan byte-level) |
| T-4 negatif | `APP_KEY=base64:<APP_KEY asli>` + `SSO_SHARED_SECRET=<APP_KEY asli>` → `failed to run server: portalsso: SSO_SHARED_SECRET tidak boleh sama dengan APP_KEY` |
| T-4 positif | tanpa `SSO_SHARED_SECRET` → server start, `healthz 200`, SSO nonaktif |
| Tanpa kebocoran | `grep -c '<APP_KEY>' /tmp/{pusat,cabang,nosso}.log` → `0 0 0` |

### Bukti AC-13 (klik nyata di `/app/portal`)

```text
afterLogin       : main_token=ADA     portal_token=ADA     active_endpoint=cabang-jkt
afterEnterPortal : main_token=ADA     portal_token=ADA     active_endpoint=cabang-jkt

whoami OK (200)  : {
  "instance": "cabang-jkt",
  "issuer": "bbh-pusat",
  "subject": "01a168d4-ca17-7aaa-abc0-d51865c4ab6d",
  "display": "root@admin.local",
  "admin_id": "01a17c1a-4b45-7d50-86ca-29386649de94",
  "groups": ["viewer"],
  "permissions": ["admin.read","endpoint.read","group.read","member.read","permission.read"]
}

afterExitPortal  : main_token=ADA     portal_token=KOSONG   active_endpoint=KOSONG
DOM snapshot     : main_token ADA · portal_token KOSONG · active_endpoint KOSONG
```

**Kesimpulan AC-13:** terbukti — keluar portal membuang `portal_token` + `active_endpoint`, `main_token` tetap utuh.

### Kendala UI yang ditemukan & diperbaiki saat pembuktian

| # | Gejala | Akar masalah | Perbaikan |
|---|---|---|---|
| 1 | `403` aset `/app/assets/*.js` | `<script crossorigin>` memaksa mode CORS; origin `:8082` tak terdaftar | `CORS_ALLOWED_ORIGINS` + `:8082` |
| 2 | `404` di `/app/portal` | `BrowserRouter` tanpa `basename` | `basename="/app"` |
| 3 | `whoami` 502 "portal token required" | klien kirim portal token via `Authorization` | kirim via `X-Portal-Token`; `main_token` tak pernah ke cabang |
| 4 | proxy dev tak terpakai | SPA diserve server Go | proxy dicabut dari `vite.config.ts` |

### Gate akhir

```text
go build ./... OK · go vet ./... OK · go test ./... 10 paket ok · gofmt bersih
bin/console & bin/server dibangun ulang
DB setelah cleanup: hanya bbh_data_board (bbh_branch, bbh_cbg_jktbaru, bbh_cbg_smg di-drop)
```

## 6b. Verifikasi tambahan Manager (5b/5c)

```text
# 5b — tanpa SSO_SHARED_SECRET, server TETAP start (SSO nonaktif)
$ noso: healthz=200   (env tanpa SSO_SHARED_SECRET)

# 5c — normalisasi prefix terbukti di unit test
TestValidateSecretConfig: identik walau format prefix berbeda → PASS

# guard tidak mencetak kunci
$ grep -c '<APP_KEY 44 char>' /tmp/{pusat,cabang,nosso}.log → 0 0 0
```

## 7. Out of Scope

- Perubahan arsitektur federasi
- RS256/JWKS (interface sudah siap, menjadi WO lanjutan)
- Pencabutan token instan
- UI pengelolaan grup

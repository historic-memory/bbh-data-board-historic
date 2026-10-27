---
title: 'WO-05: Housekeeping pasca-federasi — docs, env, dan pembuktian AC-13'
description: 'Temuan verifikasi Manager setelah WO-01..WO-04 complete: (1) docs/swagger basi (masih is_root, tanpa route endpoint/federation/portalsso), (2) INSTANCE_KEY/SSO_SHARED_SECRET/FEDERATION_FALLBACK_LOCAL tidak ada di .env.example, (3) AC-13 (keluar portal) belum diuji lewat UI, (4) SSO_SHARED_SECRET dipakai sama dengan APP_KEY — diputuskan FAIL-CLOSED. Tidak mengubah perilaku fungsional selain #4.'
status: progress
created: "2026-10-27"
updated: "2026-10-27"
revision: 2
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
| 4 | **Drop `bbh_branch`** (data uji). Buat ulang murah bila perlu demo lagi. |

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

### Desain yang wajib diimplementasikan

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

## 5. Acceptance Criteria

| # | AC | Verifikasi |
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

- [ ] Semua AC lulus dengan bukti nyata
- [ ] **Perubahan perilaku T-4 dicatat eksplisit** di commit message & WO ini
- [ ] Guard tidak pernah mencetak nilai kunci (grep log)
- [ ] Branch yang tidak memakai SSO tetap bisa start (tanpa `SSO_SHARED_SECRET`)
- [ ] Bila AC-13 tetap tidak dapat diuji, tulis **"belum terbukti"** — jangan diklaim
- [ ] Rebuild binary eksplisit (`go build -o bin/console ./cmd/console`)
- [ ] Verifikasi commit dokumen dengan **`historic git log`**, bukan `git log`
- [ ] Tidak ada push ke remote

## 7. Out of Scope

- Perubahan arsitektur federasi
- RS256/JWKS (interface sudah siap, menjadi WO lanjutan)
- Pencabutan token instan
- UI pengelolaan grup

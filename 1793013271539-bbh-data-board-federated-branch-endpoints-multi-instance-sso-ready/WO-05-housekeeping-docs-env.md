---
title: 'WO-05: Housekeeping pasca-federasi — docs, env, dan pembuktian AC-13'
description: 'Temuan verifikasi Manager setelah WO-01..WO-04 complete: (1) docs/swagger basi (masih is_root, tanpa route endpoint/federation/portalsso), (2) INSTANCE_KEY/SSO_SHARED_SECRET/FEDERATION_FALLBACK_LOCAL tidak ada di .env.example, (3) AC-13 (keluar portal) belum diuji lewat UI. Tidak mengubah perilaku fungsional.'
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-05: Housekeeping pasca-federasi

**SPEC acuan:** `SPEC.md` (D-19..D-25, AC-13, AC-22)
**Fase:** penutup
**Sifat:** **tidak mengubah perilaku** — dokumentasi, konfigurasi contoh, dan pembuktian yang tertunda.

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

## 3. ⚠️ Catatan Keamanan yang Perlu Diketahui (bukan bug kode)

Contoh perintah reproduksi di laporan memakai nilai `APP_KEY` dari `.env` sebagai `SSO_SHARED_SECRET`:

```
SSO_SHARED_SECRET="base64:LInXfU+02wrG8LS8/1x4RHj6swsd+N0ORzcQ43i4Mn0="
```

Nilai itu **identik** dengan `APP_KEY` di `.env:89`. Artinya dalam pengujian, **kunci enkripsi aplikasi (AES-256-GCM) dipakai ulang sebagai kunci penanda tangan JWT (HS256)**.

**Mengapa berisiko:** dua kunci dengan **tujuan berbeda** memakai nilai yang sama. Jika satu bocor, dua-duanya ikut berisiko; dan rotasi salah satu memaksa rotasi keduanya.

**Tindakan yang diminta di WO ini:**
1. `SSO_SHARED_SECRET` **wajib** nilai terpisah dari `APP_KEY` — dokumentasikan eksplisit di `.env.example`.
2. Tambahkan **peringatan** di tempat konfigurasi: "jangan pakai nilai APP_KEY".
3. (Opsional, disarankan) Tambahkan pemeriksaan saat startup: bila `SSO_SHARED_SECRET == APP_KEY` → **tolak jalan** atau cetak peringatan tegas.

> Ini **bukan** temuan bahwa kode salah — kode membaca env dengan benar. Yang salah adalah **praktik konfigurasi** saat pengujian, dan ketiadaannya peringatan.

## 4. Pekerjaan

| # | Pekerjaan | File |
|---|---|---|
| 1 | Regenerasi swagger (`swag init`) sehingga `is_root` hilang dan route baru muncul | `docs/swagger/*` |
| 2 | Bila regenerasi tidak memungkinkan, **hapus manual** sisa `is_root` dan tambahkan route baru | `docs/swagger/docs.go`, `swagger.json`, `swagger.yaml` |
| 3 | Tambahkan `INSTANCE_KEY`, `SSO_SHARED_SECRET`, `FEDERATION_FALLBACK_LOCAL` ke `.env.example` + komentar | `.env.example` |
| 4 | Tambahkan peringatan "jangan samakan `SSO_SHARED_SECRET` dengan `APP_KEY`" | `.env.example` |
| 5 | (Disarankan) Peringatan/penolakan saat `SSO_SHARED_SECRET == APP_KEY` | `plugins/portalsso/services/sso.go` |
| 6 | Uji AC-13 lewat UI: masuk portal → keluar → pastikan `portal_token` hilang, `main_token` utuh | browser + `localStorage` |
| 7 | Perbarui `WO-02`/`WO-03`: ubah AC-13 dari *belum terbukti* menjadi terbukti + bukti | dokumen WO |

## 5. Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | `grep -rn "is_root" docs/swagger/` = **0 hasil** | grep |
| 2 | Swagger memuat route `endpoint`, `federation`, `portalsso` | grep ≥ 1 per fitur |
| 3 | `.env.example` memuat 3 variabel baru dengan komentar jelas | grep |
| 4 | Ada peringatan eksplisit agar `SSO_SHARED_SECRET` ≠ `APP_KEY` | baca file |
| 5 | (Bila #5 dikerjakan) startup menolak/peringatan saat dua kunci sama | jalankan |
| 6 | AC-13 terbukti: setelah keluar portal, `localStorage.portal_token` kosong & `main_token` masih ada | uji UI + screenshot |
| 7 | `go build ./... && go vet ./... && go test ./...` tetap lulus | CI |

## 6. Definition of Done

- [ ] Semua AC lulus dengan bukti nyata
- [ ] **Tidak ada perubahan perilaku** selain pemeriksaan kunci (#5) — bila #5 dikerjakan, catat sebagai perubahan perilaku yang disengaja
- [ ] Bila AC-13 tetap tidak dapat diuji, tulis **"belum terbukti"** — jangan diklaim
- [ ] Rebuild binary eksplisit
- [ ] Tidak ada push ke remote

## 7. Out of Scope

- Perubahan arsitektur federasi
- RS256/JWKS (interface sudah siap, menjadi WO lanjutan)
- Pencabutan token instan
- UI pengelolaan grup

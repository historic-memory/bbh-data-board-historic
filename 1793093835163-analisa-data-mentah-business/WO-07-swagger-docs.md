---
title: "WO-07: Swagger, dokumentasi env & README plugin resident"
description: "Anotasi swagger untuk seluruh route /api/admin/residents dan terkait, regenerasi docs/swagger hingga paritas (route nyata = route terdokumentasi), dokumentasikan INSTANCE_KEY di .env.example, serta tambahkan catatan plugin resident di README."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-07: Swagger, Dokumentasi Env & README

**SPEC acuan:** `SPEC.md` (§14), pelajaran WO-07 topic federasi (paritas swagger)
**Prasyarat:** WO-06

## Cakupan

### 1. Anotasi swagger
- Tambahkan komentar anotasi pada setiap handler `plugins/resident/handlers/handlers.go`.
- **Wajib**: tidak ada route yang berjalan tetapi tidak terdokumentasi (pelajaran WO-07 federasi: `PATCH /api/admin/endpoints/{key}` & `/api/business/reports` sempat hilang).
- Definisi response memakai pola `--parseDependency --parseInternal` (sudah aktif di `build.sh`).

### 2. Regenerasi `docs/swagger`
- Regenerasi **di dalam container `app`** (lingkungan kanonik). Bukan di host (snap Go gagal parse stdlib).
- Verifikasi paritas dengan angka **yang dapat direproduksi**:
  - `paths` total, operasi total, jumlah operasi untuk tag `residents`, dll.

### 3. `.env.example`
- Dokumentasikan `INSTANCE_KEY` (nilai default cabang lokal, dipakai sebagai default `branch_key`).
- Bila ada variabel baru lain (mis. path default laporan impor), dokumentasikan juga.
- Komentar harus **sesuai perilaku kode** (pelajaran WO-06 federasi: komentar yang menjanjikan "server menolak start" padahal tidak).

### 4. README
- Tambahkan bagian singkat: plugin `resident` = data core BBH, perintah migrasi, cara impor CSV.

## Bukti yang Harus Dilaporkan
- Angka paritas: `paths` = N, operasi = M, tag `residents` = K.
- **Bukti silang**: daftar route nyata (`grep` pada `RegisterRoutes`) vs daftar path di `docs/swagger/swagger.json` → **tidak ada selisih**.
- `grep -c "resident" docs/swagger/swagger.json` > 0.
- Baris `.env.example` yang ditambahkan (dengan nomor baris).
- `docs/swagger` dapat diregenerasi ulang dan hasilnya **byte-identik** (sha256 sama).
- `go build`/`vet`/`test`/`gofmt` lulus.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | Seluruh route `resident` terdokumentasi di swagger. |
| AC-2 | **Nol selisih** antara route nyata dan path swagger (dibuktikan dua arah). |
| AC-3 | Angka paritas **dapat direproduksi** oleh Manager (bukan angka yang hanya ada di laporan). |
| AC-4 | `INSTANCE_KEY` terdokumentasi di `.env.example`, komentar sesuai perilaku. |
| AC-5 | README memuat cara migrasi + impor. |
| AC-6 | Regenerasi swagger byte-identik. |
| AC-7 | Dibangun di container `app`, bukan host. |
| AC-8 | `go build`/`vet`/`test`/`gofmt` lulus. |

---
title: "WO-05: HTTP API admin + koleksi Bruno"
description: "Paket verifikasi HTTP API admin plugin `resident` (14 route /api/admin/*) memakai koleksi Bruno berbasis file (.bru) di `api-tests/`: environment terpisah, secret via .env gitignored, tag smoke/read/mutation/auth, plus perbaikan kontrak API (list kosong = [] bukan null). Menggantikan rencana awal WO-05 (CLI) atas keputusan owner."
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-05: HTTP API admin + koleksi Bruno

**SPEC acuan:** `SPEC.md` §4, §5, §6, §7
**Prasyarat:** WO-03, WO-04
**Menggantikan:** rencana awal `WO-05-cli-resident.md` (CLI) — **keputusan owner**: fokus ke HTTP API dan verifikasi via **Bruno**, bukan CLI.

## Latar
WO-04 sudah mengirim seluruh HTTP API admin `/api/admin/*`, tetapi verifikasinya
hanya lewat `curl` ad-hoc (tidak reproducible, tidak tersimpan). WO-05 mengubah
itu menjadi **koleksi Bruno** yang versionable di repo, sehingga API bisa
di-regression-test kapan saja (lokal/CI) mengikuti skill `bruno`.

## Cakupan

### 1. Koleksi Bruno (`api-tests/`)
- `bruno.json` (manifest, `defaultEnvironment: Local`), `environments/Local.bru`,
  `.env.sample`, `.gitignore`, `README.md`.
- Cakupan **hanya `/api/admin/*`** (aturan skill: `/api/front/*` dikecualikan).
- Request mencakup **14 route admin + login**:
  - `auth/00-Login.bru` — `POST /api/admin/login`, simpan token sebagai **runtime var**.
  - `smoke/` — `GET` read-only tanpa `id`: list residents / mothers / facilities.
  - `residents/` — create → get → put → patch → add-status → list-status → add-document → list-documents → delete.
  - `facilities/` — create → update.
- Tag: `smoke` (read-only tanpa id), `read` (butuh `residentId`), `mutation`, `auth`.

### 2. Perbaikan kontrak API (`plugins/resident/handlers/handlers.go`)
- Helper `newListResponse`: hasil list kosong diserialkan `[]` (bukan `null`).
  Ditemukan oleh koleksi Bruno saat DB kosong (`{"items": null}`).

## Aturan Aman (dari skill `bruno`)
- Default run = **read-only smoke**. Mutasi hanya dengan **approval per-run**.
- Secret **hanya** dari `.env` (gitignored) via `{{process.env.*}}` — **tidak** di `.bru`, argumen shell, atau commit. `.env.sample` disediakan tanpa nilai.
- Tidak menyimpan/meng-commit PII, body/response sensitif, atau report berisi body.

## Non-Cakupan
- CLI `resident` (dibatalkan/ditunda atas keputusan owner).
- Importer CSV/template (WO-06), swagger (WO-07), generator template (WO-08), validator umpan-balik (WO-09).
- Endpoint front/public (`/api/front/*`).

## Bukti yang Harus Dilaporkan
- `bru --version` = 4.2.1.
- `bru run --env Local --tags=smoke` → lulus.
- `bru run --env Local residents facilities` → lulus (mutasi, dengan approval).
- `go build`/`vet`/`test`/`gofmt` lulus.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | Koleksi Bruno ada di repo (`api-tests/`) mencakup 14 route admin + login. |
| AC-2 | Tag `smoke`/`read`/`mutation`/`auth` memisahkan read-only vs mutasi. |
| AC-3 | Secret hanya via `.env` gitignored; `.env.sample` ada; tak ada token di `.bru`. |
| AC-4 | `bru run --env Local --tags=smoke` lulus (read-only, tanpa side-effect). |
| AC-5 | `bru run --env Local residents facilities` lulus (mutasi, approval). |
| AC-6 | List kosong diserialkan `[]`, bukan `null`. |
| AC-7 | `go build`/`vet`/`test`/`gofmt` lulus. |
| AC-8 | Cakupan hanya `/api/admin/*`. |

## Status Eksekusi (complete)
**Status: complete.** Tidak ada push.

### Bukti
- `bru --version` → **4.2.1**.
- **Smoke read-only:** `bru run --env Local --tags=smoke` → **PASS** (3/3 request, 7/7 test).
- **Mutasi (approval diberikan):** `bru run --env Local residents facilities` → **PASS** (11/11 request, 22/22 test), mencakup create/get/put/patch/status/list-status/documents/list-documents/delete + create/update facility.
- **Kontrak API:** `GET /api/admin/residents` saat DB kosong kini `{"items": []}` (sebelumnya `null`) — diverifikasi oleh assertion `body has an items array`.
- **Kualitas kode:** `go build ./... && go vet ./... && gofmt -l plugins/resident && go test ./plugins/resident/... -count=1` → **lulus** di container `app`.

| # | AC | Hasil |
|---|---|---|
| AC-1 | 14 route + login | ✅ `api-tests/**` |
| AC-2 | Tag smoke/read/mutation/auth | ✅ |
| AC-3 | Secret via `.env` (gitignored) | ✅ `git check-ignore -v api-tests/.env` → ignored |
| AC-4 | Smoke read-only lulus | ✅ 3/3, 7/7 |
| AC-5 | Mutasi lulus | ✅ 11/11, 22/22 |
| AC-6 | List kosong `[]` | ✅ `newListResponse` |
| AC-7 | build/vet/test/gofmt | ✅ |
| AC-8 | Cakupan `/api/admin/*` saja | ✅ |

### Catatan
- Field status pada agregat `ResidentFull` bernama **`current_status`** (bukan `status`); assertion disesuaikan.
- `facilities/01-Create-facility` memakai sufiks `Date.now()` karena kolom `name` UNIQUE (menghindari `409` saat re-run).
- Variabel environment yang ditandai `vars:secret` **tidak** dievaluasi CLI untuk nilai `{{process.env.*}}`; karena itu `token` tidak ditandai secret (nilainya hanya referensi; rahasia nyata tetap di `.env`).

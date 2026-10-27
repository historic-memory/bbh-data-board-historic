---
title: "WO-07: Swagger, dokumentasi env & README plugin resident"
description: "Anotasi swagger untuk seluruh route /api/admin/residents dan terkait, regenerasi docs/swagger hingga paritas (route nyata = route terdokumentasi), dokumentasikan INSTANCE_KEY di .env.example, serta tambahkan catatan plugin resident di README."
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 2
---

# WO-07: Swagger, Dokumentasi Env & README

**SPEC acuan:** `SPEC.md` (§14), pelajaran WO-07 topic federasi (paritas swagger)
**Prasyarat:** ~~WO-06~~ → **USANG**. WO-06 (importer) ditunda (HTTP API + tabel
staging, dikerjakan belakangan). WO ini mendokumentasikan API yang **sudah ada**:
CRUD resident dari WO-04 + endpoint template dari WO-08.

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

## Hasil (2026-10-27) — status: complete

**Commit kode:** `0c9a090`
`docs(resident): regenerate swagger for admin routes + document INSTANCE_KEY/README (WO-07)`

### Perubahan
- `plugins/resident/handlers/handlers.go`: menambah import
  `github.com/rolldone/donisgo/plugins/resident/model` + 4 blank var
  (`_ = model.ResidentFull{}` dst.) agar `swag init` dapat me-resolve tipe
  `model.*` yang dikutip di anotasi. Sebelumnya muncul
  `ParseComment error ... cannot find type definition: model.ResidentFull`.
- `.env.example`: melengkapi komentar `INSTANCE_KEY` (default `branch_key` plugin
  resident, fallback `"local"`, branch eksplisit wajib ada di `endpoints.key`).
- `README.md`: bagian "Plugin `resident` — inti data BBH" (tabel kanonik,
  perintah migrasi, unduh template XLSX/CSV, rencana impor WO-06, kontrak Bruno).
- `docs/swagger/{docs.go,swagger.json,swagger.yaml}` diregenerasi di container
  `app`. **Catatan:** `docs/` di-gitignore (`.gitignore:21`) → artefak tidak
  di-commit, hanya dihasilkan di workspace.

### Angka paritas (dapat direproduksi)
Verifikasi: `swag init -g cmd/server/main.go -o ./docs/swagger --parseDependency --parseInternal` (container `app`).

| Metrik | Nilai |
|---|---|
| `paths` total | 35 |
| path resident-relevan | 8 |
| operasi resident (method+path) | 15 |
| operasi tag `residents` | 12 |
| `grep -c "resident" docs/swagger/swagger.json` | 87 |

### Bukti silang (nol selisih, dua arah)
- Route nyata dari `plugins/resident/plugin.go` `RegisterRoutes` → **15 operasi / 8 path**.
- Swagger `paths` resident-relevan → **15 operasi / 8 path**.
- `ONLY in swagger (not in code): []` dan `ONLY in code (missing docs): []` → `PARITY: OK`.

Daftar operasi (method path): GET/POST `/api/admin/residents`;
GET/PUT/PATCH/DELETE `/api/admin/residents/{id}`; GET/POST
`/api/admin/residents/{id}/status`; GET/POST `/api/admin/residents/{id}/documents`;
GET `/api/admin/mothers`; GET/POST `/api/admin/facilities`; PUT
`/api/admin/facilities/{id}`; GET `/api/admin/resident-template`.

### AC-4 — `.env.example` (baris ditambah di blok INSTANCE_KEY)
```
# Plugin `resident` (data core BBH) juga memakai nilai ini sebagai branch_key
# DEFAULT (D-11: cabang = endpoints.key, tanpa tabel branches). Bila payload
# create/update tidak mengirim branch_key — atau mengirim string kosong — maka
# resident dicatat dengan branch_key = INSTANCE_KEY. Bila INSTANCE_KEY kosong,
# fallback ke "local". branch_key yang dikirim eksplisit WAJIB terdaftar di
# endpoints.key, jika tidak request ditolak 400 (ErrUnknownBranch).
```
Komentar diverifikasi cocok dengan kode: `services.resolveInstanceKey()`
(`INSTANCE_KEY` → `"local"`) dan `services.resolveBranchKey()` (kosong → instance key;
tidak dikenal → `ErrUnknownBranch`/400).

### AC-6 — regenerasi byte-identik (sha256)
Regenerasi ulang menghasilkan hash identik (diff kosong):
```
f1436b54d534669fa7701cc684bea07d5663901599a78c1e6b8aca74cc49e9c5  docs/swagger/swagger.json
9791843b0e9c50a39ccb431758de3768455863082534d71ad0df3efcc592b17e  docs/swagger/swagger.yaml
073bae99c8bb6418cfb3bea066819767aa811c42ae4ff8615cd5920d335a8faf  docs/swagger/docs.go
```
`build.sh` (container `app`): `Swagger docs: tidak ada perubahan isi (byte-identik).` → `BUILD OK`, exit 0.

### AC-7 / AC-8
- Semua dijalankan di container `app` (bukan host).
- `go build ./...` OK · `go vet ./...` OK · `gofmt -l` bersih ·
  `go test ./plugins/resident/...` ok (schema + services) · `build.sh` exit 0.

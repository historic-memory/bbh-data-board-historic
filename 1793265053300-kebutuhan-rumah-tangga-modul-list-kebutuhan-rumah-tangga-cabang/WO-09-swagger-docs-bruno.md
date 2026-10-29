---
title: "WO-09: Swagger, README & koleksi Bruno"
description: "Dokumentasi API hsh (swagger), panduan di README, dan koleksi Bruno untuk alur utama."
status: complete
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - docs
  - swagger
  - bruno
---

# WO-09: Swagger, README & koleksi Bruno

## Referensi
- SPEC: §API & Permission, §Kontrak Template.
- Pola acuan: `plugins/resident/handlers` (anotasi swag + blank var import), `build.sh`
  (`--parseDependency --parseInternal`), `api-tests/` (Bruno), README plugin `resident`.

## Tugas
1. **Anotasi swag** pada handler hsh; daftarkan tipe via blank var import bila perlu
   (pola `resident` agar `swag` resolve `model.*`).
2. **Regenerate** `docs/swagger` di container kanonik (`app`) & pastikan `build.sh` hijau.
3. **README** — bagian "Plugin `hsh` — Kebutuhan Rumah Tangga": migrasi, permission,
   unduh template, alur impor, catatan `.xlsx` saja, UI.
4. **Koleksi Bruno** `api-tests/`:
   - `needs/` kategori & item (create/list/get/patch/alias) — mutasi butuh approval per-run.
   - `needs/` periode & entri, template unduh, validate, import (fixture).
   - `smoke/` tambahan read-only (list kategori/item).
   - Fixture template valid + rusak di `api-tests/fixtures/`.
5. **Paritas**: hitung paths/ops hsh di swagger; pastikan konsisten.

## Bukti yang harus dilampirkan
- `docs/swagger` teregen; paritas path/ops tercatat (angka) & `build.sh` exit 0.
- `bru run --env Local --tags=smoke` hijau; koleksi `needs` (mutasi) hijau saat dijalankan.
- Cuplikan README + daftar request Bruno.

## Kriteria Selesai
- Dokumentasi API lengkap & konsisten dengan implementasi.
- Bruno mencakup alur utama (master → periode → entri → template → impor).
- Tidak ada rahasia/PII di repo; `.env` tetap gitignored.

## Bukti Pelaksanaan

Commit: `37e7e1f` — `docs(hsh): swagger paritas, README plugin & koleksi Bruno (WO-09)`
(21 file, +859/−8).

### 1. Anotasi swag & blank var
Handler hsh sudah membawa godoc swag lengkap: `handlers.go` 110 baris `// @`,
`entry.go` 46, `period.go` 33, `import.go` 29, `validate.go` 22, `template.go` 15.
Blank var untuk resolusi `model.*` dipertahankan/ditambah:
`model.Category`, `model.ItemAlias`, `model.Item` (`handlers.go`),
`model.Period` (`period.go`), `model.ImportBatch` (`import.go`).
Tidak ada anotasi baru yang perlu ditulis — WO-09 hanya memverifikasi cakupan.

### 2. Regenerate `docs/swagger` + `build.sh`
Dijalankan di container kanonik (Go host snap tidak bisa `--parseDependency`):

```
docker compose exec app bash build.sh            # BUILD OK
docker compose exec app bash build.sh --verify-docs   # EXIT=0
```

Keluaran kunci:

```
Swagger docs: tidak ada perubahan isi (byte-identik).
--- manifest docs/swagger (sesudah) ---
7885fafd4715fab0bf402cea36a20bfac42725d5cd4713c75620567ec59c1019  docs.go
77cdb3a28a4f13623f5a70f5934b3cb0a8c5676cee519ce2a03f4bddce2544cc  swagger.json
946e3b25a2e7948f40aca3efdb70b62e7a8c8bfae82e5c39dabce54c4fb7cb87  swagger.yaml
BUILD OK: biner + dokumentasi swagger diperbarui.
```

Regenerasi idempoten (byte-identik) → artefak di disk sudah memuat seluruh
endpoint hsh. Catatan: `docs/` **gitignored** (`.gitignore:25`), jadi tidak ada
file swagger yang ikut di-commit — bukti cukup lewat manifest di atas.

### 3. Paritas swagger ↔ route plugin
Dihitung dari `docs/swagger/swagger.json` dan dibandingkan dengan
`plugins/hsh/plugin.go`:

| Metrik | Nilai |
|---|---|
| Path hsh di swagger | **15** |
| Operasi hsh di swagger | **26** |
| Definisi (schema) hsh | **21** |
| Route terdaftar di `plugin.go` | **26** |
| Paritas | **26/26 (100%)** |

Rincian path (semua di bawah `/api/admin`):

```
hsh-template                     GET
hsh-template/import              POST
hsh-template/validate            POST
hsh/categories                   GET POST
hsh/categories/{id}              DELETE PATCH
hsh/entries                      GET POST
hsh/entries/summary              GET
hsh/entries/{id}                 DELETE PATCH
hsh/import-batches/{id}          GET
hsh/items                        GET POST
hsh/items/{id}                   DELETE GET PATCH PUT
hsh/items/{id}/aliases           GET POST
hsh/items/{id}/aliases/{aliasId} DELETE
hsh/periods                      GET POST
hsh/periods/{id}                 DELETE PATCH
```

Total operasi seluruh API di swagger: **86** (hsh = 26, resident + auth sisanya).

### 4. README
Bagian baru `Plugin \`hsh\` — Kebutuhan Rumah Tangga` disisipkan setelah bagian
plugin `resident` (sebelum `What's included`), 121 baris, berisi: skema 7 tabel
+ catatan enum statis (`plugins/hsh/schema/fields.yaml`), `### Migrasi`,
`### Permission` (pemetaan guard read/write/delete + pengecualian validate),
`### Katalog, periode & entri` (daftar endpoint + kontrak `{items,total}` +
upsert entri 201/200 + 409 hapus periode berisi entri), `### Template & impor`
(curl `?format=xlsx|csv`, tolak `.xls`, validate + header `X-Validate-*`,
impor 2 fasa + `?dry_run=1`, aturan kategori tidak auto-create / item auto-create
+ alias / periode impor non-aktif, audit `import-batches/{id}`), `### UI admin`
(tabel 5 route `/needs/*`), `### Kontrak pengujian`.

### 5. Koleksi Bruno
`api-tests/needs/` (14 request, tag `mutation`/`read`) dengan urutan:

| # | Request | Verifikasi |
|---|---|---|
| 01 | Create category | 201, simpan `hshCategoryId` |
| 02 | List categories | `?include_inactive=1`, kontrak `{items,total}`, 7 kategori seed |
| 03 | Create item | 201, simpan `hshItemId` |
| 04 | List items | filter `q` + `category_id` → total 1 |
| 05 | Patch item | `est_price`/`note` berubah, field lain tetap |
| 06 | Add alias | 201, simpan `hshAliasId` |
| 07 | Create period | 201 (atau 409 bila ada), `is_active=false` dihormati |
| 08 | List periods | resolve `hshPeriodId` dari kode `2030-01` |
| 09 | Upsert entry | 201 pertama / 200 upsert, view join item+kategori |
| 10 | Entry summary | `total` + `by_status.URGENT` per periode |
| 11 | Download template | XLSX `template_hsh_v1` |
| 12 | Validate template | fixture rusak → `X-Validate-Total=8/Valid=1/Rejected=7` |
| 13 | Import template | fixture valid, `?dry_run=1`, 2 baris |
| 14 | Get import batch | audit `batch.total=2`, `rejected_count=0`, 2 baris |

Tambahan read-only: `api-tests/smoke/05-List-needs-categories.bru` (7 kategori
kanonik), `api-tests/smoke/06-List-needs-items.bru` (kontrak + `limit`).
Fixture baru: `api-tests/fixtures/hsh-valid-template.csv` (2 baris valid, periode
`2030-01`, kategori `baby_needs`), melengkapi `hsh-broken-template.csv`.
`api-tests/bruno.json` → name `BBH Admin API (resident + hsh)`;
`api-tests/README.md` diperluas (struktur, urutan `needs/`, catatan tag, catatan
kebersihan data uji).

### 6. Hasil eksekusi
```
bru run --env Local --tags=smoke
  Requests 6 (6 Passed) | Tests 15/15 | Status PASS

bru run --env Local needs
  Requests 14 (14 Passed) | Tests 36/36 | Status PASS
```
Token diambil dari `POST /api/admin/login` (`.env` gitignored) dan diteruskan
lewat `--env-var token=...` — tidak ada token yang ditulis ke disk.

Verifikasi Go setelah perubahan: `go vet ./plugins/hsh/...` bersih,
`go test ./plugins/hsh/...` → `handlers` ok, `schema` ok, `services` ok.
DB dikembalikan ke baseline: `7 kategori / 0 item / 0 alias / 0 periode /
0 entri / 0 batch`.

### 7. Penyimpangan (B-1) — bug ditemukan & diperbaiki
Test `needs/07-Create-period.bru` gagal pada run pertama: response
`is_active=true` padahal request mengirim `false`. Akar masalah:
`model.Period.IsActive` memakai tag `gorm:"default:true"`, sehingga GORM
membuang zero value (`false`) saat `Create` dan DB memakai default `true` —
artinya **periode non-aktif tidak bisa dibuat lewat API** (WO-03).
Perbaikan di `plugins/hsh/services/period.go` (`CreatePeriod`): setelah
`tx.Create`, tulis ulang `is_active=false` secara eksplisit dalam transaksi yang
sama (pola yang sama dipakai importer untuk baris staging). Perubahan ikut di
commit `37e7e1f`. Diverifikasi: run ulang Bruno hijau, `go test` hijau.

### 8. Addendum B-1b — temuan review Manager (commit `a4d3fb0`)
Review Manager membuktikan pola B-1 **masih hidup** di jalur create lain:

```
POST /api/admin/hsh/categories {"code":"zztest","name":"ZZ Test","is_active":false}
→ "is_active": true          # SALAH
POST /api/admin/hsh/items {...,"is_active":false}
→ "is_active": true          # SALAH
```

Ketiga field `IsActive` di `plugins/hsh/model/models.go` (baris 12/26/52)
bertag `default:true`. Jalur *update* aman karena memakai `Updates(map)`.

**Koreksi diagnosis B-1**: GORM tidak sekadar "membuang zero value". Karena
kolom bertag `default`, GORM menganggapnya belum diisi → tidak dikirim pada
INSERT **dan** field in-memory di-*back-fill* dari DB (RETURNING), sehingga
`row.IsActive` sudah bernilai `true` sesudah `Create`. Inilah sebabnya percobaan
pertama B-1b (memeriksa `!row.IsActive` setelah create) **gagal** dan terdeteksi
oleh test baru. Nilai yang diminta harus ditangkap **sebelum** `Create`.

**Perbaikan** (`a4d3fb0`, 5 file, +167/−9):
- `services/service.go`: helper `forceInactive(tx, table, id)` + dokumentasi
  sebabnya.
- `services/category.go`, `services/item.go`: create dalam satu transaksi
  (insert → rewrite `is_active=false` bila diminta) dan mengembalikan `Get*`
  sehingga response = state DB (sebelumnya `CreateCategory` mengembalikan struct
  in-memory).
- `services/period.go`: memakai helper yang sama (B-1 dirapikan).
- `services/active_flag_db_test.go` (baru): `TestCreateHonoursInactiveFlag`
  (kategori/item/periode → response **dan** kolom DB harus `false`) +
  `TestCreateDefaultsToActive` (default tetap `true` bila flag dihilangkan).

**Bukti**:
- `gofmt -l plugins/hsh/` kosong, `go vet ./plugins/hsh/...` bersih,
  `go test ./plugins/hsh/...` → semua paket `ok`; kedua test baru `--- PASS`
  (bukan skip).
- API nyata: kategori `zztest` → `is_active:false`, item `Item ZZ` →
  `false`, periode `2097-03` → `false`; kategori tanpa flag → `true`.
  Kolom DB dikonfirmasi `f` lewat psql, lalu data uji dihapus.
- Regresi: `bru --tags=smoke` 6 req/15 test PASS; `bru needs` 14 req/36 test
  PASS; DB baseline `7 kategori / 0 item / 0 alias / 0 periode / 0 entri /
  0 batch`.

**Catatan proses**: bug ini lolos karena tidak ada test yang menutup
`is_active=false` **saat create** (Bruno & unit test hanya menguji update).
Pelajaran umum: setiap kolom bertag `default:` di GORM perlu test create
zero-value eksplisit.

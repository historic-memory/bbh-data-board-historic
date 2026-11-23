---
title: WO-08 — Swagger, README, Bruno
description: "Paritas swagger + README + koleksi api-tests"
status: complete
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-08 — Swagger, README, Bruno

## Prasyarat

WO-01..WO-07 lulus.

## Lingkup

- Swagger: anotasi handler lengkap sejak WO-02..06 (wajib `@Router`); `build.sh --verify-docs` byte-identik; paritas path/operasi cocok `plugin.go`.
- README root: seksi "Plugin `warehouse` — Management Gudang" (migrasi, permission, route, template/impor, UI, kontrak uji).
- Koleksi Bruno `api-tests/warehouse/` (positif + negatif + RBAC + template/validate/import) + smoke read-only + fixture valid.

## Acceptance Criteria

- AC-1 Paritas swagger 2 arah OK; regen byte-identik.
- AC-2 `bru run --env Local warehouse` + `--tags=smoke` PASS; token ambil via login dulu (jebakan token kedaluwarsa).
- AC-3 Data uji dibersihkan setelah run.

## Catatan Implementasi (Dev)

### 1. Blocker swag ditemukan & diperbaiki

`bash build.sh --verify-docs` gagal (exit 1):

```
ParseComment error in /src/plugins/warehouse/handlers/master.go ... cannot find type definition: model.Category
```

Akar masalah: `swag` me-resolve tipe `model.*` **per-file**. `master.go` mengutip `model.Category` /
`model.Warehouse` pada anotasi tapi tidak meng-import paket `model` warehouse. Pola yang sama sudah dipakai
plugin lain (`hsh`, `sec` di `period.go`) yaitu blank-var.

Perbaikan (2 file):

| File | Perubahan |
| --- | --- |
| `plugins/warehouse/handlers/master.go` | import `github.com/rolldone/donisgo/plugins/warehouse/model` + blok `var (_ = model.Category{}; _ = model.Warehouse{})` |
| `plugins/ingest/handlers/analyze_tool.go` | **PRE-EXISTING** (bukan bagian WO-01..07, berasal dari commit `e95d45e`): import `.../plugins/ingest/model` + `var _ = model.AnalysisSnapshot{}` |

Catatan untuk Manager: file kedua berada **di luar lingkup WO-08** — diperbaiki hanya karena memblokir
`--verify-docs` (regenerasi swagger selalu memindai seluruh `./plugins/...`). Di-commit terpisah dengan label jelas.

`go build ./...` + `go vet ./...` bersih setelah perbaikan.

### 2. Regenerasi swagger byte-identik

`--verify-docs` wajib dijalankan **di dalam container** `bbb-data-board-app-1` (host: GOROOT snap
`/snap/go/11411` gagal, `swag` tidak ada di PATH host). `swag` juga butuh `--parseDependency --parseInternal`
(sudah di dalam `build.sh`) agar tipe lintas-paket (`services.*`, `model.*`) ikut ter-generate.

```
Swagger docs: tidak ada perubahan isi (byte-identik).
ec4c6ea7f279fe213ccf923846db47aa7c4267b52159218901b35f3e16ad5659  docs.go
fb3256fff5b51f98159e7f20f6ba0d469bac78953f3ec5e86374bbf21d86d3ab  swagger.json
97a6f0434e59f6f215ef8cf1b5c7036f8468adbd31ca4c869fa472b0e9a6fb7b  swagger.yaml
BUILD OK: biner + dokumentasi swagger diperbarui.
EXIT=0
```

`docs/swagger/*` **gitignored** → artefak swagger tidak masuk commit (sesuai `README`).

### 3. Paritas swagger 2 arah (AC-1)

Bandingkan `docs/swagger/swagger.json` vs `plugins/warehouse/plugin.go` (path dinormalisasi `{id}` → `:id`):

| Sumber | Path | Operasi |
| --- | --- | --- |
| swagger `warehouse` | 15 | 25 |
| `plugin.go` `warehouse` | 15 | 25 |
| docs-without-code | — | **0 (kosong)** |
| code-without-docs | — | **0 (kosong)** |
| **2-WAY MATCH** | | **TRUE** |

Total repo setelah regenerasi: **141 path / 209 operasi**.

Catatan: handoff menyebut baseline "78 path / 129 operasi"; artefak usang yang ada di disk sebelum
regenerasi terukur **124 path / 182 operasi**. Angka 78/129 tidak dapat direproduksi — kemungkinan
diukur pada snapshot yang berbeda. Angka final yang dipakai: 141 / 209.

### 4. README

- `README.md` (root): bullet modul + seksi penuh "Plugin warehouse — Management Gudang"
  (karakteristik, `migrate --plugin warehouse`, matriks permission `warehouse.read/write/delete`
  dengan `delete` root-only, tabel guard, tabel endpoint master & stok, kolom template/validasi/impor,
  keputusan O-1 auto-create master, O-2 gudang asing fail-closed, idempotensi, tolak `.xls`,
  tanggal `YYYY-MM-DD`, tabel route UI `/warehouse/*`, dan kontrak Bruno).
- `api-tests/README.md`: judul/intro + `warehouse`, baris struktur folder, seksi
  "Urutan `warehouse/`" berisi tabel 45 baris, token viewer, perintah run, baris `smoke` diperluas,
  catatan urutan cleanup (FK: `wh_movements` → `wh_batches` → `wh_import_rows` → `wh_import_batches`
  → `wh_items` → `wh_categories` → `wh_warehouses`), catatan `wh-legacy.xls`.
- `api-tests/bruno.json`: `name` → `BBH Admin API (resident + hsh + sec + dashboard + visit + warehouse)`.
- `sub_app/webapp/README.md`: **tidak diubah** — tidak ada pola seksi per-modul di file tersebut
  (hanya base route, development, nav registry), sehingga kondisi handoff tidak terpenuhi.

### 5. Bruno `warehouse` — mengisi gap (AC-2)

Folder `api-tests/warehouse/` sekarang **45 request / 111 test** (28 request baru nomor 18–45):

- Master CRUD: kategori (create/list/update/duplikat 409/get/delete), produk (create/unit tak dikenal 400/
  list terfilter/update), gudang (create).
- Stok: create batch (qty 40, `entered_at` −60 hari, `expires_at` +10 hari → assert `qty_remaining` 40,
  `age_days` 60, `near_expiry` true), list, get, movement OUT 12.5 → 27.5, insufficient stock 409,
  ADJUST `qty:-2.5` → 25, ADJUST tanpa `reason` 400, list movement (2 baris, terbaru dulu), summary.
- Delete-in-use 409: gudang / produk / kategori.
- Hapus kategori sekali pakai (200) + bukti hard-delete (GET → 404).
- RBAC: viewer read tidak 403, tanpa token 401, viewer delete tidak pernah 200.

Smoke (read-only, tag `smoke`): `12-List-warehouse-categories`, `13-Get-warehouse-summary`
→ folder `smoke` jadi **13 request / 36 test**.

Jebakan terpecahkan: variabel shell `BBH_TOKEN` **di-override** oleh `api-tests/.env` sehingga semua
request 401. Solusi: jalankan dengan `--env-var token=... --env-var viewerToken=...`.

Hasil run:

```
bru run --env Local warehouse --env-var token=... --env-var viewerToken=...
  → 45 request / 45 passed, 111/111 test PASS
bru run --env Local --tags=smoke --env-var token=...
  → 13 request / 13 passed, 36/36 test PASS
```

Data uji memakai suffix `Date.now()` sehingga koleksi dapat dijalankan berulang.

### 6. Pembersihan data & akun uji (AC-3)

Hapus seluruh baris tabel warehouse + akun admin uji:

```
DELETE wh_movements 5 · wh_batches 4 · wh_import_rows 3 · wh_import_batches 4
       wh_items 6 · wh_categories 44 · wh_warehouses 10
sisa baris semua tabel = 0
GET /api/admin/wh/summary → {"total_items":0,"total_batches":0,"total_remaining":0,
                             "near_expiry_batches":0,"items":[]}
```

Akun uji (`whviewer.wo08@test.local`, `whviewer.wo08b@test.local`) di-soft-delete via
`./bin/console auth delete-admin --id <uuid>`; daftar admin menunjukkan seluruh akun uji `soft_deleted=t`.


## Review Manager (2026-11-23) — LULUS ✅

Verifikasi independen Manager:

| Cek | Hasil |
|---|---|
| `build.sh --verify-docs` (container, diulang Manager) | EXIT=0, byte-identik ✅ |
| Paritas path `wh` di swagger.json | Manager hitung ulang: 15 path / 25 ops (awal saya dapat 16/26 karena filter substring `'wh'` ikut menangkap `/api/business/whoami` — keliru filter, bukan dev) → **cocok dengan klaim dev 15/25** ✅ |
| Total repo | 141 path / 209 ops — identik klaim dev ✅ |
| README root | Seksi "Plugin `warehouse` — Management Gudang" ada (README.md:737, +168 baris) — pelaporan "README tidak diubah" merujuk `sub_app/webapp/README.md` (tanpa pola per-modul) ✅ |
| Proses | 10 commit lokal tanpa push; AdminLayout/index.css tak disentuh; data uji 0 baris ✅ |

**Keputusan review:**
1. **WO-08 DITERIMA.**
2. **`e85c0f6` (fix blocker `swag` pre-existing di `plugins/ingest/handlers/analyze_tool.go`) DIPERTAHANKAN** — minimal (import + blank-var, pola identik fix hsh/sec), mem-unblock `--verify-docs` seluruh repo, di-commit terpisah dengan label jelas. Manager mencatat di repo memory: perlu retro-catatan di topic ingest `1793711295286` (bukan pembatalan WO apa pun; `go test` tetap hijau).
3. **Diskrepansi baseline 78/129 → tidak bisa direproduksi = benar**; angka 78/129 itu snapshot usang di catatan Manager dari era WO-08 visit (sebelum banyak plugin ditambah). Keputusan: angka KANON = hasil pengukuran (141/209). Ke depan, handoff Manager tidak lagi memakai baseline hitung usang — hanya minta angka paritas dua arah.
4. Bruno folder 45 req/111 test + smoke 13/36 PASS diterima.

**WO-08 DITERIMA. WO-09 DIBUKA — mulai sekarang** (handoff terpisah di chat; ini WO TERAKHIR).

Jabatan: Manager Agent — Nama Tugas: Review WO-08 warehouse. Sayonara.

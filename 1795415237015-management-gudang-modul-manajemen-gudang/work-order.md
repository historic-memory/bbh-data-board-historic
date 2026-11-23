---
title: work order
description: "Master index WO modul Management Gudang (plugin wh)"
status: progress
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
    - gudang
---

# Master Index — Work Order Modul Management Gudang

Plugin `plugins/warehouse` (ID `wh` di tabel/endpoint; permission `warehouse.*`). SPEC terkunci di `spec.md` (D-1..D-12); sumber kebutuhan = `conversation.md` + `notes.md` (M-1..M-5, EX-1, EX-2).

## Prinsip pinned

- Plugin di `plugins/warehouse/` — TIDAK menyentuh `internal/` (aturan user).
- Tanpa sinkronisasi antar gudang (M-3): keluar/masuk dianggap event lokal per instance.
- Stok = agregasi batch (`SUM(qty_remaining)`), tanpa kolom counter.
- Enum statis di kode + CHECK; template dari `schema/fields.yaml`; satuan dropdown REFERENSI.
- Jalur Excel lengkap (D-9): generator → validator → importer staging.
- Integrasi AI chat (D-10) = segmen `wh` di plugin `ingest`, pola topic `1795107489593`; strictness tetap di Go.
- **Log implementasi WAJIB** per WO (section `## Catatan Implementasi (Dev)`, ±10-20 entri `berkas:baris` + kenapa).

## Daftar Work Order

| WO | File | Status |
|---|---|---|
| WO-01 | `wo-01-skeleton-migrasi.md` | complete |
| WO-02 | `wo-02-permission-master.md` | complete |
| WO-03 | `wo-03-transaksi-stok.md` | complete |
| WO-04 | `wo-04-generator-template.md` | complete |
| WO-05 | `wo-05-validator.md` | complete |
| WO-06 | `wo-06-importer.md` | complete |
| WO-07 | `wo-07-ui-admin.md` | complete |
| WO-08 | `wo-08-swagger-bruno.md` | complete |
| WO-09 | `wo-09-integrasi-ai-chat.md` | pending |

Urutan: WO-01 → WO-02 → WO-03 → (WO-04 → WO-05 → WO-06) → WO-07 → WO-08 → WO-09.

## Micro-decision

O-1 (auto-create kategori/produk) & O-2 (gudang asing fail-closed per baris) **TERKUNCI** oleh owner 2026-11-23 dan sudah diwujudkan di WO-06 (lihat `spec.md` §8). Sisa: O-3 (WO-09 lingkup tool chat).

## Bukti & commit

| WO | Commit kode | Bukti |
|---|---|---|
| WO-01 | `f0b3ff6` | `migrate --plugin warehouse up/down` → 5/0 tabel + 15 index + 3 CHECK; `up` ulang idempoten; `go build/vet/test ./...` hijau; `wh version` OK; server boot + `/healthz` 200 |
| WO-02 | `c73127c` | migrasi auth → 3 permission `warehouse.*` (admin=read+write, viewer=read, delete root-only); 15 route `/api/admin/wh/*`; 8 test DB RUN (409 dup, 400 kategori asing, 400 unit asing, 409 in-use, `forceInactive`); curl 16 kasus 201/200/404/409/400/401; viewer GET 200/POST 403/DELETE 403; admin POST 201/DELETE 403; root DELETE 200 |
| WO-03 | `26d5a31` | 5 route stok (`/wh/batches`, `/wh/batches/:id`, `/wh/batches/:id/movements`, `/wh/movements`, `/wh/summary`); 10 test DB RUN termasuk race 20 goroutine (10 sukses/10×409, sisa 0); curl 26 kasus 201/200/400/401/404/409; viewer read 200 & write 403, admin read+write; stok = `SUM(qty_remaining)` 1 query agregat; data uji 0/0/0/0/0 |
| WO-04 | `739c9b0` | route flat `/api/admin/wh-template` guard `warehouse.read`; kolom 100% dari `schema/fields.yaml` (tanpa hard-code); XLSX 4 sheet — PETUNJUK/DATA/**REFERENSI(hidden)**/CONTOH, 8 kolom DATA, dropdown `A2:A201 => REFERENSI!$A$2:$A$2` (Kategori), `C2:C201 => REFERENSI!$C$2:$C$2` (Gudang), `E2:E201 => REFERENSI!$B$2:$B$10` (Satuan); CSV 200 BOM `ef bb bf` header-only; `format=pdf` 400; tanpa token 401; viewer 200; 6 unit test PASS (incl. Sqref string & dropdown ikut katalog); Bruno 4 request/10 test PASS; data uji 0/0/0/0/0 |
| WO-05 | `1393cd0` | route flat `/api/admin/wh-template/validate` guard `warehouse.read`, **read-only** (0 baris DB berubah sebelum/sesudah seluruh matriks); aturan 100% dari blok `validation:` + `tables` di `fields.yaml` (tambah kind `min_value`); `.xls` ditolak via ekstensi **dan** magic OLE2 → 400; `template_version` mismatch → file-level reject (5/0/5); CSV `wh-broken-template.csv` 9 baris → **9 total/2 valid/7 ditolak**; feedback XLSX sheet `HASIL`+`RINGKASAN` / CSV +`hasil_validasi`+`alasan_ditolak` (header ASLI, bukan normalisasi); `strict=1` → 422 body sama; tanpa token 401; viewer 200; **keputusan owner: Kategori/Gudang asing = WARNING** (fail-closed di importer WO-06); 14 test PASS (incl. DB: warning-only & katalog cocok senyap); Bruno 3 request/8 test PASS (folder 7 req/18 test) |
| WO-07 | `7cb339a` | SPA `/warehouse/*`: `src/api/wh.ts` (tipe manual ikut struct Go, `saveBlob` blob-URL → dist bebas URL backend), `src/lib/whFields.ts` (mirror `fields.yaml` + resep drift-check; 2 arah, 9 satuan, 8 kolom template, near-expiry teks-only), 16 berkas `src/pages/admin/warehouse/*` (kategori/produk/gudang CRUD, stok + badge near-expiry, detail batch + masuk/keluar/penyesuaian + riwayat, template: unduh/uji kering/impor/audit); rute statis **sebelum** `/warehouse/stock/:id` (`App.tsx:113-118`); grup nav "Gudang" filter `warehouse.read` (`menu.ts:94-98`); izin 3 tingkat (`write` tombol mutasi, `delete` tombol hapus) + `PermissionGate` 403 rapi; tanpa sentuh `AdminLayout.tsx`/`index.css`; **AC-1** `tsc -b` + `npm run build` hijau, `grep` URL backend di `dist` kosong; **AC-2** Playwright `:5173` Tailscale: kategori/produk duplikat → 409 tampil, gudang CRUD, batch masuk 20 box expiry 2026-12-10 → badge "Segera kadaluarsa", keluar 999 → 409 "Stok tidak cukup. Sisa batch ini hanya 20 box.", keluar 5 → 15, `ADJUST 0` ditolak klien, `ADJUST +3` → 18 + riwayat `+3 box`, unduh XLSX 10.180 B & CSV 89 B 200 `Content-Disposition`, uji kering 3 baris `pending` master 0, impor **2 dibuat/1 ditolak (gudang asing, HTTP 200)** + `new_masters` 3, audit batch 3 baris, umpan balik `X-Validate-*` 3/3/0; **AC-3** `viewer` read-only (tombol mutasi absen, seksi impor → Alert `warehouse.write`), tanpa `warehouse.read` → sidebar tanpa grup Gudang + gate "Akses ditolak (403)"; **AC-4** `TableSkeletonRows`/`TableStateRow` `colSpan` tanpa prop `columns`/children; data uji dibersihkan (0/0/0/0/0 + 2 admin uji dihapus) |
| WO-06 | `dd5f74f` | migrasi v2 `2026_11_23_140600_create_wh_import_tables` (`wh_import_batches` + `wh_import_rows`, paritas fields.yaml hijau); route flat `/api/admin/wh-template/import` guard `warehouse.write` (+header `X-Import-Batch/Total/Created/Skipped/Rejected`) & `GET /api/admin/wh/import-batches/:id` guard read; fasa 1 = engine WO-05 apa adanya (nol tulisan master); fasa 2 = **satu transaksi** (gudang resolve O-2 fail-closed → kategori/produk **auto-create O-1** lewat map insert → batch via `createBatchTx`, `qty_remaining = qty_in`); dedup `(warehouse_id, item_id, entered_at, qty_in)` termasuk duplikat dalam berkas; CSV 4 baris → **4 total/3 dibuat/1 dilewati**, re-impor → **0/0/4 skipped** + count tak berubah (`SUM(qty_remaining)` 44.50, movements 0); gudang asing → **baris rejected, HTTP 200** + laporan (keputusan owner; `strict=1` → 400); `dry_run=1` → master 0/0/0/0/0 (snapshot identik) & baris `pending`; `.xls` 400; 0 byte 400; header-saja 400; tanpa token 401; viewer write 403 & read 200; item nonaktif → rejected (kontrak WO-03); 11 test importer PASS (8 DB-backed); Bruno folder **18 request/46 test PASS**; data uji 0/0/0/0/0 + admin uji dihapus |


| WO-08 | `5860b08` (+`075c8d3`, `e85c0f6`) | perbaikan blocker `swag` (2 berkas: `plugins/warehouse/handlers/master.go` in-scope + `plugins/ingest/handlers/analyze_tool.go` PRE-EXISTING di luar lingkup, commit terpisah); `go build/vet ./...` hijau; `build.sh --verify-docs` di container `app` → **EXIT=0**, `Swagger docs: tidak ada perubahan isi (byte-identik)`, manifest `ec4c6ea7…`/`fb3256ff…`/`97a6f043…`; **paritas 2 arah MATCH**: swagger `warehouse` **15 path/25 ops** == `plugin.go` **15 path/25 ops**, `docs-without-code` 0 & `code-without-docs` 0; total repo **141 path/209 ops** (artefak usang sebelum regen terukur 124/182 — klaim handoff "78/129" tak tereproduksi); README root + seksi penuh `warehouse` + `api-tests/README.md` (tabel 45 baris, catatan FK cleanup) + `bruno.json`; webapp README **tidak diubah** (tak ada pola seksi per-modul); Bruno `warehouse` **45 request/111 test PASS** (28 baru: master CRUD, batch qty 40/−60d exp +10d → `qty_remaining` 40/`age_days` 60/`near_expiry` true, OUT 12.5→27.5, insufficient 409, ADJUST −2.5→25, ADJUST tanpa reason 400, 2 baris movement terbaru dulu, summary 25/1/1, delete-in-use 409 gudang+produk+kategori, hapus kategori 200 + GET 404, RBAC viewer read≠403 / tanpa token 401 / viewer delete≠200); smoke **13 request/36 test PASS** (`12-List-warehouse-categories`, `13-Get-warehouse-summary`); data uji **0 baris semua tabel** + API summary nol, 2 admin uji soft-deleted; tanpa push |

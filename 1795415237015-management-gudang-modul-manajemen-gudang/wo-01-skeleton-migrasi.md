---
title: WO-01 — Skeleton plugin warehouse + migrasi inti
description: "Skeleton plugin wh + migrasi 5 tabel inti"
status: complete
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-01 — Skeleton plugin `warehouse` + migrasi inti

## Tujuan

Buat plugin `plugins/warehouse` (terdaftar di `cmd/server` + `cmd/console`) dengan migrasi 5 tabel inti. Tanpa route (WO-02+).

## Lingkup

- **A. Skeleton** — `plugin.go` (ID `warehouse`, RegisterRoutes no-op), `cli.go` (`wh version`), `model/{id,models,enums}.go`, `services/service.go` (New/DB/Schema + sentinel error + `forceInactive` pola hsh), `handlers/handlers.go` stub, `schema/{fields.yaml,schema.go,schema_test.go}` (units: pcs/unit/set/box/lusin/gram/kg/liter + aturan template).
- **B. Migrasi** `2026_11_23_140100_create_wh_core_tables.{up,down}.sql` — `wh_categories`, `wh_items` (category_id FK, name UNIQUE per kategori), `wh_warehouses`, `wh_batches` (warehouse_id NOT NULL, qty_remaining CHECK >= 0, entered_at, expires_at NULL), `wh_movements` (direction CHECK OUT|ADJUST). UUIDv7 TEXT PK. Tanpa seed data (D-12).

## Acceptance Criteria

- AC-1 `go build/vet/test ./...` hijau.
- AC-2 `console migrate --plugin warehouse up` → 5 tabel + index + CHECK; `down` → 0 tabel; up ulang idempoten.
- AC-3 Plugin terdaftar & binary jalan.

## Non-tujuan

Route/permission/CRUD (WO-02), transaksi (WO-03), template/impor (WO-04..06), UI (WO-07), swagger (WO-08), AI chat (WO-09).

## Catatan Implementasi (Dev)

- `plugins/warehouse/plugin.go:1` — plugin di `plugins/warehouse/` (ID `warehouse`, prefix `wh_`) sesuai aturan user "fitur = plugin", TIDAK menyentuh `internal/` (SPEC D-1). `RegisterRoutes` sengaja no-op: route butuh permission `warehouse.*` yang baru di-seed di WO-02 — mount lebih awal hanya menambah guard untuk permission yang belum ada.
- `plugins/warehouse/plugin.go:52` — `RegisterServices` menyimpan handle DB + `services.New` + `handlers.New`; TANPA DDL di boot (semua tabel lewat migrasi eksplisit, pola hsh/sec).
- `plugins/warehouse/model/id.go:17` — `NewID()` memakai `internal/uuid.NewString()` (UUID v7, time-ordered). DB TIDAK auto-generate id (`id TEXT PRIMARY KEY`, tanpa DEFAULT) — konvensi D-1.
- `plugins/warehouse/model/models.go:21` — `Category`/`Item`: `is_active` bertag `gorm:"default:true"` (butuh `forceInactive` saat create false, lihat gotcha B-1/B-1b hsh). `wh_warehouses` TIDAK punya `is_active` (SPEC §2 hanya id/name/note/timestamps).
- `plugins/warehouse/model/models.go:78` — `Batch.QtyRemaining` bukan counter mandiri: diturunkan transaksional oleh movement + CHECK `>= 0` (SPEC §2). `ExpiresAt *time.Time` nullable (D-4).
- `plugins/warehouse/model/models.go:108` — `Movement.Qty` diberi komentar bertanda untuk ADJUST (selisih ±, SPEC §3); OUT selalu positif. Semantik tanda ditegakkan di service WO-03, DDL hanya mengunci enum.
- `plugins/warehouse/model/enums.go:8` — enum statis `OUT|ADJUST` sebagai konstanta Go + helper `IsValidMovementDirection` (D-7, tanpa tabel ref_*).
- `plugins/warehouse/schema/fields.yaml:1` — single source of truth: 5 tabel, `enums.wh_direction`, `units` (D-8: pcs/unit/set/box/pak/lusin/gram/kg/liter), blok `template` (kolom batch masuk, WO-04) + `validation` (WO-05).
- `plugins/warehouse/schema/fields.yaml:118` — kolom template memetakan `table.column` sehingga generator WO-04 tidak hard-code kolom; rename=edit YAML (D-9).
- `plugins/warehouse/schema/schema.go:98` — `LookupField`/`EnumCodes`/`UnitCodes`/`RulesFor` di-embed via `//go:embed fields.yaml` (pola hsh/sec), dipakai WO-04..06 tanpa parse ulang.
- `plugins/warehouse/schema/schema_test.go:56` — `TestFieldsMatchMigration`: tiap field YAML harus ada di migrasi (migrasi ⊇ schema), penjaga drift skema.
- `plugins/warehouse/schema/schema_test.go:83` — `TestMigrationHasNoSeedData`: menolak `INSERT INTO wh_` di migrasi inti — memaksa D-12 (seed ditunda, dibuat user).
- `plugins/warehouse/schema/schema_test.go:99` — `TestDirectionCheckConstraint` + `:111` `TestQtyRemainingCheckConstraint` + `:121` `TestWarehouseIDNotNull` + `:131` `TestExpiresAtNullable`: kontrak keras yang diuji lewat regex DDL (gagal build kalau migrasi lepas dari SPEC).
- `plugins/warehouse/migrations/postgres/2026_11_23_140100_create_wh_core_tables.up.sql:1` — 5 tabel dengan urutan FK `wh_categories → wh_items → wh_warehouses → wh_batches → wh_movements`; unique index case-insensitive `LOWER(name)` untuk kategori/items-per-kategori/gudang.
- `plugins/warehouse/migrations/postgres/2026_11_23_140100_create_wh_core_tables.up.sql:47` — `wh_batches`: `warehouse_id NOT NULL` (D-2), `qty_in > 0`, `qty_remaining >= 0`, `entered_at DATE NOT NULL` (sumber umur M-2), `expires_at DATE` nullable (D-4), index agar umur/near-expiry murah.
- `plugins/warehouse/migrations/postgres/2026_11_23_140100_create_wh_core_tables.up.sql:68` — `wh_movements`: CHECK `direction IN ('OUT','ADJUST')` (D-7), `destination` teks bebas tanpa sinkronisasi (D-5/M-3).
- `plugins/warehouse/migrations/postgres/2026_11_23_140100_create_wh_core_tables.down.sql:1` — drop FK-first; tanpa seed yang perlu dibersihkan (D-12).
- `plugins/warehouse/services/service.go:20` — sentinel error (`ErrInsufficientStock` disiapkan untuk WO-03 → 409) + `normalizeUnit` (D-8) + `forceInactive` (pola hsh) + `nilIfBlank`/`trimToNil`/`parseDate` dipakai WO-02/03.
- `plugins/warehouse/handlers/handlers.go:39` — `ListResponse` selalu `[]` (helper `newListResponse` reflect) + `writeError` memetakan sentinel→HTTP (409 untuk duplicate/in-use/stok kurang) sekali, dipakai ulang semua handler lanjutan.
- `plugins/warehouse/cli.go:14` — CLI `wh version` (pola `sec version`); subcommand domain menyusul.
- `cmd/server/main.go:35` & `cmd/console/main.go:19` — registrasi `warehouseplugin.New()`; ini yang membuat target migrasi `warehouse` dikenali `migrate --plugin warehouse`.

### Bukti (2026-11-23, container `app`)

- **AC-1**: `go build ./... && go vet ./...` hijau; `go test ./...` **0 FAIL** (termasuk `plugins/warehouse/schema` 9 test).
- **AC-2**: `./bin/console migrate --plugin warehouse up` → `pg_tables wh_*` = **5** + 15 index + CHECK `qty_in>0`, `qty_remaining>=0`, `direction IN ('OUT','ADJUST')`; `down` → **0** tabel; `up` ulang → **5** tabel, apply kedua **tidak** menambah baris migrasi (idempoten).
- **AC-3**: `./bin/console wh version` → `warehouse plugin 0.1.0`; `./bin/server` boot sukses (GIN listening `:8080`), `GET /healthz` = **200**.
- Tidak ada push. Commit kode terpisah dari commit dokumen historic.

## Review Manager (2026-11-23) — LULUS ✅

Verifikasi independen Manager (bukan hanya mengambil laporan dev):

| Cek | Hasil |
|---|---|
| `git show --stat f0b3ff6` | Scope bersih: hanya `cmd/server` + `cmd/console` (registrasi) + `plugins/warehouse/**` — TIDAK ada file `internal/` berubah ✅ |
| DB spot-check | `pg_tables LIKE 'wh\_%'` = **5**; CHECK `wh_batches_qty_in_check`, `wh_batches_qty_remaining_check`, `wh_movements_direction_check` terpasang ✅ |
| `go test ./plugins/warehouse/...` (container) | `schema` ok (9 test), sisanya no test files (wajar skeleton) ✅ |
| `git log origin/master..HEAD` | Hanya `f0b3ff6` — tidak ada push ✅ |
| Master index | `work-order.md` progress, wo-01 complete, progress 8% ✅ |
| Log implementasi | 17 entri, rujuk D-x/gotcha, kualitas baik ✅ |

**Keputusan review:**
1. **WO-01 DITERIMA.** WO-02 DIBUKA — mulai sekarang (tidak perlu konfirmasi ulang).
2. **Perubahan pra-ada `AdminLayout.tsx` + `index.css`** = fix UI halaman **chat ingest** (tinggi viewport CopilotKit) — milik topic ingest `1793711295286`, BUKAN warehouse. Dev: **BIARKAN uncommitted, jangan disertakan commit WO warehouse manapun.** Manager tangani terpisah (patch UI ingest).
3. Semantik `Movement.Qty` bertanda untuk ADJUST dicatat Manager → WAJIB ditegakkan + ditest di WO-03 (OUT > 0 selalu; ADJUST ±, 409 bila minus melebihi sisa).

Jabatan: Manager Agent — Nama Tugas: Review WO-01 warehouse. Sayonara.



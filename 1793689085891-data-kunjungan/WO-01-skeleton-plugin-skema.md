---
title: WO-01 — Skeleton plugin visit + migrasi skema
description: Plugin plugins/visit, model, stub service/handler, migrasi 3 tabel (visitors, visits, visit_status_history) + enum statis.
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-01 — Skeleton plugin `visit` + migrasi skema

**Prasyarat:** spec.md terkunci (D-1..D-13, O-1..O-7).

## Lingkup

1. Buat `plugins/visit/` mengikuti pola `plugins/sec`: `plugin.go` (`ID()="visit"`, tanpa route di WO-01), `cli.go` (`visit version`), `model/{id,models}.go`, `services/service.go` (stub + sentinel error), `handlers/handlers.go` (stub), `schema/{fields.yaml,schema.go,schema_test.go}`.
2. Daftarkan plugin di `cmd/server/main.go` + `cmd/console/main.go`.
3. Migrasi `2026_MM_DD_HHMMSS_create_visit_core_tables` — 3 tabel sesuai spec §4:
   - `visitors` (id UUIDv7 PK, full_name NOT NULL, origin, phone_raw, phone_e164 UNIQUE bila terisi, note, audit) + index `phone_e164`.
   - `visits` (id, visitor_id FK, branch_key NOT NULL, party_size INT NULL, party_raw, status_code NOT NULL default `SCHEDULED` + CHECK, scheduled_at TIMESTAMPTZ NULL, note, audit) + index `(branch_key, scheduled_at)`, `visitor_id`.
   - `visit_status_history` (id, visit_id FK, status_code + CHECK, effective_at, note, changed_by, created_at) + index `(visit_id, effective_at)`.
4. Enum `visit_status` statis di `fields.yaml` + konstanta Go: `SCHEDULED, DONE, CANCELLED, NO_SHOW, RESCHEDULED`.
5. **Tanpa seed** (impor data awal di WO-09).

## Acceptance Criteria

- AC-1: `go build ./...` + `go vet` + `go test ./...` hijau di container `app`.
- AC-2: `console migrate --plugin visit` up → 3 tabel + CHECK + index ada; down → 0 tabel; up ulang idempoten.
- AC-3: `visit version` merespons.
- AC-4: `fields.yaml` memuat enum 5 nilai; test paritas schema lulus.

## Catatan

- Tidak ada tabel `branches` (R-3) dan `ref_*` (R-4).
- GORM `default:` jebakan B-1b (hsh) — catat untuk WO-02.

## Bukti

- Kode commit `d0ca37a` (14 file, +747/−2).
- `go build ./...` + `go vet ./plugins/visit/...` + `go test ./...` hijau di container `app`.
- `console migrate --plugin visit up` → 3 tabel + 2 CHECK + 5 index (termasuk partial UNIQUE `phone_e164`); `down` → 0 tabel; `up` ulang idempoten (`current=1`, tanpa apply kedua).
- `console visit version` → `visit plugin 0.1.0`.
- `go test ./plugins/visit/schema` lulus: cakupan field vs migrasi, prefix `visit_`, tanpa `ref_*`, CHECK 5 nilai, partial unique index, urutan enum.
- Tabel final: `visit_visitors`, `visit_visits`, `visit_status_history`. Konstanta enum di `model/enums.go`.

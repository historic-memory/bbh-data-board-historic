---
title: WO-08 — Swagger, README, Bruno, RBAC e2e kunjungan
description: Paritas swagger route plugin visit, seksi README, koleksi api-tests/visit + smoke, uji RBAC end-to-end.
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-08 — Swagger + README + Bruno + RBAC e2e

**Prasyarat:** WO-02..WO-07.

## Lingkup

1. Swagger: `build.sh --verify-docs` di container `app` (`--parseDependency --parseInternal`); paritas dua arah jumlah route `plugin.go` vs dok swagger untuk prefix `visit`; regen byte-identik.
2. README: seksi "Plugin `visit` — Kunjungan" (migrasi, permission, route, template/validate/import, UI, cara impor Notion).
3. Bruno: `api-tests/visit/` (CRUD + status + template + validate + import) + `smoke/` read-only baru + fixture valid/broken; `bruno.json` name diperbarui.
4. RBAC e2e: admin dgn hanya `visit.read` → tulis 403; viewer → read 200; tanpa token 401.
5. Bersihkan data uji.

## Acceptance Criteria

- AC-1: Paritas swagger OK (nol selisih); `docs/` tetap gitignored.
- AC-2: `bru run --env Local --tags=smoke` PASS; folder `visit` PASS.
- AC-3: Bukti RBAC tercatat di laporan WO.

## Bukti

**Commit:** `4bd62f7` — `docs(visit): swagger + README + koleksi Bruno + RBAC e2e (WO-08)` (23 file, +837/−5).

**AC-1 (swagger):**
- Awalnya `swag init` gagal: `cannot find type definition: model.StatusHistory`
  lalu `model.Visitor` — `plugins/visit/handlers` tidak meng-import paket
  `model`. Perbaikan: import `plugins/visit/model` + blank var
  (`var _ = model.StatusHistory{}` di `status.go`, `var _ = model.Visitor{}` di
  `visitor.go`), pola yang sama dengan `plugins/resident/handlers`.
- `docker compose exec -T app bash build.sh --verify-docs` → `BUILD OK: biner +
  dokumentasi swagger diperbarui`; run kedua → **"tidak ada perubahan isi
  (byte-identik)"**.
- Paritas: `plugin.go` mendaftarkan **16 operasi** (5 visitors + 5 visits + 2
  status + 4 template/import), swagger memuat **9 path / 16 operasi** untuk
  prefix `visit` → **selisih 0**. Total repo: **78 path / 129 operasi / 122
  definisi**. `docs/` tetap gitignored (tidak ada artefak yang di-commit).

**AC-2 (Bruno):**
```
bru run --env Local --tags=smoke   → 11 request / 32 test  PASS
bru run --env Local visit          → 15 request / 34 test  PASS
```
Folder `api-tests/visit/` (15 request): create visitor → list (cari via HP) →
patch visitor → create visit (status awal `SCHEDULED`, `visitor_name` ter-join)
→ list visits → get visit → add status `DONE` (201) → list status (append-only,
`DONE` terakhir) → patch visit (`current_status` tetap `DONE`) → download
template CSV → validate `fixtures/visit-broken-template.csv` (header
`X-Validate-Total=9`, `Valid=6`, `Rejected=3`) → import
`fixtures/visit-valid-template.csv` dengan `?dry_run=1` (`staged`, total 2,
created 0) → audit batch (2 baris staging, `phone_e164` ternormalisasi `+62`) →
delete visit → delete visitor. Smoke baru: `smoke/10-List-visits.bru`,
`smoke/11-List-visitors.bru`. `bruno.json` name diperbarui.

Catatan: satu test sempat gagal karena `rows[].action` masih kosong pada `dry_run`
(fase 1 belum menetapkan aksi final) — ekspektasi diperbaiki menjadi
`["", "pending"]`.

**AC-3 (RBAC e2e, port 8082):**

| Aktor | GET visits/visitors/template | POST visitor/visit/status/import | DELETE visitor |
|---|---|---|---|
| admin grup `viewer` (read-only) | 200 / 200 / 200 | 403 / 403 / 403 / 403 | 403 |
| admin tanpa `visit.read` | 403 / 403 | — | — |
| tanpa token | 401 | 401 | — |

Admin & grup uji dihapus setelah verifikasi; data uji bersih
(`visit_visitors=0 visits=0 staging=0`).

**Berkas:** `README.md` (seksi "Plugin `visit`"), `api-tests/README.md`,
`api-tests/bruno.json`, `api-tests/visit/*` (15), `api-tests/smoke/10-11`,
`api-tests/fixtures/visit-valid-template.csv`, `plugins/visit/handlers/{status,visitor}.go`.

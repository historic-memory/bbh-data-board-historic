---
title: WO-08 — Swagger, README, Bruno, RBAC e2e kunjungan
description: Paritas swagger route plugin visit, seksi README, koleksi api-tests/visit + smoke, uji RBAC end-to-end.
type: work-order
status: pending
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

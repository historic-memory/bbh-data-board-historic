---
title: WO-08 — Swagger, README, Bruno
description: "Paritas swagger + README + koleksi api-tests"
status: pending
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

(disi bertahap)


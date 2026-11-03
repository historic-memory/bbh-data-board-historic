---
title: WO-02 — Permission + CRUD visitors & visits + dedup phone
description: Seed permission visit.* via migrasi plugin auth; CRUD master pengunjung dengan dedup E.164; CRUD booking kunjungan berdimensi cabang.
type: work-order
status: pending
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-02 — Permission + CRUD `visitors` & `visits` + dedup phone

**Prasyarat:** WO-01.

## Lingkup

1. Migrasi plugin **auth** `2026_..._add_visit_permissions`: seed `visit.read`, `visit.write`, `visit.delete` (admin = read+write, viewer = read; root via `*`).
2. Route `/api/admin/visit/visitors` + `/:id` (list/get/create/patch/delete) — guard `visit.read/write/delete`.
   - **Dedup saat create:** normalisasi HP ke E.164 (08…→+62…, 821…→+62…, rapikan `+<kode>`, spasi buang); bila `phone_e164` sudah ada → 409 `ErrDuplicateVisitor` dengan id yang bentrok di body.
   - HP tidak dapat dinormalisasi → simpan `phone_raw` saja, `phone_e164` NULL.
3. Route `/api/admin/visit/visits` + `/:id` (list/get/create/patch/delete) — guard sama.
   - Create wajib `visitor_id` valid (404 bila tidak ada) & `branch_key` ada di `endpoints.key` (400 `ErrUnknownBranch`, pola resident).
   - `party_size` opsional INT; `party_raw` opsional TEXT (D-12).
   - List: filter `branch_key`, `status`, rentang `scheduled_from`/`scheduled_to`, query nama/HP (JOIN visitors), paginasi `{items, total}`.
4. Anotasi swagger lengkap di setiap handler sejak WO ini (pola sec — WO-08 jadi murah).
5. Test: unit + DB test (dedup, branch tidak dikenal, party_size NULL).

## Acceptance Criteria

- AC-1: Build/vet/test hijau.
- AC-2: curl — create visitor 201; duplikat HP 409; create visit 201 dengan status awal `SCHEDULED` + 1 baris history otomatis (lihat WO-03 bila paralel; kalau belum, history dibuat di WO-03).
- AC-3: Viewer (grup viewer) GET 200, POST 403.
- AC-4: `branch_key` tak dikenal 400; unauthenticated 401.

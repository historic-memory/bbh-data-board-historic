---
title: WO-02 — Permission + CRUD visitors & visits + dedup phone
description: Seed permission visit.* via migrasi plugin auth; CRUD master pengunjung dengan dedup E.164; CRUD booking kunjungan berdimensi cabang.
type: work-order
status: complete
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

## Bukti

- Kode commit `2019f78` (13 file, +1428/−10).
- Migrasi plugin auth `2026_11_03_100100_add_visit_permissions` → 3 permission; admin=`visit.read`+`visit.write`, viewer=`visit.read` (root via `*`).
- AC-1: `gofmt -l` kosong, `go build ./...`, `go vet ./plugins/visit/...`, `go test ./...` hijau.
- AC-2: create visitor `201` (`phone_raw 0812-1111-2222` → `phone_e164 +6281211112222`); create ulang dengan `+62 812 1111 2222` → `409 {"visitor_id":"..."}`; create visit `201` status `SCHEDULED`, `party_raw 2-4`, `party_size` NULL, `scheduled_at 2026-04-23T09:00+08 → 2026-04-23T01:00Z`; DB `visit_status_history` berisi 1 baris `SCHEDULED` (baris awal dibuat di WO-02, agar AC terpenuhi; WO-03 menambah transisi/list).
- AC-3: viewer `GET /visitors` 200, `GET /visits` 200, `POST /visitors` 403 (`butuh visit.write`), `DELETE /visits/:id` 403.
- AC-4: `branch_key=nope` → 400 `branch_key tidak dikenal`; tanpa token → 401.
- Tambahan terbukti: list filter `branch_key`+`status`(status agregat)+`q`+rentang tanggal → 1 baris/total 1; PATCH catatan/`party_size` 200; PATCH `status_code` ditolak 400 (ubah status lewat WO-03).
- Test DB RUN (bukan skip): dedup E.164 (+id bentrok), HP ambigu (`06-40961768`) → e164 NULL & raw tersimpan, default status + 1 history, branch tak dikenal, visitor tak dikenal, status eksplisit (`done`) tersimpan, `party_size=0` tidak hilang (jebakan GORM `default:`).
- Data uji dibersihkan (0 visitors/0 visits/0 history); akun viewer uji dihapus.
- `.gitignore` tidak berubah; tidak ada push.

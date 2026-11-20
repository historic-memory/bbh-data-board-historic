---
title: "Work order — CRUD master data per segmen via tool chat generik"
description: "Master index WO topic 1795107489593: 5 WO (patch editor → segment_read → write/apply+kartu → MergeVisitors → prompt+e2e); SPEC disetujui owner 2026-11-20; WO-01 progress (handoff dev), WO-02..05 pending"
status: progress
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - crud
    - work-order
---
# Work order — CRUD master data per segmen via tool chat generik

Topic `1795107489593` (extends `1795083433686`). Sumber kebenaran:
`notes.md` (DESAIN FINAL owner ok 2026-11-20) + `spec.md` (draft).
**SPEC disetujui owner 2026-11-20**. WO-01 complete (review LULUS 2026-11-20,
kode `111bf03`+`8d0eafb`); WO-02 & WO-02a complete (2026-11-20, kode
`9aa0933`); WO-03..05 pending (urutan WO-03→04→05).
Backlog dari review WO-01 (tidak blocker): (a) default `kind` dari
`fields.yaml` untuk bentuk mapping legacy; (c) sesi uji dev
`wo01-smoke-1795126932` boleh dibersihkan.

## Bukti WO-02 / WO-02a (2026-11-20)

- Unit test Go `plugins/ingest/services/segment_read_test.go` (13 test):
  whitelist per segmen, larangan keras, higiene path, `:id`, jepitan
  `limit`, deidentify (ID utuh + objek PII), truncation, key kosong
  fail-closed, pemetaan status upstream. `go build/vet/test ./...` hijau.
- Node: `tool-schema.test.ts` +1 kasus `query` bebas → `npm test` 21/21;
  `tsc --noEmit` hijau; `npm run build` webapp hijau.
- Bruno `api-tests/ingest-crud/` 18 request / 39 test HIJAU: positif 4
  segmen (visit, resident, sec, hsh), whitelist 400 (lintas segmen +
  path terlarang keras), viewer 200 / tanpa token 401, `truncated` +
  jepitan 200 baris, introduce positif, 409 state salah, 400 modul asing,
  401 tanpa token.
- Smoke UI `http://100.125.175.34:5173`: sesi baru `uploaded` → chat murni
  memilih modul Kunjungan → agent memanggil `introduce_session` → DB
  `state=introduced`; lanjut `segment_read` → agent menjawab tabel tanpa
  `full_name`/`phone_raw`, menyebut `truncated: true` + `total 2990`, dan
  menganjurkan operasi by ID. `ingest_audit provider='ingest-tool'`
  mencatat 200 untuk kedua tool.
- Regresi editor (AC-4 WO-02a): `SaveMapping` tetap memakai transisi yang
  sama (`introduceFromUploaded`); koleksi `ingest/` tidak berubah perilaku.

## Latar

Owner ingin agent chat punya tools CRUD lengkap untuk master data segmen
sesi ("pokoknya toolsnya lengkap"). Kasus nyata pertama: kunjungan dev
dobel hasil re-impor (`time_only`). Setelah diskusi arsitektur (2026-11-20),
owner memilih **Hybrid**: 2-3 tool generik untuk semua segmen via proxy
API admin existing + proposal/approve untuk tulis.

## Daftar WO

| WO | Judul | Status | Commit kode | Dok historic |
|---|---|---|---|---|
| WO-01 | Patch editor: hapus tombol Ekstrak & Konfirmasi + modal Simpan draft → chat | **complete** (review LULUS 2026-11-20) | `111bf03`, `8d0eafb` | wo-01-hapus-tombol-ekstrak-konfirmasi-editor.md |
| WO-02 | Backend `segment_read` (whitelist + deidentify) + skema Node | **complete** (2026-11-20) | `9aa0933` | wo-02-segment-read.md |
| WO-02a | Tool `introduce_session` (chat murni dari `uploaded`) — keputusan owner review WO-01 | **complete** (2026-11-20) | `9aa0933` | wo-02a-introduce-session-tool.md |
| WO-03 | Backend `segment_write_propose` + `apply_record_actions` + kartu approve | pending (2026-11-20) | — | wo-03-write-propose-apply.md |
| WO-04 | `MergeVisitors` plugin visit + endpoint admin | pending (2026-11-20) | — | wo-04-merge-visitors-visit.md |
| WO-05 | Prompt agent + Bruno + uji end-to-end duplikat kunjungan | pending (2026-11-20) | — | wo-05-prompt-bruno-e2e.md |

## Prinsip lintas WO (pinned)

1. **P-3/P-5/D-11/O-25 topic induk TIDAK berubah**: AI hanya mengusulkan
   (tulis via proposal → `confirmed_by_user` → Go eksekusi); PII
   dideidentify sebelum ke LLM; Node hanya skema; snapshot jsonb per
   sesi tanpa tabel relasional baru.
2. **Whitelist per segmen di Go** (fail-closed 400) + **RBAC plugin
   target**: baca = permission baca plugin target; tulis = permission
   tulis plugin target; larangan keras endpoint non-segmen
   (`admins/groups/permissions/audit/ingest/providers/dashboard/
   endpoints`).
3. **Eksekusi tulis = panggilan API admin asli** → RBAC + audit endpoint
   tercatat; ingest menambah audit `provider='ingest-tool'` sendiri
   (audit ganda).
4. **Kartu approve di chat** dibuat backend Go (pola `rejected_listing`
   WO-03), bukan teks LLM.
5. **Log Implementasi (Dev)** wajib per WO (aturan pinned topic induk).
6. Kode: backend hanya `plugins/ingest/services` (+ `plugins/visit`
   khusus WO-04) + UI `sub_app/webapp/src/pages/admin/ingest/` + Node
   `sub_app/ingest_runtime/`.
7. Restart `ingest-runtime` setelah perubahan Node (tsx non-watch); Go
   rebuild + restart server (pola terdokumentasi).

## Urutan eksekusi

WO-01 (patch editor, independen) → WO-02 → WO-03 → WO-04 → WO-05.
Kasus uji akhir: kunjungan dobel dev end-to-end via chat (WO-05).

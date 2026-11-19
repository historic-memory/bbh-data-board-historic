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
**SPEC disetujui owner 2026-11-20**. WO-01 progress (handoff dev);
WO-02..05 pending (urutan WO-01→02→03→04→05; WO-02 bebas paralel
bila kapasitas dev memungkinkan).

## Latar

Owner ingin agent chat punya tools CRUD lengkap untuk master data segmen
sesi ("pokoknya toolsnya lengkap"). Kasus nyata pertama: kunjungan dev
dobel hasil re-impor (`time_only`). Setelah diskusi arsitektur (2026-11-20),
owner memilih **Hybrid**: 2-3 tool generik untuk semua segmen via proxy
API admin existing + proposal/approve untuk tulis.

## Daftar WO

| WO | Judul | Status | Commit kode | Dok historic |
|---|---|---|---|---|
| WO-01 | Patch editor: hapus tombol Ekstrak & Konfirmasi + modal Simpan draft → chat | **progress** (handoff 2026-11-20) | — | wo-01-hapus-tombol-ekstrak-konfirmasi-editor.md |
| WO-02 | Backend `segment_read` (whitelist + deidentify) + skema Node | pending (2026-11-20) | — | wo-02-segment-read.md |
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

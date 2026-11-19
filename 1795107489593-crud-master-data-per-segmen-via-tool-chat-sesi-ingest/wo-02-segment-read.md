---
title: "WO-02 — Backend segment_read (whitelist + deidentify) + skema Node"
description: "POST /admin/ingest/tools/segment_read — proxy GET-only ke API admin segmen sesi dengan whitelist prefix per segmen (fail-closed di Go), RBAC baca plugin target, deidentify tunggal sebelum response ke LLM (ID utuh), paginasi max 200; skema Node pola existing"
status: pending
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - backend
    - tool
    - crud
---
# WO-02 — Backend: `segment_read` (whitelist + deidentify) + skema Node

Sumber: SPEC `spec.md` topic `1795107489593` + desain final `notes.md`
(owner ok 2026-11-20). Prasyarat: SPEC disetujui owner. Status:
**pending**. Pola ikut tool strict WO-01 topic `1795083433686`
(`analysis_tools.go` + `handlers/analysis.go`).

## Lingkup

### A. Endpoint Go

1. `POST /admin/ingest/tools/segment_read` — input
   `{session_id, path, query?}`:
   - `path` RELATIF segmen sesi (contoh segmen visit:
     `visit/visitors`, `visit/visits`); tanpa slash awal; segmen sesi
     wajib ada (`target_plugin` kosong → 400 `ErrSegmentRequired`,
     pola T-1).
   - **GET-only**: method lain → 400 (internal hanya GET; jangan
     pernah meneruskan tulisan lewat jalur baca).
   - Ambil ID admin + permission dari token user (Bearer user), BUKAN
     service token — proxy meneruskan otorisasi user.
2. **Whitelist prefix per segmen** (map statis di Go, fail-closed —
   `ErrSegmentPathForbidden` 400 bila di luar daftar):
   - `visit` → `visit/visitors`, `visit/visits`, `visit/visits/:id`,
     `visit/visitors/:id`, `visit/visits/:id/status`
   - `resident` → `residents`, `residents/:id`, `residents/stats`,
     `facilities`, `facilities/:id`
   - `sec` → `sec/*`; `hsh` → `hsh/*`, `needs/*` (map yang sederhana
     per plugin ID).
   - **LARANGAN keras** (400, apapun segmennya): `admins`, `groups`,
     `permissions`, `audit`, `ingest/*` (termasuk tools sendiri),
     `providers`, `dashboard`, `endpoints`, `resident-template`.
     Path/segment di luar daftar → 400.
3. **RBAC baca plugin target**: pemanggil wajib punya permission baca
   segmen (`visit.read`, `resident.read`, `sec.read`, `hsh.read`) —
   tanpa itu 403 (JANGAN cukup `ingest.write` untuk baca lintas modul).
4. **Deidentify tunggal sebelum response ke LLM** (P-5): response JSON
   dari API asli dilewati jalur `DeidentifyColumns` (pola
   `safeRowValues` WO-01 topic lama) — field yang ter-mapping PII
   dimask deterministik; **ID tetap utuh** agar operasi by ID mungkin;
   fail-closed: key deidentify kosong → 503 (sama dengan safe-payload).
5. **Paginasi asli API** (W-7): `query` diteruskan (q/limit/offset/
   branch_key/…); tool memotong respons max **200 baris** per petikan
   (bila API kembalikan lebih, potong + tandai `truncated`).
6. Guard `ingest.write` (masuk daftar tools) + audit
   `provider='ingest-tool'` — audit ganda dengan endpoint asli (W-8).

### B. Node

1. Skema 1 tool `segment_read` di `copilot-runtime.ts` — pola existing
   (JSON Schema → `convertJsonSchemaToZodSchema`, Bearer user,
   `X-Ingest-Session-Id`, tanpa logika bisnis).
2. Deskripsi tool: "baca data master segmen (deidentified) — jangan
   tampilkan nilai PII mentah ke user; operasi selalu by ID".

## Acceptance Criteria

- AC-1 Baca visit visitors/visits via tool → data terdeidentify, ID
  utuh, jumlah baris ≤200 + flag `truncated` bila dipotong.
- AC-2 Whitelist: path di luar segmen (mis. sesi visit meminta
  `sec/items`, atau `admins`) → 400 dengan pesan jelas.
- AC-3 RBAC: pemanggil tanpa permission baca target → 403; tanpa
  token → 401; viewer dengan `visit.read` → 200 baca.
- AC-4 GET-only: body yang mencoba tulis via jalur baca tidak berubah
  apa pun (bukti DB/audit).
- AC-5 P-5: nilai PII tidak mentah di response tool (uji kolom
  `full_name`/`phone_raw` visit → masked); fail-closed 503 bila key
  deidentify kosong.
- AC-6 Audit `provider='ingest-tool'` tercatat per panggilan.
- AC-7 `go build/vet/test` hijau + `tsc --noEmit` Node hijau; Bruno
  (positif 4 segmen + negatif whitelist/RBAC); **Log Implementasi
  (Dev)**.

## Non-tujuan

- Tulis/eksekusi (WO-03), `MergeVisitors` (WO-04), prompt agent
  (WO-05), perubahan API admin asli (kecuali bila filter baru dipakai
  — tidak, gunakan yang ada).

## Catatan Implementasi (Dev)

(belum ada — isi bertahap per bagian sesuai aturan pinned)

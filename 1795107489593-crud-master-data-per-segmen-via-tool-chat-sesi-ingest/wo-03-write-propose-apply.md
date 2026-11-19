---
title: "WO-03 — Backend segment_write_propose + apply_record_actions + kartu approve chat + Node"
description: "Tulis generik: usulan create/update/delete/merge → proposal store (snapshot sesi) + kartu approve di chat (backend Go pola rejected_listing WO-03) → apply_record_actions confirmed_by_user wajib → eksekusi via API admin asli (RBAC + audit ganda) + dry-run preview + max 200 aksi"
status: pending
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - backend
    - tool
    - crud
---
# WO-03 — Backend: `segment_write_propose` + `apply_record_actions` + kartu approve + Node

Sumber: SPEC `spec.md` topic `1795107489593` + desain `notes.md`.
Prasyarat: WO-02 selesai (whitelist + deidentify hidup — dipakai ulang
di sini). Status: **pending**. Pola ikut WO-03 topic `1795083433686`
(`analysis_apply.go` + kartu `rejected_listing`).

## Lingkup

### A. `POST /admin/ingest/tools/segment_write_propose` (M-2)

1. Input: `{session_id, actions[]}`, `action =
   {kind: create|update|delete|merge, path, method, body?, ids?}`:
   - kind `create`/`update` → `path` + `body`; `delete` → `ids[]`
     (1..200); `merge` → `{survivor_id, merged_ids[]}` (hanya visit
     untuk saat ini — endpoint merge WO-04; segmen lain 409 dulu
     sampai tersedia).
   - **Validasi keberadaan ID** saat propose (GET internal): ID tidak
     ada → 400 (fail-closed, sebelum kartu tampil).
   - method wajib POST/PUT/PATCH/DELETE (GET ditolak); path wajib di
     whitelist tulis segmen (peta WO-02, GET-prefixed path ditolak);
     **TIDAK menerapkan apa pun** (P-3) — disimpan sebagai PROPOSAL
     di snapshot sesi (pola WO-01 topic lama; O-25 persisten) + audit.
2. Proposal ber-ID (`proposal_id`), status `proposed`; muncul sebagai
   **kartu approve `record_actions` di chat** — dibuat backend Go
   (pola `rejected_listing` WO-03; bukan teks LLM), ringkasan per aksi
   (kind, target, delta deidentified) + tombol Setujui.

### B. `POST /admin/ingest/tools/apply_record_actions` (M-3)

1. Input: `{session_id, proposal_id, action_idx[]?}` — subset aksi boleh
   (default semua); `confirmed_by_user: true` WAJIB eksplisit
   (pointer; hilang/false → 400 `ErrApplyNotConfirmed`, pola WO-03
   topic lama).
2. **Dry-run preview** otomatis (W-3) untuk delete/merge SEBELUM
   eksekusi: jumlah + daftar record terdampak (visit yang pindah/
   hilang) — kembalikan preview bila user belum pernah approve aksi
   yang sama (idempotensi per client_id deterministik, pola T-4).
3. **Eksekusi = panggilan API admin asli** (internal service-layer
   call, pola dispatcher `target.go`) dengan admin ID dari token user:
   - `create/update/delete` → endpoint admin segmen yang dipetakan
     (method sesuai kind); RBAC TULIS plugin target wajib (403 tanpa
     `visit.write` dst. — fail-closed).
   - `merge` → endpoint `POST /api/admin/visit/visitors/merge` (WO-04).
4. **Bulk limit** (W-1): max 200 aksi/proposal → 400 di atas itu.
5. **Audit ganda** (W-8): audit endpoint asli tercatat otomatis +
   audit `provider='ingest-tool'` per proposal/aksi (siapa approve,
   isi aksi, hasil).
6. Hasil re-report ke chat (pola pesan sistem O-18) + snapshot
   proposal diperbarui (`applied`/`partially_applied`/gagal per aksi).
7. Guard `ingest.write` + audit ingest-tool (konsisten tool lain).

### C. UI chat (mini)

1. Renderer kartu `record_actions` di `IngestChatPage.tsx` (pola kartu
   `rejected_listing` WO-03 topic lama): daftar aksi + tombol
   "Setujui" per aksi/semua → panggil `apply_record_actions` (API
   backend), PermissionGate `ingest.write`.
2. Tidak ada UI lain (listing panel tidak dipakai di sini).

### D. Node

Skema 2 tool (`segment_write_propose`, `apply_record_actions`) di
`copilot-runtime.ts` — pola existing, Bearer user, tanpa logika bisnis;
deskripsi menegaskan: agent tidak boleh mengisi `confirmed_by_user`
sendiri; setelah propose, tunggu keputusan user.

## Acceptance Criteria

- AC-1 Propose tidak mengubah data master (bukti DB + audit); proposal
  persisten (reload tetap) dengan ID + status.
- AC-2 Kartu approve tampil di chat (backend Go) dengan ringkasan
  deidentified; Setujui per aksi/semua berfungsi.
- AC-3 Apply tanpa `confirmed_by_user` → 400; dengan → eksekusi via
  API asli (bukti data berubah + audit endpoint asli + audit
  ingest-tool).
- AC-4 ID tak ada / method salah / path di luar whitelist → 400
  (sebelum kartu tampil).
- AC-5 RBAC: eksekusi tanpa permission tulis target → 403; bulk >200
  → 400.
- AC-6 Dry-run preview tampil sebelum eksekusi delete/merge (W-3).
- AC-7 Regresi: 4 tool analisa (WO-01 topic lama) + segment_read
  (WO-02) tidak berubah perilaku; `go build/vet/test` + tsc + npm
  build hijau; Bruno; **Log Implementasi (Dev)**.

## Non-tujuan

- `MergeVisitors` internal (WO-04), prompt agent (WO-05), soft delete,
  auto-apply oleh sistem, perubahan state machine/tabel baru.

## Catatan Implementasi (Dev)

(belum ada — isi bertahap per bagian sesuai aturan pinned)

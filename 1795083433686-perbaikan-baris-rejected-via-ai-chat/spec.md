---
title: "SPEC — Perbaikan baris rejected via AI chat (loop + listing)"
description: "SPEC final: loop analisa AI (max 3 putaran) + tabel listing rejected di editor (checkbox scope P-7/P-8 + edit manual) + 4 tool call Node↔Go + CSV perbaikan menggantikan berkas; prinsip fail-closed & deidentify tetap; WO-01..WO-03"
status: progress
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - gate
    - ai-chat
    - spec
---
# SPEC — Perbaikan baris rejected via AI chat

Topic `1795083433686` (extends `1793711295286`). Sumber: desain final
`desain-flow-dan-toolcall.md` (owner ok 2026-11-19). Status:
**progress — disetujui owner 2026-11-19, eksekusi WO-01 dimulai.**

## 1. Masalah

Gate Aturan menolak baris (fail-closed) tapi perbaikan saat ini manual
dan tersebar: user harus membaca pesan gate, menambah rule di editor,
atau membetulkan berkas di luar sistem. Real case: 144 pelanggaran
`visit_visitors.phone_raw.lenient_format` + 12 `phone_e164.required` +
warning `scheduled_at.warn_when_empty` dalam satu berkas uji.

## 2. Solusi (ringkasan)

1. **Snapshot analisa** per sesi: hasil dry-run enforcement disimpan
   persisten ringan (jsonb, plugin ingest — O-25).
2. **Tabel "Baris Rejected"** di editor (mapping draft): checkbox scope
   (P-7/P-8), edit manual inline, tombol "Analisa (N)" + "Setujui".
3. **Agent loop** (max 3 putaran): klasifikasi per kelompok → proposal
   batch (rule dari katalog / row-fix) → `submit_analysis` (T-4).
4. **Apply**: rule → profil append-only; row-fix → CSV perbaikan
   menggantikan berkas sesi (P-1). Re-scan otomatis → loop.
5. **Dua arah chat ↔ listing** via blok terstruktur backend (pola
   `editor_link` WO-04).

## 3. Keputusan terkunci (dari desain)

- P-1 nilai = berkas perbaikan baru (bukan edit staging DB).
- P-2 rule → profil; nilai → storage. P-3 AI hanya mengusulkan.
- P-4 dry run wajib. P-5 deidentify untuk chat & listing.
- P-6 tanpa state baru/tabel baru; snapshot = jsonb plugin ingest.
- P-7 checkbox = scope eksplisit. P-8 default cerdas + larangan
  mengarang nilai field wajib yang kosong (fail-closed Go).
- T-1 `get_rejected_rows` (baca; agent + UI), T-2 `propose_fixes`
  (batch proposal), T-3 `apply_fixes` (`confirmed_by_user` wajib),
  T-4 `submit_analysis` (penutup loop).
- O-21: max 500 baris/petikan; O-22: approve cukup `ingest.write`;
  O-25: snapshot persisten ringan (jsonb), dibersihkan bersama retensi.
- P-9: tombol Analisa menyimpan scope lalu auto-trigger SATU pesan
  basa-basi di chat — detail diambil AI via tool (session id &
  snapshot sudah di sesi); single-shot, tanpa data dalam pesan.

## 4. Lingkup per WO (3)

### WO-01 — Backend: snapshot + tool T-1/T-2/T-4 + fail-closed P-8

- Endpoint analisa: `POST /admin/ingest/sessions/:id/analyze`
  (dry-run full scan → snapshot jsonb; input: row_no scope).
- Snapshot: `ingest_profiles` jsonb baru (`analysis`) atau tabel jsonb
  per sesi (keputusan implementasi dev, tetap tanpa tabel relasional baru).
- Tool endpoint `/admin/ingest/tools/{get_rejected_rows,propose_fixes,
  submit_analysis}` — pola `agent_tools.go` (guard `ingest.write` + audit).
- Fail-closed: rule di luar katalog → 400 `ErrUnknownRule`; row-fix
  field wajib kosong tanpa rule → ditolak; >500 baris/petikan → 400.
- AC-1..AC-6: full scan akurat vs pesan gate; snapshot persisten
  (reload tetap ada); proposal tidak menerapkan apa pun; audit lengkap;
  build/vet/test hijau; Bruno.

### WO-02 — Frontend: listing UI + checkbox + edit manual + tombol analisa

- Panel "Baris Rejected" di `IngestMappingPage` (katalog field, Violates,
  usulan AI, status, checkbox default dari flag P-8 backend).
- Tombol "Analisa (N)" = simpan scope (analyze) → tutup editor →
  auto-submit pesan basa-basi di chat (P-9; detail diambil AI via
  tool), "Setujui" per baris/kategori/semua, edit inline
  (`edited_by: user`), tombol "perbaiki via chat" (link kontekstual).
- AC-1..AC-6: listing tampil + persisten; checkbox mengontrol scope;
  edit manual masuk proposal; tombol Analisa memicu agent; RBAC; build
  hijau; Playwright smoke Tailscale.

### WO-03 — Apply + berkas perbaikan + re-scan loop + blok chat

- `apply_fixes` (T-3): rule → profil (append-only, StatedBy); row-fix →
  CSV perbaikan → replace berkas sesi; re-scan otomatis (dry run) →
  snapshot ter-update.
- Blok terstruktur `rejected_listing` di chat (backend Go, pola
  `editor_link` WO-04) + pesan sistem hasil analisa (pola O-18/summary).
- Node: registrasi skema 4 tool (`copilot-runtime.ts`, tanpa logika).
- AC-1..AC-8: apply dari listing & chat; CSV perbaikan terbukti mengganti
  berkas; re-scan mengurangi pelanggaran; loop berhenti ≤3 putaran;
  regresi visit/resident/sec/hsh; Bruno end-to-end; log implementasi.

## 5. Non-tujuan

- Perubahan state machine / tabel relasional baru / katalog rule baru.
- Auto-apply oleh sistem; AI mengarang rule/nilai field wajib kosong.
- Retensi snapshot lanjutan (menunggu WO-10 topic induk).
- Fitur listing untuk segmen lain di luar mesin enforcement existing
  (otomatis ikut karena mesinnya generik — hanya diuji).

## 6. Risiko

- LLM bisa salah klasifikasi massal → mitigasi: proposal perlu approve,
  dry run wajib, remaining ditandai "perlu manusia" (T-4).
- Ukuran snapshot besar pada berkas 1.6k baris → mitigasi: snapshot per
  pelanggaran (bukan per baris mentah), kompres kelompok per rule id.
- PII pada listing → mitigasi P-5 deidentify konsisten dengan sample
  editor (satu jalur, `BuildSafePayload`/deidentify Go).

## Rencana bukti (per WO)

- WO-01: Bruno tools + snapshot persisten + audit rows.
- WO-02: Playwright (Tailscale) listing + checkbox + edit + Analisa.
- WO-03: uji berkas asli (144 lenient) end-to-end + regresi 4 segmen.

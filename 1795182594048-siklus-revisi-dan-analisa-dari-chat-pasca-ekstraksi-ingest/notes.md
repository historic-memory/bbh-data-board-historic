---
title: "Notes — Siklus revisi & analisa dari chat pasca ekstraksi ingest"
description: "Riset & bukti kasus sesi 873db24a / d1f3d30b (2026-11-20)"
status: progress
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - notes
---
# Notes — Siklus revisi & analisa dari chat pasca ekstraksi ingest

## Bukti investigasi (2026-11-20)

1. **Runtime chat punya 11 tool** — TIDAK ADA tool transisi state dan
   TIDAK ADA tool pemulai analisa:
   `sandbox_ping, get_mapping_guess, confirm_mapping, run_extraction,
   get_rejected_rows, propose_fixes, submit_analysis, apply_fixes,
   segment_read, segment_write_propose, apply_record_actions,
   introduce_session`.
2. **State machine sehat** (`model/session.go:60-65`):
   `revise: {From: [mapping_confirmed, extracted], To: mapping_draft}`.
   Uji API langsung dua sesi berbeda: `extracted → mapping_draft`
   berhasil (09:25 dan 22:0x). Tidak ada deadlock.
3. **Tidak ada kolom `mapping_confirmed`** di `ingest_sessions` —
   klaim "flag tidak sinkron" halusinasi.
4. **Endpoint transisi** `POST /admin/ingest/sessions/:id/transition`
   (`session_state.go`) sudah ada & teruji; `allowed_actions` ekspos
   `revise` saat `extracted`.
5. **Endpoint analisa** `POST /admin/ingest/sessions/:id/analyze`
   sudah ada (dry-run penuh + snapshot persisten) tapi tidak diekspos
   sebagai tool.
6. **Agent chat berulang kali salah putih**: menyimpulkan "deterministik
   gagal" tanpa cek state terkini, membuat laporan bug palsu, dan
   menyarankan sesi baru padahal revise cukup. → perlu Kontrak baru di
   prompt.
7. **UI tidak menawarkan revise** — grep frontend: 0 referensi
   `revise`; tombol hanya "muat ulang" + "Analisa (N)".
8. NIK berkas resident rusak oleh Excel (notasi ilmiah, digit hilang) —
   bukan pekerjaan topic ini (backlog rule/template terpisah).

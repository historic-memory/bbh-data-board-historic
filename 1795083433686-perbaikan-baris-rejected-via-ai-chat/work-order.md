---
title: "Work order — Perbaikan baris rejected via AI chat"
description: "Master index WO topic 1795083433686: 3 WO (backend snapshot+tool → listing UI → apply+loop+chat); SPEC progress (owner ok 2026-11-19); WO-01 complete, WO-02/03 pending"
status: progress
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - gate
    - ai-chat
    - work-order
---
# Work order — Perbaikan baris rejected via AI chat

Topic `1795083433686` (extends `1793711295286`). Sumber kebenaran desain:
`desain-flow-dan-toolcall.md` (final, owner ok 2026-11-19) + `spec.md`
(progress — owner setujui 2026-11-19). **WO-01 SELESAI (complete);
WO-02/03 pending** sampai prasyaratnya selesai.

## Latar

Owner menemui gate rejected saat uji ingest berkas asli (144
`phone_raw.lenient_format` + 12 `phone_e164.required`). Ide: listing
rejected jadi tabel interaktif di editor + AI yang mengerjakan perbaikan
via loop (vibecoding), dengan gate fail-closed tetap. Desain dirampokkan
melalui beberapa iterasi owner (2026-11-19): tombol analisa tunggal →
listing interaktif dua arah → checkbox scope → final ok.

## Daftar WO

| WO | Judul | Status | Commit kode | Dok historic |
|---|---|---|---|---|
| WO-01 | Backend snapshot analisa + tool T-1/T-2/T-4 + fail-closed P-8 | **complete** (2026-11-19) | `fffe34c` | wo-01-backend-snapshot-tool.md |
| WO-02 | Listing UI: tabel rejected + checkbox + edit manual + tombol Analisa | pending (siap: WO-01 complete) | — | wo-02-listing-ui.md |
| WO-03 | Apply fixes + CSV perbaikan + re-scan loop + blok chat | pending (menunggu WO-01 & 02) | — | wo-03-apply-rescan-loop.md |

## Prinsip lintas WO (pinned)

1. **Fail-closed** di Go: rule hanya dari katalog `exceptions.yaml`;
   AI tidak pernah menulis — semua via API Go (RBAC `ingest.write`
   + audit `provider='ingest-tool'`).
2. **Deidentify** untuk chat & listing (P-5) — satu jalur deidentify.
3. **Dry run wajib** di evaluasi ulang (P-4); nilai = **CSV perbaikan
   menggantikan berkas** (P-1, bukan edit staging DB).
4. Max 3 putaran analisa; max 500 baris/petikan (O-21); approve cukup
   `ingest.write` (O-22); snapshot persisten ringan (O-25).
5. **Log Implementasi (Dev)** wajib per WO (aturan pinned topic induk).
6. Kode fitur hanya di `plugins/ingest/` + UI
   `sub_app/webapp/src/pages/admin/ingest/` + Node
   `sub_app/ingest_runtime/` — tanpa state machine/tabel relasional baru.
7. Tombol Analisa = simpan scope ke snapshot (analyze) → tutup editor
   → auto-trigger SATU pesan basa-basi di chat (P-9); detail diambil
   AI via tool, bukan dari isi pesan.

## Urutan eksekusi

WO-01 → WO-02 → WO-03 (prasyarat bertingkat). Kasus uji akhir: berkas
asli owner end-to-end (AC-6 WO-03).

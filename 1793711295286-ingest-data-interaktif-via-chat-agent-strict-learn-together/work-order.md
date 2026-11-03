---
title: work order
description: "Master index WO plugin ingest — pendekatan checkpoint (partial: CP-1 saja; WO checkpoint berikutnya menyusul)"
status: draft
created: "2026-11-03"
---
# Master Index — Work Order Plugin `ingest` (checkpoint)

Sumber desain: `spec.md` (D-1..D-9 terkunci, pendekatan checkpoint bagian 9) +
`spec-techstack.md` (baseline & dependensi baru).

**Pendekatan (keputusan owner):** WO TIDAK langsung on-target. Empat checkpoint;
setiap checkpoint punya milestone yang bisa dites sebelum lanjut. File WO dibuat
**partial** — hanya checkpoint berikutnya yang dirinci; WO checkpoint selanjutnya
baru dibuat setelah checkpoint sebelumnya selesai (agar pembelajaran dari tahap
sebelumnya masuk ke rancangan).

| Checkpoint | Isi | Status | Bukti |
|---|---|---|---|
| CP-1 | Setup CopilotKit + proxy + sandbox belajar | `progress` (WO-01) | — |
| CP-2 | Real case pertama `visit` via chat (tanpa editor) | dibuat setelah CP-1 | — |
| CP-3 | Excel mapping editor (berdiri sendiri) | dibuat setelah CP-2 | — |
| CP-4 | Integrasi editor ↔ CopilotKit | dibuat setelah CP-3 | — |

## Daftar WO aktif

| WO | Judul | Status | Commit kode | Dok historic |
|---|---|---|---|---|
| WO-01 | Setup CopilotKit + proxy LLM + sandbox belajar | pending | — | — |
| WO-02..(CP-2 dst) | dirinci SETELAH CP-1 selesai | — | — | — |

**Aturan lintas WO (pinned):**

- Kode fitur hanya di `plugins/ingest/` + UI di `sub_app/webapp/src/pages/admin/ingest/`;
  migrasi permission di plugin `auth` (fail-closed).
- Strictness = state machine backend + system prompt agent; CopilotKit hanya lapisan
  UI; tool-calling klien tidak pernah menulis langsung.
- API key LLM tidak pernah keluar dari backend; proxy menolak payload PII yang belum
  dideidentifikasi (fail-closed).
- Dependensi baru wajib direkam di `spec-techstack.md` (saat ini: `deidentify` di Go,
  `openai`-compat client / CopilotKit di frontend).

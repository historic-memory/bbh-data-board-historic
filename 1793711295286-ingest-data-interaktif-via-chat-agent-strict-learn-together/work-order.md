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
sebelumnya masuk ke rancangan). **Pengecualian (keputusan owner, 2026-11-04):**
CP-1 dipecah menjadi tiga sub-WO (WO-01a/b/c) yang semuanya dirinci sekarang,
karena CP-1 adalah checkpoint belajar tanpa data bisnis — risikonya rendah dan
milestone-nya sudah jelas sejak awal.

| Checkpoint | Isi | Status | Bukti |
|---|---|---|---|
| CP-1 | Setup CopilotKit + proxy + sandbox belajar — **dipecah 3 sub-WO**: 01a proxy+chat → 01b tool+upload → 01c MCP+housekeeping | `progress` (WO-01a→01b→01c) | — |
| CP-2 | Real case pertama `visit` via chat (tanpa editor) | dibuat setelah CP-1 | — |
| CP-3 | Excel mapping editor (berdiri sendiri) | dibuat setelah CP-2 | — |
| CP-4 | Integrasi editor ↔ CopilotKit | dibuat setelah CP-3 | — |

## Daftar WO aktif

| WO | Judul | Status | Commit kode | Dok historic |
|---|---|---|---|---|
| WO-01a | Proxy LLM + CopilotKit chat hidup (CP-1) | pending | — | — |
| WO-01b | Tool dummy `sandbox_ping` + upload ke storage (CP-1) | pending | — | — |
| WO-01c | Eksperimen MCP + housekeeping (CP-1) | pending | — | — |
| WO-02 | Sesi strict + real case `visit` via chat (tanpa editor) | pending | — | — |
| WO-03 | Excel mapping editor (berdiri sendiri) | pending | — | — |
| WO-04 | Integrasi editor ↔ CopilotKit + housekeeping (CP-4 final) | pending | — | — |
| WO-05.. | tidak ada — CP-4 adalah checkpoint terakhir | — | — | — |

**Aturan lintas WO (pinned):**

- Kode fitur hanya di `plugins/ingest/` + UI di `sub_app/webapp/src/pages/admin/ingest/`;
  migrasi permission di plugin `auth` (fail-closed).
- Strictness = state machine backend + system prompt agent; CopilotKit hanya lapisan
  UI; tool-calling klien tidak pernah menulis langsung.
- API key LLM tidak pernah keluar dari backend; proxy menolak payload PII yang belum
  dideidentifikasi (fail-closed).
- Dependensi baru wajib direkam di `spec-techstack.md` (saat ini: `deidentify` di Go,
  `openai`-compat client / CopilotKit di frontend).

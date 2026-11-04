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
milestone-nya sudah jelas sejak awal. **Pengecualian kedua (2026-11-04):** CP-2
ditutup sebagian — inti (0/A/B/C/D + compact) selesai & teruji dengan data
sintetis; sisa AC-4 (penegakan rules), AC-8 (idempotensi re-impor), dan uji
berkas asli BBH dibawa ke CP-3/WO-03 sebagai prasyarat verifikasi, karena
butuh berkas asli dari owner & keputusan desain rules.

| Checkpoint | Isi | Status | Bukti |
|---|---|---|---|
| CP-1 | Setup CopilotKit + proxy + sandbox belajar — **dipecah 3 sub-WO**: 01a proxy+chat → 01b tool+upload → 01c MCP+housekeeping | **complete** (2026-11-04) | e4f4ea9, a019c26, a22c03b, f00e542, 397fbff |
| CP-2 | Real case pertama `visit` via chat (tanpa editor) | **complete (sebagian, 2026-11-04)** — inti selesai; sisa: uji berkas asli BBH + AC-8 + AC-4 | 032941e |
| CP-3 | Excel mapping editor (berdiri sendiri) + rules O-20 + prasyarat verifikasi sisa CP-2 | TERBUKA — WO-03 rancangan final, siap eksekusi | — |
| CP-4 | Integrasi editor ↔ CopilotKit | dibuat setelah CP-3 | — |

## Daftar WO aktif

| WO | Judul | Status | Commit kode | Dok historic |
|---|---|---|---|---|
| WO-01a | CopilotKit chat hidup (CP-1) — **DIREVISI D-11** | complete (2026-11-04) | e4f4ea9, a019c26, a22c03b | — |
| WO-01b | Tool dummy `sandbox_ping` + upload ke storage (CP-1) | complete (2026-11-04) | f00e542 | — |
| WO-01c | Eksperimen MCP + housekeeping (CP-1) | complete (2026-11-04) — CP-1 DITUTUP | 397fbff | a017ba2 |
| WO-02 | Sesi strict + real case `visit` via chat (tanpa editor) — **DIREVISI pasca CP-1** | complete (sebagian, 2026-11-04) — sisa AC-4/AC-8 + uji berkas asli | 032941e |
| WO-03 | Excel mapping editor (berdiri sendiri) — §0 prasyarat sisa CP-2 + §D rules O-20 | pending (rancangan final) | — | — |
| WO-04 | Integrasi editor ↔ CopilotKit + housekeeping (CP-4 final) | pending | — | — |
| WO-05.. | tidak ada — CP-4 adalah checkpoint terakhir | — | — | — |

**Aturan lintas WO (pinned):**

- Kode fitur hanya di `plugins/ingest/` + UI di `sub_app/webapp/src/pages/admin/ingest/`;
  migrasi permission di plugin `auth` (fail-closed).
- **(D-11 final, 2026-11-04)** Runtime Node CopilotKit = **backend chat AI penuh**
  (satu-satunya pemanggil provider LLM; key hanya di Node); **Go = data service** —
  semua keputusan bertanggung jawab tetap di Go: tool calls + state machine +
  deidentify (Node meminta payload aman dari Go) + audit (`ingest_audit` pusat).
  Node TIDAK pernah menulis profil. N-17 dilonggarkan khusus service
  `ingest-runtime`. Proxy Go OpenAI-compatible keluar dari jalur chat (nasib akhir
  diputuskan di WO-01c).
- Strictness = state machine backend Go + system prompt agent; CopilotKit hanya
  lapisan UI + transport; tool-calling klien maupun Node tidak pernah menulis profil
  langsung — semua lewat action API Go yang di-guard RBAC.
- Dependensi baru wajib direkam di `spec-techstack.md` (kini: `deidentify` di Go,
  CopilotKit `react-core`+`runtime` di frontend/Node, service `ingest-runtime`).

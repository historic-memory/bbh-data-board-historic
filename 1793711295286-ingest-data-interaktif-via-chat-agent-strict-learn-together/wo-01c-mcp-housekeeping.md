---
title: WO-01c — Eksperimen MCP + housekeeping (CP-1, sub 3/3)
description: "Sub-checkpoint CP-1: eksperimen MCP di luar jalur utama, catat temuan, rapikan dokumen & keputusan lanjut sebelum CP-2."
status: pending
created: "2026-11-03"
---
# WO-01c — Eksperimen MCP + Housekeeping (Sub-Checkpoint CP-1, 3/3)

Induk: `wo-01-setup-copilotkit-sandbox.md` (dipecah 2026-11-04). Urutan: 1a → 1b
→ **1c** (terakhir di CP-1). **Prasyarat: WO-01b lulus AC-nya.**

## Tujuan

Menutup CP-1: eksperimen MCP (keputusan tetap **eksploratif**, tidak jadi standar
arsitektur), merekam semua temuan belajar, dan merapikan dokumen sebelum masuk
CP-2 (real case `visit`).

## Lingkup

### 1. Eksperimen MCP (di luar jalur utama)

- Pasang SATU MCP server kecil ke CopilotKit di sandbox; catat di notes topic:
  - kompatibilitas dengan proxy OpenAI-compatible + CopilotKit runtime;
  - effort setup vs tool backend biasa (WO-01b) — perbandingan nyata;
  - risiko (jalur auth, audit, guard) bila MCP dipakai serius.
- **Eksperimen TIDAK menjadi standar**; keputusan apakah MCP dipakai di CP-2+
  direkam sebagai keputusan di notes (default: tidak — tool backend saja).

### 2. Housekeeping penutup CP-1

- Rekaman effort riil adaptasi CopilotKit di proxy (dari WO-01a) + pola tool (dari
  WO-01b) — masukan langsung untuk rancangan CP-4.
- Rapikan swagger & Bruno untuk semua endpoint sandbox; bersihkan kode eksperimen
  yang gagal/tidak dipakai (biarkan yang hidup).
- Checklist readiness CP-2 di notes: env `INGEST_LLM_*` aman, permission seed
  benar, tidak ada data bisnis yang pernah lewat sandbox.
- Perbarui status WO-01a/b/c + CP-1 di master index (`work-order.md`) dengan bukti.

## Acceptance Criteria

- AC-7 Temuan eksperimen MCP tercatat di notes topic (kompatibilitas, effort, risiko).
- AC-8 Keputusan lanjut soal MCP terekam; `go vet`/`go test` hijau; `npm run build`
  hijau; dokumen & status checkpoint diperbarui.

## Non-tujuan

- Semua hal CP-2 (sesi strict, state machine, deidentify nyata, ekstraksi `visit`)
  — baru mulai setelah CP-1 ditutup lewat checkpoint ini.

## Risiko

- Jangan habiskan waktu berlebihan di MCP — batasi eksperimen ini ke satu server
  kecil + catatan; nilai utamanya adalah KEPUTUSAN yang terekam, bukan fiturnya.

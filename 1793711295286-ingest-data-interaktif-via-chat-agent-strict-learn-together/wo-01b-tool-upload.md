---
title: WO-01b — Tool dummy sandbox_ping + upload storage (CP-1, sub 2/3)
description: "Sub-checkpoint CP-1: pola tool-calling (tool = API backend) via tool dummy, dan upload file ke storage tanpa diproses."
status: pending
created: "2026-11-03"
---
# WO-01b — Tool Dummy `sandbox_ping` + Upload ke Storage (Sub-Checkpoint CP-1, 2/3)

Induk: `wo-01-setup-copilotkit-sandbox.md` (dipecah 2026-11-04). Urutan: 1a →
**1b** → 1c. **Prasyarat: WO-01a lulus AC-nya** (chat streaming hidup, key aman,
fail-closed terbukti).

## Tujuan

Membuktikan dua pola interaktivitas dasar yang akan dipakai semua checkpoint
berikutnya: (1) **tool = API backend** yang dipanggil agent via tool-calling, dan
(2) file yang di-upload via chat tersimpan aman di storage. Masih sandbox — tanpa
data bisnis, tanpa pemrosesan file.

## Lingkup

### 1. Tool dummy `sandbox_ping`

- Satu tool echo sederhana: agent memanggil `sandbox_ping` via tool-calling CopilotKit
  → dieksekusi di **backend** (guard RBAC `ingest.write`, audit) → hasil tampil di chat.
- Tool didaftarkan di sisi backend (bukan didefinisikan di klien) — pola yang akan
  dipakai semua tool nyata di CP-2 (mis. `get_mapping_guess`, `confirm_mapping`).
- Catat di notes: bagaimana CopilotKit melewatkan tool-call ke backend (mechanism,
  latency, kegagalan).

### 2. Upload file via chat

- Komponen upload di panel chat: file masuk `internal/storage`, tercatat di DB
  (tabel/metadat minimal), **TIDAK diproses** — hanya nama + ukuran tampil di chat.
- Batas ukuran/ekstensi ditolak fail-closed (rebut pola `.xls` N-17 sejak sekarang
  untuk membiasakan perilakunya).
- Guard RBAC + audit sama seperti proxy.

## Acceptance Criteria

- AC-4 Tool dummy `sandbox_ping` terpanggil dari chat dan hasilnya tampil di chat.
- AC-5 Upload file via chat tersimpan di storage + tercatat; tidak ada proses lain.
- AC-6 Tanpa token (401) / tanpa `ingest.write` (403) → pemanggilan tool & upload ditolak.
- AC-8 `go vet`/`go test` hijau; `npm run build` hijau; swagger berisi endpoint tool
  & upload; temuan pola tool-calling dicatat di notes topic.

## Non-tujuan

- Pemrosesan isi file apa pun (parsen Excel/CSV — CP-2); MCP (WO-01c); sesi strict/
  state machine/deidentify/ekstraksi/editor (CP-2/3/4).

## Risiko

- Jangan biarkan tool dummy menyimpan/mengembalikan apa pun selain echo — fondasi
  konvensi tool (nama, guard, audit) yang akan disalin tool nyata.

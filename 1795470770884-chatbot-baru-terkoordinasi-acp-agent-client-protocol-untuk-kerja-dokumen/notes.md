---
title: notes
description: "Kronologi keputusan & langkah berikutnya topic chatbot ACP."
status: progress
created: "2026-11-24"
updated: "2026-11-24"
tags: [notes]
---

# Catatan

## Kronologi
- 2026-11-23: riset ACP terbentuk (topic 1795469702082) — dua makna ACP dijernihkan.
- 2026-11-24 (pagi): owner mengusulkan agent umum ala OpenCode sebagai manipulator dokumen ingest ("1 paket"). Tercatat sebagai alternatif desain di topic sandbox 1795414203062 (save 477bbe1) + donis-agent deterministik (save 07d7dbf).
- 2026-11-24: owner memutuskan: (a) hapus/clear "fitur deterministik" dari bbh-data-board; (b) buat chatbot BARU terkoordinasi ACP tanpa menyentuh ingest chat eksisting; (c) dulu DICATAT saja, lalu riset alternatif agent selain OpenCode.
- Audit pembersihan dilakukan Manager sebelum eksekusi: fitur "deterministik" praktis tidak ada di repo (classifier/Jev tak pernah dikodekan; teks deterministik milik provider donis-agent). Yang tak boleh ikut terhapus: deidentify HMAC (PII) + pipeline deterministik fail-closed. Pembersihan cukup mencabut komentar/referensi usang — DIUNDUR ke housekeeping topic ini.

## Riset alternatif agent selain OpenCode (2026-11-24) — dari daftar resmi ACP
Sumber: agentclientprotocol.com/overview/agents (daftar agent ACP resmi, dipotong ke kandidat paling relevan untuk server kami, BUKAN IDE koding):

### Kelas 1 — CLI agent "1 paket" yang bisa jalan headless di server (kandidat utama)
| Agent | Buatan | ACP | Catatan |
|---|---|---|---|
| **OpenCode** (baseline pembanding) | sst | asli | Baseline intuisi owner; Node/TS, tools lengkap |
| **Gemini CLI** | Google | asli | Open-source (Apache-2.0), matang, tapi provider LLM default = Gemini (PII keluar kecuali set custom endpoint) |
| **Codex CLI** | OpenAI | via adapter `codex-acp` | Ada adapter resmi; provider default OpenAI |
| **Claude Agent** (Claude Code) | Anthropic | via adapter `claude-agent-acp` (Zed) | Adapter SDK resmi Zed; provider default Anthropic |
| **Qwen Code** | QwenLM | asli | Fork Gemini CLI; bisa diarahkan ke OpenAI-compatible API (cocok donis-agent?) |
| **Kimi CLI** | Moonshot | asli | Moonshot; provider Kimi |
| **Goose** | Block (Apache-2.0) | asli | Ekstensibel (Rust/TS), provider configurable (OpenAI-compatible) — kandidat menarik untuk donis-agent |
| **OpenHands** | All Hands AI | asli | Berat (docker runtime sendiri); kemungkinan overkill |
| **Cline** | Cline Bot | asli | Berasal VS Code extension; headless? perlu cek |
| **GitHub Copilot CLI** | Microsoft | public preview | Preview resmi; bergantung akun Copilot |
| **Docker cagent** | Docker | asli | Deklaratif YAML; provider configurable |

### Kelas 2 — adapter/gateway yang memungkinkan agent SENDIRI via ACP (paling menarik arsitektur)
- **acp_rpc_bridge** (Intellexie): stdio ↔ HTTP — memberi kemampuan jaringan ke agent non-network.
- **ACP to AG-UI** (`acp-to-agui`): jembatan agent ACP → **CopilotKit/AG-UI** (SSE) — bisa berarti UI chatbot baru kita tetap pakai pola CopilotKit yang sudah dikuasai tim, tanpa menulis ACP client dari nol.
- **Open Agent API**: gateway self-hosted — jalan ACP v1 agent → OpenAI-compatible HTTP API (ada approval permission request owner).
- **a2acode**: ACP agent ⇄ A2A — menjembatani dengan standar antar-agent.
- **Remote ACP Bridge**: website bisa kirim task ke agent lokal via outbound-only relay, per-site consent, project-folder firewall.
- **chips-acp / acpantenna / acp-connector dkk.**: kelas "client siap-pakai" — bukan yang kita cari (kita = client sendiri).

### Observasi arsitektural awal (belum dikunci)
1. Ekosistem ACP jauh lebih matang dari dugaan — 50+ agent, 100+ client/connector. Banyak jalan: (a) pakai agent CLI siap-pakai, (b) bungkus agent sendiri lewat SDK ACP (TS/Rust/Python), (c) pakai jembatan AG-UI agar UI tetap CopilotKit.
2. Pertanyaan kunci baru: **"agent 1 paket" vs "agent kita sendiri ber-ACP"** — agent sendiri (opsi b/c) lebih mudah dikontrol (no-net by design, tools whitelist kita sendiri) dan lebih selaras D-11; agent CLI siap-pakai lebih cepat tapi hardening lebih sulit.
3. **donis-agent deterministik** bisa masuk via kelas 1 provider OpenAI-compatible (Goose/Qwen/Docker cagent) atau kelas 2 (agent sendiri memakai donis-agent sebagai LLM provider).
## Riset Goose sebagai pilihan agent (2026-11-24, follow-up owner)
Owner condong ke **Goose** (Block, Apache-2.0, Linux Foundation/AAIF; 56.9k★; v1.60.2; Rust). Fakta kunci dari repo/dok resmi:
- **Dukungan ACP resmi built-in**: `goose acp` = goose sebagai agent subprocess di belakang aplikasi kita ("your app sends prompts and receives streamed responses, tool activity, and permission requests"). Persis pola yang kita butuhkan.
- **85+ provider LLM configurable** (OpenAI-compatible included) → jalur memakai **donis-agent** milik owner sebagai provider mungkin.
- **oi-kit** (Open Intelligence Kit): runtime Rust yang bisa dikomposisi (state machine, approvals, tools, context management, local model in-process) — kelas "agent kita sendiri ber-ACP" juga terbuka lewat oi-kit bila suatu saat perlu kontrol lebih dalam.
- Ada Dockerfile resmi di repo (docs BUILDING_DOCKER) — container-izable.
- **Jawaban pertanyaan owner "setup container isolasi ketat?"**: YA — itu justru prasyarat desain. Rencana isolation minimal: (1) container khusus `goose` terpisah dari `app`/`ingest-runtime`; (2) network internal/egress deny kecuali LLM provider via proxy whitelist; (3) volume kerja per-sesi (scratch) hanya berkas snapshot read-only; (4) non-root user, cap memori/CPU, timeout; (5) permission prompt ACP (`session/request_permission`) dipakai sebagai gerbang approve/deny — diterjemahkan jadi kartu konfirmasi di UI chatbot baru; (6) keluaran kerja = usulan/diff, penyimpanan tetap lewat validasi Go (fail-closed pola ingest).
- PERHATIAN: memilih goose = keputusan O-1 diarahkan, tapi belum KUNCI final sampai: O-2 (container terpisah vs menumpang ingest_runtime — rekomendasi Manager: container terpisah) dan O-3 (provider LLM: donis-agent vs eksternal) diputus. Catat sebagai usulan finalis.



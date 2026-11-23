---
title: spec
description: "Riset dua protokol bernama ACP: Agent Communication Protocol (BeeAI, →A2A) dan Agent Client Protocol (Zed/OpenCode). Riset saja, tanpa implementasi."
status: progress
created: "2026-11-23"
updated: "2026-11-24"
tags: [riset, agent, acp, a2a, protokol]
related: ["1795414203062"]
---

# SPEC — Riset ACP/A2A untuk Agent-Platform

## Tujuan
Memahami protokol-protokol "ACP" yang dipakai platform agent & agent koding, dan menilai relevansinya untuk pola "chatbot bikin aplikasi di atas website" di project kita.

## Peringatan Disambiguasi (PENTING)
Istilah "ACP" dipakai untuk **dua protokol berbeda**:

| | ACP — Agent Communication Protocol | ACP — Agent Client Protocol |
|---|---|---|
| Buatan | BeeAI/IBM → Linux Foundation (digabung ke **A2A**) | Zed → agentclientprotocol.com (dipakai OpenCode) |
| Fungsi | Komunikasi antar-agent & agent↔aplikasi di server (REST) | **Editor/CLI ↔ agent koding** (JSON-RPC over stdio; HTTP/WebSocket remote — masih WIP) |
| Analogi | "HTTP untuk agent" | "LSP untuk coding agent" |
| Kasus | Platform web app-builder, multi-agent | Agent koding di terminal/IDE (OpenCode, Zed, dsb.) |

## Temuan 1 — Agent Communication Protocol (riset 2026-11-23/24)
- Konsep inti: Agent Manifest (discovery), Run (sync/streaming/background), Message multimodal, **Await** (agent pause & minta input user), Sessions (state lintas interaksi).
- REST API biasa (bukan JSON-RPC); SDK Python/TS; bisa curl.
- **Status: repo i-am-bee/acp diarsipkan Agustus 2025 — digabung ke A2A (Linux Foundation). Standar aktif: A2A; Agent Stack (i-am-bee) membangun di atasnya.**
- Pola platform app-builder: Chat UI → agent runtime (A2A/ACP) → agent app-builder → tools (sandbox exec, deploy, DB) + Await untuk konfirmasi user.
- Pemetaan ke bbh-data-board: CopilotKit (8200) ≈ Run/Message; kartu propose_fixes+apply_fixes ≈ Await; 12+ tool ingest ≈ tools. Yang belum ada: manifest/discovery & sandbox exec (lihat topic 1795414203062).

## Temuan 2 — Agent Client Protocol / "OpenCode ACP" (riset 2026-11-24)
- Sumber: agentclientprotocol.com + repo agentclientprotocol/agent-client-protocol (4.7k★, aktif; protokol versi stabil `1`, schema v1/v2).
- Tujuan: **standarisasi komunikasi editor ↔ coding agent** (analogi LSP). Editor yang mendukung ACP bisa pakai agent ACP mana pun; agent ACP bisa dipakai editor ACP mana pun (OpenCode, Zed, dll).
- Transport: JSON-RPC **over stdio** (agent = subprocess editor); remote (HTTP/WebSocket) sedang dalam pengembangan bersama platform agentic.
- Reuses tipe JSON MCP di mana mungkin; output user-facing = Markdown; ada tipe khusus UX agentic (mis. render diff).
- SDK: Rust (official), TypeScript, Python, Kotlin, Java.
- **Relevansi ke pertanyaan user (2026-11-24): OpenCode ACP = Agent Client Protocol → tidak diperlukan untuk pola "chatbot bikin aplikasi di web".** Itu protokol workflow koding di editor/terminal, bukan infrastruktur agent server-side. Untuk platform web, jalur relevan tetap CopilotKit (kita) atau A2A (standar terbuka).
- Konsekuensi riset: setiap klaim "ACP" dari platform luar harus dicek dulu ACP yang mana (Communication vs Client) — jangan berasumsi.

## Kesimpulan Sementara
- Kebutuhan kita saat ini: **tidak ada protokol baru yang wajib diadopsi**. CopilotKit sudah menjalankan peran chat↔agent; A2A baru menarik kalau mau multi-agent lintas project/framework.
- Agent Client Protocol relevan hanya jika suatu saat kita membangun editor-agent sendiri atau ingin agent koding lokal (OpenCode dsb.) bekerja di atas protokol standar — itu ranah dev-tool, bukan produk web.

## Batasan
- HANYA riset — tanpa implementasi, tanpa mengubah source code, tanpa WO teknis.
- Sandbox ingest (topic 1795414203062) tetap fondasi prioritas.

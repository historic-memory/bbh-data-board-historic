---
title: notes
description: "Sumber & konteks riset ACP/A2A + klarifikasi OpenCode ACP."
status: progress
created: "2026-11-23"
updated: "2026-11-24"
tags: [notes]
---

# Catatan

## Sumber
- Agent Communication Protocol: agentcommunicationprotocol.dev, github.com/i-am-bee/acp (arsip), github.com/i-am-bee/agentstack (A2A).
- Agent Client Protocol ("OpenCode ACP"): agentclientprotocol.com, github.com/agentclientprotocol/agent-client-protocol.
- DeepLearning.AI short course "ACP: Agent Communication Protocol" tersedia.

## Kronologi pertanyaan user
- 2026-11-23: user menemukan platform "chatbot bikin aplikasi di atas website" yang klaim pakai ACP → dibuat topic riset ini (awalnya diasumsikan Agent Communication Protocol).
- 2026-11-24: user mengklarifikasi — yang dimaksud adalah **OpenCode ACP** = Agent Client Protocol (Zed). Temuan: itu protokol editor↔agent koding, BUKAN protokol platform web. Pertanyaan "berarti kalau pakai OpenCode ACP enggak perlu ya?" → jawaban: benar, tidak diperlukan untuk pola app-builder web; jalur relevan tetap CopilotKit/A2A.
- Pelajaran: istilah "ACP" ambigu — dua protokol berbeda. Cek dulu maknanya sebelum riset lanjutan.

## Bukti verifikasi
- agentclientprotocol.com: "standardizes communication between code editors/IDEs and coding agents"; JSON-RPC over stdio; remote WIP; protocolVersion=1.
- github.com/i-am-bee/acp: banner "ACP is now part of A2A under the Linux Foundation" (arsip Agustus 2025).

## Meeting 2026-11-24 — usulan user
User merasa dengan "ACP ini" agent lebih mudah memanipulasi dokumen karena "1 paket: sandbox, tools, dll" (rujukan: OpenCode).
- Manager: yang menyediakan paket itu **agent OpenCode-nya** (built-in tools: file edit, shell, code exec) — bukan protokolnya. Protokol ACP (Client) hanya standar koneksi editor↔agent.
- Intuisi valid: inilah kekuatan agent umum (ala ChatGPT Data Analyst). Risiko & jalan tengah "proposer dalam sandbox ketat" tercatat di topic sandbox `1795414203062` (notes.md, bagian Alternatif desain) — keputusan desain di sana, bukan di riset ini.

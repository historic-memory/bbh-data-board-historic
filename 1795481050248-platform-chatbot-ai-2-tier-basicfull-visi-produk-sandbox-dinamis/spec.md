---
title: spec
description: "Visi produk platform chatbot AI 2-tier: Basic (chatbot + tools basic, tanpa sandbox) dan Full (sandbox dinamis ketat ala Goose — runner persistent + launcher spawn per-sesi + config provider dari DB) — dipakai sebagai fondasi integrasi AI untuk project client."
status: draft
created: "2026-11-24"
updated: "2026-11-24"
tags: [visi, produk, platform, chatbot, sandbox, client]
related: ["1795470770884", "1795414203062", "1795469702082"]
---

# SPEC — Platform Chatbot AI 2-tier (Basic/Full)

## Status
VISI PRODUK — keputusan arah dari owner (2026-11-24). Belum ada WO teknis; membaca WO topic terkait.

## Konteks
Lahir dari diskusi topic `1795470770884` (chatbot ACP Goose) saat WO-CP1 revisi 2 (runner persistent + launcher spawn dinamis + config provider dari DB) disetujui owner. Owner memandang konsep itu **generik dan portabel** — layak jadi dasar **platform produk** untuk project client yang ingin integrasi AI, bukan hanya fitur internal bbh-data-board.

## Keputusan Visi Owner (2026-11-24)
1. **Konsep sandbox dinamis → platform produk** untuk client integrasi AI:
   - Runner persistent + launcher spawn per-sesi (proses goose terpisah, HOME + `/work` per-sesi, di-wipe saat kill).
   - Start-config per-sesi disusun dari **config provider di DB milik deployment itu** (polanya sudah terbukti di bbh-data-board: `plugins/ingest/services/providers.go` + dekripsi AES-GCM hanya di Go, `provider_key.go`).
   - Deployment per-client = infra sandbox + DB/secret terpisah (tidak share sandbox antar client).
2. **Dua tier produk**:
   | Tier | Konsep | Faktanya di bbh-data-board |
   |---|---|---|
   | **Basic** | Chatbot + tools basic **TANPA sandbox** | Sudah ada — ingest chat eksisting (CopilotKit, Node runtime, tools via registry) |
   | **Full** | Jalur sandbox ketat ala Goose + permission + fail-closed | Sedang dibangun (topic `1795470770884`, CP-1..CP-3) |
   - Tier = jalur integrasi yang dipilih per deployment/client; dapat menjadi flag per deployment di DB (masih usulan, belum dikunci).

## Implikasi
- **TIDAK mengubah scope** WO berjalan (CP-1..CP-3 di topic chatbot) — topic ini hanya arah produk.
- Desain CP-2 (UI/role) di topic chatbot sebaiknya **tidak mengunci** asumsi "satu tier" — biarkan Basic/Full bisa dipilih di level deployment.
- Bila suatu saat platform ini jadi produk formal: perlu topic sendiri (portabilitas image, onboarding client, billing?, dukungan multi-tenant) — TIDAK dibuat sekarang.

## Sumber keputusan
- Diskusi 2026-11-24 (pertanyaan owner "kita bisa gitu kan?" → ya; "2 jenis, Basic dan Full" → valid dan didukung arsitektur).
- Catat juga di notes topic `1795470770884` (save `97868db`).

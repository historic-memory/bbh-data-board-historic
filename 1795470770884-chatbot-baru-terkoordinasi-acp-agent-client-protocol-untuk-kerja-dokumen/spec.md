---
title: spec
description: "Catatan keputusan owner + audit: chatbot baru sebagai ACP client dengan agent umum (ala OpenCode) untuk kerja dokumen, TANPA menyentuh ingest chat eksisting."
status: draft
created: "2026-11-24"
updated: "2026-11-24"
tags: [riset, agent, acp, chatbot, catatan]
related: ["1795469702082", "1795414203062", "1793711295286"]
---

# SPEC — Chatbot Baru Terkoordinasi ACP (Catatan Awal)

## Status
CATATAN AWAL — riset & SPEC menyusul. Belum ada keputusan desain final, belum ada WO teknis.

## Keputusan Owner (2026-11-24)
1. Chatbot **baru, terpisah** dari ingest chat eksisting (`/ingest/:id/chat` via CopilotKit) — tidak menghapus/mengubah yang sekarang.
2. Chatbot baru berperan **ACP client** (Agent Client Protocol, Zed/OpenCode) — agent umum "1 paket" (edit file, shell, exec kode) jadi pekerja manipulasi dokumen.
3. "Fitur deterministik" di bbh-data-board **diputus dihapus/clear**.

## Hasil Audit "Fitur Deterministik" (2026-11-24) — prasyarat pembersihan
Pemindaian kode + dokumen (grep DecisionClassifier/classifier/Jev/deidentify):
- Fungsi teks deterministik = milik **provider donis-agent** (proyek owner, di luar repo ini). Tidak ada kodenya di bbh-data-board.
- `DecisionClassifier`/Jev (rencana WO-02 D-3) **TIDAK PERNAH DIKODEKAN** — hanya komentar arsitektural di `plugins/ingest/services/rules.go:18` ("pola classifier D-3") + referensi di SPEC/notes topic ingest.
- **TIDAK DIHAPUS** (bukan target): `deidentify` HMAC deterministik di `plugins/ingest/services/deidentify.go` (proteksi PII data warga) dan pipeline deterministik (state machine + penegak rule) — keduanya inti fail-closed ingest chat eksisting.
- Kesimpulan: "menghapus fitur deterministik" = mencabut **referensi/komentar usang** (kecil, kosmetik). Eksekusi pembersihan DIUNDUR sampai riset chatbot baru selesai (biar satu topic housekeeping, tidak bikin WO kecil sekarang).

## Kerangka Chatbot Baru (usulan awal — belum dikunci)
- UI: halaman admin baru terpisah (bukan `/ingest/*`), berperan ACP client.
- Agent: runtime umum (kandidat: OpenCode dkk.) berjalan **dalam sandbox ketat** dari topic `1795414203062` (no-net, fs terbatas per-sesi, audit, keluaran hanya usulan/diff; eksekusi penyimpanan tetap lewat validasi Go). Fail-closed tetap.
- Transport ACP: agent sebagai **subprocess** (JSON-RPC over stdio) — jalur yang sudah stabil; jalur remote (HTTP/WSS) masih WIP di protokol, jadi bukan asumsi desain.
- PII: sebelum dokumen masuk agent, deidentify di Go (pola existing).

## Risiko Terbuka (untuk riset)
- O-1: pemilihan agent (OpenCode vs alternatif — research menyusul per minta owner).
- O-2: kelayakan subprocess di container `ingest_runtime` vs container terpisah.
- O-3: LLM provider agent (donis-agent deterministik milik owner? API eksternal?) + jaminan determinisme.
- O-4: RBAC/audit: bagaimana aksi agent dipetakan ke permission & `ingest_audit` (baru `acp_audit`?).
- O-5: scoping data: berkas apa saja yang boleh dilihat agent (read-only snapshot vs live).

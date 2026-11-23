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
RANCANGAN FINAL — Goose terkunci sebagai agent; menunggu review owner sebelum WO dibuat.

## Keputusan Terkunci (2026-11-24, owner "aku mantap")
- **O-1 DIKUNCI**: agent = **Goose** (Block, Apache-2.0, Linux Foundation/AAIF) via `goose acp` sebagai subprocess.
- **O-2 DIKUNCI**: container **terpisah** khusus `goose` (rekomendasi Manager disetujui owner) — bukan menumpang `ingest-runtime`.
- **O-3 DIKUNCI**: provider LLM = **donis-agent** (OpenAI-compatible, proyek owner) via konfigurasi provider Goose; fallback env provider lain disiapkan hanya sebagai dev convenience, bukan produksi. PII tetap dideidentify di Go sebelum berkata keluar sandbox.
- Isolasi ketat container = prasyarat desain (tabel 6 lapis di notes 2026-11-24).

## Rencana Checkpoint (usulan Manager — pola checkpoint ingesti)
- **CP-1**: container `goose` terisolasi + `goose acp` hidup + smoke test JSON-RPC stdio dari proses Node (tanpa data bisnis; fixture fiktif). → **WO-CP1 didraft 2026-11-24** (`wos/01-cp1-container-goose-acp-smoke.md`, status planned): transport final = `goose serve` HTTP di network internal (alasan: container terpisah membuat stdio subprocess tidak bisa tanpa mount docker.sock = DITOLAK; `goose serve` adalah mekanisme resmi goose dengan auth `X-Secret-Key` di port 3284).
- **CP-2**: UI chatbot baru (halaman admin terpisah, ACP client sisi Node) + kartu permission (approve/deny dari `session/request_permission`) + audit aksi ke tabel audit baru (`acp_audit`).
- **CP-3**: integrasi kerja dokumen ingest: snapshot berkas sesi read-only → Goose usul transform/diff → approve → eksekusi lewat validasi Go. Pembersihan referensi "deterministik usang" (rules.go:18 + dok) masuk WO housekeeping CP-3.

## Pertanyaan Terbuka Sisa
- O-4: mapping RBAC → permission prompt ACP (siapa boleh approve apa).
- O-5: scoping data: berkas apa yang boleh masuk sandbox per sesi (usulan: hanya snapshot read-only berkas sesi aktif).


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

---
title: "Integrasi LLM & AI-ETL"
description: "Rencana AI-assisted ETL: CopilotKit, Jobmanager + Sandbox, ACP Agent, opsi model lokal/fine-tune"
status: progress
created: "2026-10-27"
---
# Integrasi LLM & AI-ETL (CopilotKit, Jobmanager Sandbox, ACP Agent)

## Tujuan
- Merancang alur impor data mentah (tiap cabang berbeda format) menjadi data canonical BBH Data Board, dengan AI sebagai jembatan pembuat transformasi.
- Topik ini TERPISAH dari topic "Analisa Data Mentah Business" yang fokus ke skema ERD tabel.

## Ide Inti User
- Raw data (format tiap cabang beda) -> AI jadi jembatan -> keluar Python script sesuai standar BBH Data Board.
- Webapp akan integrasi LLM via **CopilotKit**.
- LLM (awal rencana) membaca **header saja**, lalu buat script, hasilnya ditampung ke **data sementara (staging)**.
- Runtime Python via **Jobmanager**: tiap job item jalan di **sandbox** khusus.
- Di sandbox ada LLM yang bisa diperintah; user menyebut pola **ACP agent**.

## Catatan/Penajaman Manager
1. **"Header saja" tidak cukup**: butuh header + sampel nilai ter-mask. Bukti dari CSV bayi: `Berat "2600 gram"`, `Panjang "-"`, dua format tanggal (`19 November 2021` vs `28/02/2022`), RM berisi teks+URL, kolom `Place` kosong vs `Tempat Lahir`, enum `Status`/`Agama`/`Ditolong Oleh` butuh daftar nilai unik.
2. **Script tidak bisa jalan di browser**; container `app` Go-only (tanpa Python). Perlu **Import Service** terpisah / runner Python.
3. **ACP ada dua protokol berbeda**:
   - **Agent Client Protocol** (Zed) — JSON-RPC over stdio, mengemudikan coding agent (Claude Code/Gemini CLI/dsb). INI yang dimaksud user.
   - **Agent Communication Protocol** (IBM/BeeAI) — REST antar-agent.
4. **Artefak kontrak per job**: `mapping.yaml`, `transform.py` (deterministik/idempotent), `test_cases.json`, `report.json`, `staging.ndjson`.
5. **Guardrail sandbox**: non-root, no docker socket, raw read-only, timeout CPU/mem/disk/wall-clock, egress allow-list (hanya ke LLM), batas iterasi agent, audit log.
6. **Rekomendasi pola**: AI usulkan **mapping deklaratif (YAML)** dulu -> generator deterministik cetak Python. Alternatif "LLM tulis Python penuh" hanya dengan golden test + dry-run + diff report + review manusia + script versioned.
7. **Target canonical schema wajib ada dulu** (ada di topic ERD) — AI-ETL tanpa target = menembak buta.

## Opsi Model
- **Embedded / local LLM** (llama.cpp/llama-server, Ollama, vLLM, llama-cpp-python, ONNX): model dibundel/dijalankan lokal, TANPA egress cloud -> **PII aman total**, biaya tetap, deterministik lebih terjaga. Trade-off: kualitas lebih rendah, boros RAM/CPU (7B Q4 ~4-6GB; 14B ~8-12GB), image besar, cold start.
  - Pola disarankan: LLM lokal di-serve sebagai service internal (shared), sandbox hanya client ACP.
- **Embedding model** (beda hal): model kecil teks->vektor, untuk mencocokkan nama kolom ke field canonical secara semantik (bukan penulis kode).
- **Hybrid**: lokal default (data sensitif) + cloud opt-in (dengan pseudonym) untuk format sulit.
- **Fine-tune model kecil (LoRA)**: BISA tetapi BUKAN langkah pertama. Butuh dataset berlabel (belum ada, baru 1 CSV), target schema belum beku, variasi tinggi, schema masih evolve, beban MLOps. Jalur bertahap: prompt -> agent+tools+validator -> kumpulkan golden set -> LoRA untuk pola stabil -> model offline khusus.

## Keputusan Terbuka (D1-D9)
- D1. AI boleh lihat data mentah? (header saja / sampel ter-mask / pseudonym format-preserving / lokal penuh)
- D2. Output AI: mapping YAML dulu atau Python penuh?
- D3. Script hidup di mana: tooling offline atau dalam sistem?
- D4. Target: staging canonical dulu atau langsung DB?
- D5. Mapping disimpan permanen & versioned per sumber/cabang?
- D6. Sandbox Python: container terpisah dari `app`?
- D7. Runtime CopilotKit mengarah ke backend Go atau langsung provider LLM?
- D8. Source of truth staging: tabel DB cabang atau NDJSON?
- D9. LLM lokal (self-hosted) penuh, atau hybrid (lokal default + cloud opt-in)?

## Urutan Fase (usulan)
1. Canonical Schema / ERD (topik terpisah) — target
2. Staging table + validator (Go)
3. Import Service / runner + Jobmanager + sandbox image (python + agent ACP)
4. Runner final tanpa LLM + kontrak artefak
5. LLM mapping/generate (mulai mapping YAML dari header+sample masked)
6. CopilotKit UX (chat, review, approve, diff)
7. (Nanti) kumpulkan dataset -> fine-tune LoRA bila perlu

## Status
- Diskusi berjalan; belum ada keputusan final. Fokus dialihkan sementara ke topic ERD.


---
title: spec
description: "doniswork — visi produk platform chatbot AI 2-tier: Basic (chatbot + tools basic, tanpa sandbox) dan Full (sandbox dinamis ketat ala Goose — runner persistent + launcher spawn per-sesi + config provider dari DB) — dipakai sebagai fondasi integrasi AI untuk project client."
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
3. **Bentuk penyajian: SaaS** — platform ini disiapkan sebagai SaaS, dengan "kemudahan-kemudahan" untuk client (detail di bawah).
4. **Nama platform: `doniswork`** — dikunci owner (2026-11-24). Nama resmi produk platform (paket Basic & Full); dipakai untuk penamaan dokumen/repo/nama produk ke depan.

## Detail Tier BASIC (dari owner, 2026-11-24 — SaaS)
Goal Basic = SaaS self-service untuk bikin chatbot ber-tools, tanpa sandbox:

1. **User membuat agent** (banyak agent per client/tenant; tiap agent = definisi di DB: nama, prompt, tools ter-assign, konfigurasi).
2. **Editor function tools per agent** — kemudahan utama: user mendefinisikan/menyunting tools lewat editor; format standar **YAML** sebagai kandidat (belum dikunci) atau **stateless scripting minimal** untuk kebutuhan logic. Arah kandidat teknis: skema YAML deklaratif ala tool-spec (nama, deskripsi, parameter JSON-Schema, mapping) — mirip pola tools registry yang sudah ada di runtime Node (CopilotKit); logic kompleks opsional via scripting stateless kecil. Keamanan tetap: tools basic TIDAK dijalankan di sandbox, jadi scripting harus stateless + terbatas (tanpa IO, tanpa network, timeout).
3. **Routing antar-agent** — tiap agent mengenali agent lain di tenant-nya; bila permintaan customer tidak tercakup knowledge/capability agent aktif, permintaan **dialihkan ke agent lain** yang relevan (handover/routing multi-agent, level tenant).

Catatan: **versi ACP/Full = paket berbeda** dengan kasus berbeda (kerja dokumen di sandbox) — tidak dicampur fitur dengan Basic; Saya mempertahankan pemisahan tier.

## Riset Feasibility Tier Basic (Manager, 2026-11-24)
Owner menantang feasibility ("tapi possible gak seperti itu") → Manager riset web. **Kesimpulan: YA, POSSIBLE** — keempat pilar Basic punya preseden industri:

1. **Routing antar-agent** — terbukti:
   - **OpenAI Agents SDK** (docs resmi): handoffs *"are represented as tools to the LLM"* — tool `transfer_to_<agent_name>`, `handoff_description` sebagai hint kapan model memilih handoff, `handoff()` kustom (`input_type`, `input_filter`, `is_enabled`); input filter membersihkan history tool saat handover.
   - **AutoGen SelectorGroupChat** (Microsoft): next-speaker **dipilih model** berdasar `name` + `description` tiap agent (capability-descriptor routing); ada `selector_prompt`, `selector_func`/`candidate_func` kustom, termination conditions; contoh e2e Planning/WebSearch/DataAnalyst.
   - Implikasi: routing = **deskriptor capability per agent** (+ handoff-as-tool atau selector terpusat); disarankan tetap ada **policy mesin** (bukan murni prompt) untuk keandalan.
2. **Format declarative function tools** — ada standar: **OpenAPI 3.2.1** (spec.openapis.org) = *"standard, language-agnostic interface description"*; YAML↔JSON round-trip; Schema Object = superset **JSON Schema Draft 2020-12** → format YAML tool-spec (nama, deskripsi, parameter JSON-Schema) selaras standar de-facto function calling.
3. **Scripting stateless tanpa sandbox** — kelas **expression engine** terbukti aman (BUKAN full scripting JS/Python):
   - **expr-lang/expr** (Go, v1.18.1, MIT): TIGA jaminan eksplisit di README — **"Memory-Safe"**, **"Side-Effect-Free"** (output hanya dari input, tanpa efek samping), **"Always Terminating"** (tidak ada infinite loop); static typing; dipakai produksi Google Cloud Platform, Uber, GoDaddy, ByteDance, Argo, OpenTelemetry, CoreDNS. Go-native → cocok backend bbh Go.
   - Alternatif: **CEL** (Google, cel-spec) — common expression language gradual typing, didesain safe/fast.
   - Implikasi: scripting = **expression-only**; karena sengaja TANPA IO/network, sandbox TIDAK diperlukan — safety dari desain bahasa + evaluator.
4. **Platform SaaS agent-builder eksisting** — **Botpress**: Studio drag-and-drop agent builder + integrations/tools/channels + webchat embed → pola "user/client bikin agent" sudah jadi produk nyata.

Gagal fetch (tidak krusial): higress.ai, docs.n8n.io advanced-ai/multi-agent (404).

**Jawaban atas 4 pertanyaan terbuka** (section di bawah):
- Format editor → **terjawab**: YAML deklaratif ala OpenAPI/JSON-Schema; scripting minimal bila perlu = expression engine (expr/CEL).
- Mekanisme routing → **terjawab**: capability-descriptor + handoff-as-tool/selector; tambah policy mesin.
- Keamanan scripting → **terjawab**: expression engine side-effect-free + always-terminating (bukan interpreter umum).
- Model multi-tenant → **masih terbuka** (isolasi data/billing/onboarding).

## Pertanyaan/keputusan teknis yang terbuka (untuk topic teknis nanti)
- Format tool editor final: YAML deklaratif vs scripting minimal vs hybrid (YAML + hook logic).
- Mekanisme routing antar-agent: router terpusat vs deskriptor capability vs prompt-based handover.
- Batas keamanan scripting stateless (interpreter apa, sandbox CPU, kuota).
- Model multi-tenant SaaS: isolasi data, billing, onboarding.

## Implikasi
- **TIDAK mengubah scope** WO berjalan (CP-1..CP-3 di topic chatbot) — topic ini hanya arah produk.
- Desain CP-2 (UI/role) di topic chatbot sebaiknya **tidak mengunci** asumsi "satu tier" — biarkan Basic/Full bisa dipilih di level deployment.
- Bila suatu saat platform ini jadi produk formal: perlu topic sendiri (portabilitas image, onboarding client, billing?, dukungan multi-tenant) — TIDAK dibuat sekarang.

## Sumber keputusan
- Diskusi 2026-11-24 (pertanyaan owner "kita bisa gitu kan?" → ya; "2 jenis, Basic dan Full" → valid dan didukung arsitektur).
- Catat juga di notes topic `1795470770884` (save `97868db`).

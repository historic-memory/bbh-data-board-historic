---
title: WO-01 — Setup CopilotKit + proxy LLM + sandbox belajar
description: "CP-1: CopilotKit di admin panel via proxy OpenAI-compatible; belajar skill/tool dummy, eksperimen MCP, testing upload. TANPA data bisnis."
status: pending
created: "2026-11-03"
---
# WO-01 — Setup CopilotKit + Proxy LLM + Sandbox Belajar (Checkpoint 1)

Checkpoint: **CP-1**. Sumber: `spec.md` §9 CP-1, `spec-techstack.md` §2.2/§2.3.

## Tujuan

Menghidupkan rantai chat end-to-end di admin panel sebagai **lingkungan belajar**
(sandbox) — BUKAN fitur final. Setelah WO ini, tim memahami perilaku CopilotKit,
proxy, skill/tool, MCP, dan upload; tidak ada data bisnis yang tersentuh.

## Lingkup

### A. Backend — proxy LLM di plugin `ingest`

- Skeleton plugin `plugins/ingest` (pola `hsh`: `plugin.go`, `cli.go`, stub
  `services`/`handlers`; daftar di `cmd/server` + `cmd/console`).
- Endpoint proxy OpenAI-compatible (guard `ingest.write` sementara — seed permission
  via migrasi plugin `auth`):
  - `POST /api/admin/ingest/llm-proxy/*` (passthrough chat completions);
  - injeksi `Authorization: Bearer $INGEST_LLM_API_KEY` di SATU titik (middleware);
  - `INGEST_LLM_BASE_URL` / `INGEST_LLM_MODEL` via env; fail-closed bila key kosong;
  - streaming passthrough SSE (`c.Stream`); polling fallback bila SSE gagal;
  - audit setiap request proxy (pola endpoint_audit).
  - **Gate deidentify PENUH ditunda ke WO CP-2/3** (payload khusus sandbox); tapi
    struktur gate (checker kolom) sudah disiapkan sebagai stub yang default-deny
    untuk kolom tertanda PII di masa depan.
- Tabel minimal: hanya yang diperlukan sandbox — `ingest_messages` (log chat) via
  migrasi plugin `ingest` v1. (`ingest_sessions`/`ingest_profiles` menyusul CP-2.)

### B. Frontend — CopilotKit di admin panel

- Tambah dependensi CopilotKit (runtime + React) — SATU penambahan npm baru selain
  `openai`-compat jika diperlukan; base URL runtime → proxy backend.
- Halaman/rute sandbox (grup nav baru "Ingest", guard `ingest.read` untuk membuka,
  `ingest.write` untuk memakai): chat dasar + streaming tampil.

### C. Eksperimen belajar (di dalam sandbox, tervalog, bukan fitur)

1. **Skill/tool dummy** — satu tool echo (mis. `sandbox_ping`) yang dipanggil agent
   via tool-calling CopilotKit → dieksekusi di backend (guard RBAC), hasilnya tampil
   di chat. Membuktikan pola "tool = API backend" yang akan dipakai semua tool nyata.
2. **Eksperimen MCP** — coba pasang satu MCP server kecil di luar jalur utama;
   catat temuan (kompatibilitas dengan proxy, effort) di notes topic. TIDAK dijadikan
   standar arsitektur.
3. **Testing upload** — tombol/komponen upload file di chat: file masuk
   `internal/storage`, tercatat di DB, TIDAK diproses. Tampil ukuran/nama di chat.

## Acceptance Criteria

- AC-1 Chat dasar + streaming hidup: UI → proxy → provider, balasan mengalir SSE.
- AC-2 API key TIDAK muncul di response/network klien (uji: grep respons & devtools).
- AC-3 Tanpa key env → proxy 503 fail-closed dengan pesan jelas.
- AC-4 Tool dummy `sandbox_ping` terpanggil dari chat dan hasilnya tampil.
- AC-5 Upload file via chat tersimpan di storage + tercatat; tidak ada proses lain.
- AC-6 Tanpa token (401) / tanpa `ingest.write` (403) → proxy & upload ditolak.
- AC-7 Temuan eksperimen MCP tercatat di notes topic.
- AC-8 `go vet`/`go test` hijau; `npm run build` hijau; swagger berisi route proxy.

## Non-tujuan (dipindah ke checkpoint lain)

- Sesi strict, state machine lengkap, tebakan mapping, deidentify nyata, classifier
  gate, ekstraksi, mapping editor (CP-2/3/4).

## Risiko & catatan

- CopilotKit runtime butuh layer adaptasi kecil di proxy (formatnya sedikit lebih
  dari OpenAI-compatible) — catat effort riil di notes untuk rancangan CP-4.
- Provider sandbox: gunakan `INGEST_LLM_BASE_URL` yang OpenAI-compatible; hindari
  mengirim data nyata BBH di masa sandbox (prinsip: hanya contoh fiktif).

---
title: WO-01a — Proxy LLM + CopilotKit chat hidup (CP-1, sub 1/3)
description: "Sub-checkpoint CP-1: skeleton plugin ingest + proxy OpenAI-compatible + CopilotKit sandbox page dengan streaming hidup. Tanpa tool/upload/MCP."
status: pending
created: "2026-11-03"
---
# WO-01a — Proxy LLM + CopilotKit Chat Hidup (Sub-Checkpoint CP-1, 1/3)

Induk: `wo-01-setup-copilotkit-sandbox.md` (dipecah 2026-11-04). Urutan: **1a → 1b
→ 1c**; prasyarat sub berikutnya = sub sebelumnya lulus AC. Sumber: `spec.md` §9
CP-1, `spec-techstack.md` §2.2.

## Tujuan

Menghidupkan rantai chat end-to-end **minimal** di admin panel: UI CopilotKit →
proxy backend → provider LLM, streaming mengalir. Belum ada tool, upload, atau MCP.
Sandbox — BUKAN fitur final; tidak ada data bisnis tersentuh.

## Lingkup

### Backend — proxy LLM di plugin `ingest`

- Skeleton plugin `plugins/ingest` (pola `hsh`: `plugin.go`, `cli.go`, stub
  `services`/`handlers`; daftar di `cmd/server` + `cmd/console`).
- Migrasi permission via plugin `auth` (fail-closed): seed `ingest.read`/`ingest.write`.
- Endpoint proxy OpenAI-compatible (guard `ingest.write`):
  - `POST /api/admin/ingest/llm-proxy/*` (passthrough chat completions);
  - injeksi `Authorization: Bearer $INGEST_LLM_API_KEY` di SATU titik (middleware);
  - `INGEST_LLM_BASE_URL` / `INGEST_LLM_MODEL` via env; fail-closed bila key kosong (503);
  - streaming passthrough SSE (`c.Stream`); polling fallback bila SSE gagal;
  - audit setiap request proxy (pola endpoint_audit).
- Struktur gate deidentify disiapkan sebagai stub default-deny untuk kolom tertanda
  PII di masa depan (aktivasi penuh di WO-02/CP-2).
- Migrasi plugin `ingest` v1: tabel `ingest_messages` (log chat) saja.

### Frontend — CopilotKit sandbox page

**Keputusan UI (owner, 2026-11-04):** penempatan chat = **Opsi 1 — halaman chat
khusus (dedicated page)**; design kit **standar**: ada daftar sesi (session list)
dan kotak pesan chat (chat message box). Bukan chat global melayang, bukan chat
ter-embed di halaman domain (kandidat CP-4/lanjutan).

- Tambah dependensi CopilotKit (runtime + React) — satu-satunya penambahan npm baru
  di sub-WO ini; base URL runtime → proxy backend.
- Rute (grup nav baru "Ingest" via `NAV_GROUPS` di `navigation/menu.ts`, guard
  `ingest.read` membuka / `ingest.write` memakai):
  - `/ingest` — daftar sesi chat (session list: judul/waktu, tombol buka sesi);
  - `/ingest/:id/chat` — halaman chat: panel pesan (bubble user/agent, indikator
    streaming, kotak input) — CopilotKit di-mount per halaman ini saja, tidak di
    `AdminLayout` (sandbox terisolasi).
- Gaya visual: pakai komponen `components/ui/*` + Tailwind v4 yang ada; UI default
  CopilotKit boleh dipakai di sandbox tanpa skinning (polish ditunda ke CP-4).

## Acceptance Criteria

- AC-1 Chat dasar + streaming hidup: UI → proxy → provider, balasan mengalir SSE.
- AC-2 API key TIDAK muncul di response/network klien (grep respons & devtools).
- AC-3 Tanpa key env → proxy 503 fail-closed dengan pesan jelas.
- AC-9 Halaman `/ingest` menampilkan daftar sesi (session list) dan halaman
  `/ingest/:id/chat` berisi kotak pesan standar (bubble + input + streaming);
  CopilotKit TIDAK di-mount di `AdminLayout` (isolasi per halaman).
- AC-6 Tanpa token (401) / tanpa `ingest.write` (403) → proxy ditolak.
- AC-8 `go vet`/`go test` hijau; `npm run build` hijau; swagger berisi route proxy.

## Non-tujuan (pindah ke 1b/1c)

- Tool dummy `sandbox_ping` + upload (WO-01b); eksperimen MCP + housekeeping (WO-01c);
  sesi strict/state machine/deidentify nyata/classifier gate/ekstraksi/editor (CP-2/3/4).

## Risiko

- CopilotKit runtime butuh layer adaptasi kecil di proxy (formatnya sedikit lebih
  dari OpenAI-compatible) — catat effort riil di notes untuk rancangan CP-4.
- Provider sandbox: `INGEST_LLM_BASE_URL` OpenAI-compatible; hanya contoh fiktif.

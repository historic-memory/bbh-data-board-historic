---
title: WO-01a — CopilotKit chat hidup (CP-1, sub 1/3) [DIREVISI D-11]
description: "Sub-checkpoint CP-1: skeleton plugin ingest + CopilotKit (runtime Node) chat hidup; Go = data service. TANPA tool/upload/MCP."
status: pending
created: "2026-11-03"
---
# WO-01a — CopilotKit Chat Hidup (Sub-Checkpoint CP-1, 1/3) — DIREVISI D-11

Induk: `wo-01-setup-copilotkit-sandbox.md` (dipecah 2026-11-04). Urutan: **1a → 1b
→ 1c**; prasyarat sub berikutnya = sub sebelumnya lulus AC. Sumber: `spec.md` §9
CP-1, `spec-techstack.md` §2.2.

> **⚠ DIREVISI (D-11 final, 2026-11-04).** Arsitektur diganti: **runtime Node
> CopilotKit = backend chat AI penuh** (satu-satunya pemanggil provider LLM);
> **Go = data service** (RBAC, audit, state machine, deidentify). Proxy Go
> OpenAI-compatible dihapus dari jalur chat. Bagian yang sudah dibangun tim dev
> (proxy Go, injeksi model, normalisasi base URL) dipertahankan sebagai artefak
> belajar CP-1 dan diputuskan ulang di housekeeping WO-01c (hapus/tandai deprecated).

## Status REALITA (update 2026-11-04, laporan tim dev)

Spike + backend WO-01a telah dikerjakan tim dev dan **terverifikasi end-to-end**:
- Frontend: dependensi CopilotKit (`@copilotkit/react-core@1.84.1` + `@copilotkit/runtime`),
  halaman D-10 (`/ingest` + `/ingest/:id/chat`), nav grup "Ingest", CopilotKit per
  halaman; sesi masih stub `localStorage` (`lib/ingestSessions.ts`).
- Backend Go: skeleton plugin `ingest`, proxy OpenAI-compatible, injeksi model default,
  normalisasi base URL, `ingest_audit` terisi, fail-closed 503/401/403 — build/vet/test
  hijau (commit `e4f4ea9`, `a019c26`).
- Runtime Node: `runtime/copilot-runtime.ts` (`npm run dev:runtime`), Vite proxy,
  AG-UI stream terbukti; chat di browser hidup (bukti LLM menjawab).
- Catatan operasional: server Go & runtime Node masih dijalankan MANUAL di container
  (tidak ada auto-start); `.env` gitignored.

## Sisa pekerjaan WO-01a (setelah revisi D-11)

1. **Pindahkan kontrol ke kontrak D-11**: pastikan runtime Node memanggil API Go
   untuk audit/tool (untuk 1a cukup: audit event proxy → `ingest_audit` tetap pusat);
   kunci `INGEST_LLM_*` hanya di runtime Node.
2. **Migrasi plugin `ingest` v1**: tabel `ingest_messages` (log chat) — jalankan
   migrasi & sambungkan (ganti stub `localStorage` bila sudah siap; boleh menyusul
   di 1b bila backend belum selesai).
3. **docker-compose**: service `ingest-runtime` (Node) resmi (recreate otomatis,
   env dari `.env`) — hilangkan pola jalan-manual (`nohup`, `docker compose exec ... dev:runtime`).
4. Tandai/hapus endpoint proxy Go dari jalur chat (keputusan final di WO-01c).

## Acceptance Criteria (diperbarui)

- AC-1 Chat dasar + streaming hidup: UI CopilotKit → runtime Node → provider
  (SUDAH terbukti di spike; bukti dipertahankan).
- AC-2 API key TIDAK muncul di response/network klien (SUDAH; grep respons & devtools).
- AC-3 Tanpa key env → 503 fail-closed dengan pesan jelas (SUDAH di proxy Go;
  perilaku sama wajib berlaku di runtime Node).
- AC-9 Halaman `/ingest` (session list) + `/ingest/:id/chat` (bubble + input +
  streaming); CopilotKit TIDAK di-mount di `AdminLayout` (SUDAH).
- AC-6 Tanpa token (401) / tanpa `ingest.write` (403) → ditolak (SUDAH; tetap berlaku
  di semua endpoint baru).
- AC-8 `go vet`/`go test` hijau; `npm run build` hijau; swagger berisi route
  (SUDAH; ulangi setelah sisa pekerjaan).
- AC-10 (BARU, D-11) Audit event dari runtime Node tercatat di `ingest_audit` Go;
  key LLM hanya di env Node; tidak ada key di env Go proxy jalur chat.

## Non-tujuan (pindah ke 1b/1c)

- Tool dummy `sandbox_ping` + upload (WO-01b); eksperimen MCP + housekeeping
  (WO-01c — termasuk keputusan nasib endpoint proxy Go); sesi strict/state machine/
  deidentify nyata/classifier gate/ekstraksi/editor (CP-2/3/4).

## Risiko

- Node runtime menambah service produksi — N-17 dilonggarkan secara sadar (D-11);
  jaga supaya Node TIDAK memegang kebijakan (strictness/gate tetap Go).
- Jalan manual container (`nohup`, exec foreground) = pola sementara; wajib jadi
  service compose yang proper sebelum CP-1 ditutup.
- Provider sandbox: hanya data fiktif.

---
title: WO-01a — CopilotKit chat hidup (CP-1, sub 1/3) [DIREVISI D-11]
description: "Sub-checkpoint CP-1: skeleton plugin ingest + CopilotKit (runtime Node) chat hidup; Go = data service. TANPA tool/upload/MCP."
status: complete
created: "2026-11-03"
---
# WO-01a — CopilotKit Chat Hidup (Sub-Checkpoint CP-1, 1/3) — DIREVISI D-11

**COMPLETE (owner, 2026-11-04)** — semua AC-1..AC-10 lulus; commit kode
`e4f4ea9`, `a019c26`, `a22c03b`. WO-01b terbuka.

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

1. ~~Pindahkan kontrol ke kontrak D-11 (audit Node→Go)~~ — **SELESAI** (commit
   `a22c03b`): runtime Node mengirim audit event → `ingest_audit` Go terisi
   (`POST /api/copilotkit/agent/default/run`, 200); key isolation diverifikasi
   (`GO_KEYLEN=0`, Node `KEYLEN=58`).
2. ~~Migrasi `ingest_messages` + ganti stub localStorage~~ — **SELESAI**: chat
   browser → Go tersimpan (`user` + `assistant`, `client_id` AG-UI); daftar sesi
   `/app/ingest` dari API (bukan localStorage; `ingestSessions.ts` dihapus);
   idempotensi POST `client_id` sama → `duplicate` 200, count tetap.
3. ~~Service `ingest-runtime` resmi di docker-compose~~ — **SELESAI** (compose
   diperbarui; verifikasi akses via IP Tailscale `:5173` dan chat hidup E2E).
4. Keputusan nasib endpoint proxy Go (`/api/admin/ingest/llm-proxy/*`, kini
   ditandai DEPRECATED) — **disengaja ke WO-01c (housekeeping)**.

**Status: seluruh lingkup WO-01a lulus — menunggu konfirmasi owner untuk naik
status complete + buka WO-01b.** Bug yang ditemukan & diperbaiki selama eksekusi
(di `copilot-runtime.ts`, commit `a22c03b`): (1) `reportAudit` menerima cast
`Request` padahal objek asli `IncomingMessage` (URL relatif) → error handler
`finish` → unhandled rejection → proses Node mati (Node 22); (2) tee SSE membaca
chunk `Uint8Array` dengan `String(chunk)` → decode salah, 0 pesan tersimpan;
perbaikan: data plain + try/catch/catch-all agar audit tak pernah mematikan proses,
dan decode UTF-8 streaming via `TextDecoder(..., {stream:true})`.

## Acceptance Criteria (diperbarui)

- AC-1 Chat dasar + streaming hidup: UI CopilotKit → runtime Node → provider
  (SUDAH terbukti di spike + E2E browser/Tailscale; bukti dipertahankan).
- AC-2 API key TIDAK muncul di response/network klien (SUDAH; key isolation
  diverifikasi per container).
- AC-3 Tanpa key env → 503 fail-closed dengan pesan jelas (SUDAH di proxy Go;
  perilaku sama wajib berlaku di runtime Node).
- AC-9 Halaman `/ingest` (session list) + `/ingest/:id/chat` (bubble + input +
  streaming); CopilotKit TIDAK di-mount di `AdminLayout` (SUDAH).
- AC-6 Tanpa token (401) / tanpa `ingest.write` (403) → ditolak (SUDAH; tetap berlaku
  di semua endpoint baru).
- AC-8 `go vet`/`go test` hijau; `npm run build` hijau; swagger berisi route
  (SUDAH; `tsc --noEmit` juga hijau).
- AC-10 (BARU, D-11) Audit event dari runtime Node tercatat di `ingest_audit` Go;
  key LLM hanya di env Node; tidak ada key di env Go proxy jalur chat (SUDAH).

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

## Catatan Implementasi (Dev)

| Berkas:baris | Yang dilakukan | Kenapa |
|---|---|---|
| `sub_app/ingest_runtime/copilot-runtime.ts:41-46` | `normalizeBaseUrl` menambah skema: `https` default, `http` untuk localhost/loopback | `.env` sering diisi `host/v1` tanpa skema; SDK OpenAI menuntut URL absolut — pola sama dengan normalisasi di plugin Go |
| `sub_app/ingest_runtime/copilot-runtime.ts:49-57` | `INGEST_LLM_*` disalin ke `OPENAI_*` **di satu titik** saja | CopilotKit membaca `OPENAI_*`; menyebar nama env ke banyak file = sumber bug "kenapa key tidak terbaca" |
| `sub_app/ingest_runtime/copilot-runtime.ts:90-130` | `identifyUser` membaca klaim JWT untuk **tampilan** saja, dengan komentar eksplisit "jangan dipakai sebagai bukti identitas" | Backend tetap penentu (N-10); Node tidak boleh jadi sumber otorisasi |
| `sub_app/ingest_runtime/copilot-runtime.ts:478-530` | `reportAudit` mengirim metadata saja (path/method/status/durasi), best-effort, tidak pernah throw | Node 22 mematikan proses pada unhandled rejection → audit gagal tidak boleh mematikan chat |
| `sub_app/ingest_runtime/copilot-runtime.ts:485-492` | Bearer milik **user** diteruskan apa adanya ke Go | `admin_id` diambil Go dari token, bukan dari body — tidak bisa dipalsukan klien |
| `sub_app/ingest_runtime/copilot-runtime.ts:569-600` | `teeAgUiEvents` memakai `TextDecoder(..., { stream: true })` dan menyangga baris | Bug: karakter UTF-8 terpotong antar chunk membuat `JSON.parse` gagal → event hilang/absen → pesan assistant tidak tersimpan |
| `sub_app/ingest_runtime/copilot-runtime.ts:620-640` | `attachPersistence` menerima `IncomingMessage` + `ServerResponse` eksplisit | Bug: tipe request/response ditebak dari objek generik → `req.headers` undefined saat runtime; anotasi tipe membuat kesalahan ketahuan saat compile |
| `sub_app/ingest_runtime/copilot-runtime.ts:757-772` | Persistensi hanya untuk `/agent/` dengan sesi dari header `X-Ingest-Session-Id` | Id sesi di header, bukan query: tidak bocor di log proxy; endpoint non-run tidak menulis pesan palsu |
| `sub_app/ingest_runtime/copilot-runtime.ts:773-790` | Audit dipasang di `res.on('finish')` | Status/durasi baru final saat respons selesai; mencatat lebih awal = angka bohong |
| `sub_app/ingest_runtime/copilot-runtime.ts:768` + `plugins/ingest/handlers/handlers.go` (`audit`) | Persistensi pesan/audit menulis ke Go (bukan localStorage) | D-11: satu sumber kebenaran; riwayat chat tetap ada meski browser dibersihkan |
| `sub_app/ingest_runtime/copilot-runtime.ts:791-800` | Peringatan start bila `INGEST_GO_API_URL` kosong | Fail-fast yang terlihat di log, bukan kegagalan senyap saat chat dipakai |

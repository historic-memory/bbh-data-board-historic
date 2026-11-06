---
title: WO-01b — Tool dummy sandbox_ping + upload storage (CP-1, sub 2/3)
description: "Sub-checkpoint CP-1: pola tool-calling (tool = API backend) via tool dummy, dan upload file ke storage tanpa diproses."
status: complete
created: "2026-11-03"
---
# WO-01b — Tool Dummy `sandbox_ping` + Upload ke Storage (Sub-Checkpoint CP-1, 2/3)

**COMPLETE (owner + Manager review, 2026-11-04)** — commit `f00e542`; AC-4/5/6/8
lulus dengan bukti; pola tool & upload terbukti; prasyarat WO-01c terpenuhi.

Induk: `wo-01-setup-copilotkit-sandbox.md` (dipecah 2026-11-04). Urutan: 1a →
**1b** → 1c. **Prasyarat: WO-01a lulus AC-nya** (chat streaming hidup, key aman,
fail-closed terbukti).

## Tujuan

Membuktikan dua pola interaktivitas dasar yang akan dipakai semua checkpoint
berikutnya: (1) **tool = API backend** yang dipanggil agent via tool-calling, dan
(2) file yang di-upload via chat tersimpan aman di storage. Masih sandbox — tanpa
data bisnis, tanpa pemrosesan file.

## Lingkup

### 1. Tool dummy `sandbox_ping`

- Satu tool echo sederhana: agent memanggil `sandbox_ping` via tool-calling CopilotKit
  → dieksekusi di **backend** (guard RBAC `ingest.write`, audit) → hasil tampil di chat.
- Tool didaftarkan di sisi backend (bukan didefinisikan di klien) — pola yang akan
  dipakai semua tool nyata di CP-2 (mis. `get_mapping_guess`, `confirm_mapping`).
- Catat di notes: bagaimana CopilotKit melewatkan tool-call ke backend (mechanism,
  latency, kegagalan).

### 2. Upload file via chat

- Komponen upload di panel chat: file masuk `internal/storage`, tercatat di DB
  (tabel/metadat minimal), **TIDAK diproses** — hanya nama + ukuran tampil di chat.
- Batas ukuran/ekstensi ditolak fail-closed (rebut pola `.xls` N-17 sejak sekarang
  untuk membiasakan perilakunya).
- Guard RBAC + audit sama seperti proxy.

## Acceptance Criteria

- AC-4 Tool dummy `sandbox_ping` terpanggil dari chat dan hasilnya tampil di chat.
- AC-5 Upload file via chat tersimpan di storage + tercatat; tidak ada proses lain.
- AC-6 Tanpa token (401) / tanpa `ingest.write` (403) → pemanggilan tool & upload ditolak.
- AC-8 `go vet`/`go test` hijau; `npm run build` hijau; swagger berisi endpoint tool
  & upload; temuan pola tool-calling dicatat di notes topic.

## Non-tujuan

- Pemrosesan isi file apa pun (parsen Excel/CSV — CP-2); MCP (WO-01c); sesi strict/
  state machine/deidentify/ekstraksi/editor (CP-2/3/4).

## Risiko

- Jangan biarkan tool dummy menyimpan/mengembalikan apa pun selain echo — fondasi
  konvensi tool (nama, guard, audit) yang akan disalin tool nyata.

## Implementasi (Dev Agent, 2026-11-04)

Rujukan bukti lengkap + jebakan: `notes.md` bagian "Temuan pola tool-calling &
upload (WO-01b)".

### Tool `sandbox_ping` — tool = API backend

- **Go** `POST /api/admin/ingest/tools/sandbox_ping` (guard `ingest.write`,
  audit `provider='ingest-tool'`). Body `{message, session_id}`; respons
  `{tool, echo, received_at}`. Hanya echo — tanpa state/DB (`services/tools.go`).
- **Node** (`runtime/copilot-runtime.ts`): `CopilotRuntime.agents` dijadikan
  *factory* per-request; `BuiltInAgent` dibangun ulang dengan tool
  `sandbox_ping` yang `execute()`-nya memanggil API Go memakai Bearer user +
  `X-Ingest-Session-Id`. Tool tidak didefinisikan di klien.
- Kompatibilitas: `runtime/provider-compat.ts` menyisipkan `index` pada chunk
  `tool_calls` SSE provider (gateway mengirimnya tanpa `index`; AI SDK menolak
  → run gagal). Shim dicatat sebagai temuan; hapus bila gateway diperbaiki.

### Upload file — simpan, jangan proses

- **Go** `POST /api/admin/ingest/sessions/:id/uploads` (guard `ingest.write`)
  + `GET` daftar (guard `ingest.read`). Disimpan via `internal/storage`
  (key acak `ingest/uploads/<session>/<uuid><ext>`), metadata di tabel baru
  `ingest_uploads` (idempotent `client_id`, pola WO-01a).
- Fail-closed: `.csv`/`.xlsx` saja; `.xls`/`.xlsm`/`.xlsb` → 415 (N-17),
  ekstensi lain → 415, > 10 MiB → 413, storage tidak ada → 503.
- **SPA**: panel `IngestUploadPanel` di halaman chat (nama + ukuran tampil;
  tidak ada render isi file).

### Bukti AC

| AC | Bukti |
|----|-------|
| AC-4 | Chat browser "tolong jalankan sandbox_ping…" → `TOOL_CALL_RESULT` `pong: halo dari browser` tampil di chat; audit `/api/admin/ingest/tools/sandbox_ping` 200 |
| AC-5 | Upload `contoh-sandbox.csv` 17 B → baris `ingest_uploads` + file `storage/ingest/uploads/<sid>/<uuid>.csv`; tidak ada proses lanjutan |
| AC-6 | tool & upload tanpa token → 401; viewer (`ingest.read` saja) → 403 `izin ditolak: butuh ingest.write`; list sesi viewer → 200 |
| AC-8 | `go build`/`go vet`/`go test ./plugins/ingest/...` hijau; `tsc --noEmit` + `npm run build` hijau; swagger memuat `/admin/ingest/tools/sandbox_ping` & `/admin/ingest/sessions/{id}/uploads`; temuan dicatat di `notes.md` |

## Catatan Implementasi (Dev)

| Berkas:baris | Yang dilakukan | Kenapa |
|---|---|---|
| `sub_app/ingest_runtime/copilot-runtime.ts:374-460` | `createAgent(request)` membangun agent **per request**, tool didaftarkan di sisi server (bukan klien) | Tool yang didefinisikan klien bisa dipalsukan; registrasi server = satu-satunya sumber daftar tool (D-11) |
| `sub_app/ingest_runtime/copilot-runtime.ts:261-300` | `callGoTool` memakai Bearer **milik user** dan meneruskan 401/403 sebagai error tool | RBAC + audit tetap di Go; Node tidak punya identitas istimewa dan model diberi tahu kegagalan, bukan pura-pura sukses |
| `plugins/ingest/plugin.go` (`RegisterRoutes`) | Semua route tool/upload di belakang `ingest.read`/`ingest.write` + `h.audit(...)` per request | Pola definitif CP-2: guard RBAC fail-closed + jejak audit tanpa perlu diingat per handler |
| `plugins/ingest/handlers/tools.go:41-60` | `SandboxPing` = echo tool pertama, dieksekusi di Go | Membuktikan rantai tool-calling Node → Go sebelum tool nyata ditambahkan |
| `plugins/ingest/handlers/uploads.go:45-95` | Upload: `io.LimitReader(f, MaxUploadBytes+1)` lalu serahkan ke service | Batas dibaca di sisi baca, bukan `Content-Length` yang bisa dibohongi; `.xls` ditolak 415 (N-17) |
| `plugins/ingest/services/uploads.go:31-80` | `ValidateUpload`: whitelist `.csv/.xlsx`, tolak `.xls/.xlsm/.xlsb`, batas 10 MiB | `.xls` (BIFF8) punya pembaca berbeda & rapuh; pesan penolakan menyuruh simpan ulang, bukan error kriptik |
| `plugins/ingest/services/uploads.go:100-150` | `SaveUpload` menyimpan lewat `internal/storage` dengan key acak + metadata; `client_id` opsional → UNIQUE `(session_id, client_id)` | Retry jaringan tidak membuat dua baris upload; `stored_key` tidak pernah dikirim ke klien |
| `plugins/ingest/handlers/uploads.go:119-135` | `writeUploadError` memetakan `UploadRejected` → status HTTP-nya (413/415) + audit `rejected`/`duplicate` | Kegagalan validasi tetap tercatat; klien dapat kode yang bisa ditindaklanjuti |
| `sub_app/ingest_runtime/provider-compat.ts:1-60` | Shim menyisipkan `index` yang hilang pada `delta.tool_calls[]` di SSE provider | Gateway mengirim tool_calls tanpa `index`; AI SDK memvalidasi strict → `RUN_ERROR`. Ini penambal ketidakpatuhan provider, bukan protokol — hapus bila provider diperbaiki |
| `sub_app/ingest_runtime/provider-compat.ts:43-56` | `index` diambil dari posisi dalam delta (gateway mengirim satu tool-call utuh per event) | Tidak menebak id yang belum tentu ada; posisi = index yang benar untuk pola gateway ini |
| `sub_app/webapp/src/pages/admin/ingest/IngestUploadPanel.tsx:60-90` | Validasi ekstensi/ukuran juga di klien sebelum request | Umpan balik cepat; SERVER tetap penentu (fail-closed 413/415) |

---
title: spec techstack
description: "Tech stack plugin ingest — baseline existing + komponen baru yang perlu disiapkan"
status: draft
created: "2026-11-03"
---
# Tech Stack — Plugin `ingest` (sesi chat learn-together + mapping editor)

`status: draft` — melengkapi `spec.md`. Prinsip umum: **maksimalkan stack existing**,
tambah dependensi baru hanya bila memang tidak ada padanannya.

## 1. Baseline SUDAH ADA (tidak perlu disiapkan)

| Lapisan | Teknologi | Catatan |
|---------|-----------|---------|
| Backend | Go 1.25 + gin + GORM (postgres/pgx) | Pola plugin `hsh`/`sec`/`visit` langsung dipakai |
| Plugin framework | `internal/plugins` (`ServiceDeps`, registry) + RBAC middleware plugin auth | Seed permission `ingest.*` pola sama |
| Parsing Excel/CSV | `xuri/excelize/v2` (sudah dipakai template generator + validator) | Cukup untuk header/sample + baris |
| Penyimpanan file | `internal/storage` (plugin storage existing) | Untuk file upload mentah |
| Enkripsi/secret | `golang.org/x/crypto` (pola AES-GCM token endpoint) | Secret key deidentify disimpan/diturunkan pola sama |
| Migrasi | SQL migrasi per plugin (`migrations/postgres`) | `ingest_*` tables |
| Swagger | `swag` + build.sh `--parseDependency --parseInternal` | WO terakhir |
| Frontend | React 19 + TypeScript 5.7 + Vite 6 + Tailwind v4 + react-router-dom 7 + axios | Pola UI `secondary/`/`visits/` dipakai |
| RBAC UI | `PermissionGate`, `hasPermission` | Guard halaman ingest |
| Audit | pola endpoint_audit (plugin endpoint) | Aksi agent masuk audit |

## 2. BARU — harus disiapkan (komponen & dependensi)

### 2.1 `deidentify` — github.com/aliengiraffe/deidentify (Go)

- Dependensi Go BARU (`go get`). Deterministik HMAC-SHA256, format-preserving,
  referential integrity.
- Pemakaian: tipe kolom EKSPLISIT dari mapping editor (`Name`/`Phone`/generic);
  TIDAK bergantung inferensi regex bawaan (kekhawatiran pattern AS selesai). (D-4)

### 2.2 LLM client & backend chat — COPILORKIT RUNTIME NODE (D-11 FINAL, 2026-11-04)

> **Revisi besar (keputusan owner, 2026-11-04, D-11 final):** desain awal "proxy
> OpenAI-compatible di Go" DIGANTI — **runtime Node CopilotKit (`@copilotkit/runtime`)
> menjadi backend chat AI penuh**: ia satu-satunya komponen yang memanggil provider
> LLM. Aplikasi Go (bbh app) menjadi **data service murni**. Alasan: CopilotKit v2
> membutuhkan protokol AG-UI + runtime Node WAJIB (spike WO-01a terbukti); memelihara
> proxy Go paralel berarti 2 hop + normalisasi ganda tanpa manfaat.

Arsitektur jalur chat:

1. **CopilotKit UI (React)** → **Runtime Node** (`ingest-runtime`, basePath
   `/api/copilotkit`) → **provider LLM** (env `INGEST_LLM_BASE_URL` /
   `INGEST_LLM_API_KEY` / `INGEST_LLM_MODEL` di Node).
2. **API key TIDAK PERNAH sampai ke klien** — hanya dipegang runtime Node.
3. **Go = data service**: semua aksi berdampak data lewat API plugin `ingest` di Go
   (RBAC + audit + state machine tetap di Go). Runtime Node memanggil API Go untuk:
   - **tool calls** (mapping/confirm/extract/upload) — Node TIDAK pernah menulis
     profil; state machine backend Go tetap penegak tunggal;
   - **payload aman**: sebelum prompt berisi data dikirim ke provider, Node meminta
     Go menyiapkan payload yang sudah dideidentifikasi (deidentify + secret key
     tetap di Go — TIDAK direplikasi ke Node);
   - **audit**: Node mengirim event audit (`ingest_audit` di Go tetap pusat) —
     path/status/provider/durasi, tanpa body/token.
4. RBAC: klien membawa JWT; runtime Node meneruskan token ke API Go untuk tiap
   tool call; endpoint Go fail-closed tanpa token/izin (401/403) — sudah terbukti
   di WO-01a.
5. Dependensi & infrastruktur: `@copilotkit/react-core` + `@copilotkit/runtime`
   (npm), runtime dev via `npm run dev:runtime` (`runtime/copilot-runtime.ts`),
   **service baru `ingest-runtime` di docker-compose (Node)** — N-17 (image
   Go-only) dilonggarkan khusus service ini (D-11).
6. **Sisa Go proxy yang tetap dipakai**: endpoint OpenAI-compatible yang sudah
   dibangun di WO-01a (`POST /api/admin/ingest/llm-proxy/*`) JATUH TEMPO dihapus/
   di-tandai deprecated dari jalur chat — dipertahankan hanya bila berguna untuk
   eksperimen internal non-CopilotKit (keputusan di housekeeping WO-01c).
7. Streaming: AG-UI stream dari runtime Node (SSE) — bukan lagi SSE buatan Go.
8. Identitas: `identifyUser` dari JWT (temuan spike: `/threads` 403 tanpa itu);
   telemetri CopilotKit dimatikan (`COPILOTKIT_TELEMETRY_DISABLED=true`) di
   housekeeping WO-01c.

### 2.3 Classifier gate (pola Jev) — implementasi sendiri (D-3)

- Antarmuka `DecisionClassifier` (noul/choice/score) + implementasi
  **`deterministic`** (aturan/ambang hard-coded untuk dev & uji) + implementasi
  **`llm-json`** (klasifikasi via LLM dengan output JSON terketat) sebagai jembatan;
  implementasi **`typesafe`** opsional kelak.
- Tanpa dependensi LangChain — itu ekosistem Python; kita implement konsepnya di Go.

### 2.4 System prompt & conversation state

- System prompt agent disimpan sebagai **file templat di plugin** (`plugins/ingest/prompt/*.md`
  atau embedded string di Go) — bisa direview/di-edit lewat PR, bukan hard-code di
  tengah handler.
- Riwayat percakapan = tabel `ingest_messages` (persist, O-7); state sesi =
  `ingest_sessions.state` (state machine bagian 4 SPEC).

### 2.5 Mapping editor (React)

- Dibuat sendiri dengan komponen UI existing (`components/ui/*`, Tailwind v4) —
  tanpa library grid/spreadsheet eksternal (D-8).
- Kebutuhan kecil baru di frontend: parser XLSX sisi-klien TIDAK diperlukan —
  header/sample disajikan backend (WO-03), UI hanya merender.

### 2.6 Tabel baru (migrasi plugin `ingest`)

- `ingest_sessions` (state, target_plugin, file_id, branch/instance, timestamps)
- `ingest_messages` (session_id, role, content, meta jsonb)
- `ingest_profiles` (session_id, mapping jsonb, rules jsonb, is_encrypt flags, version)

## 3. Yang SENGAJA TIDAK dipakai

| Teknologi | Alasan |
|-----------|--------|
| LangChain (Python) | Ekosistem salah bahasa; konsepnya diadopsi, implement sendiri di Go |
| MCP server | Ditunda (O-4) — tool-calling internal cukup utk pilot |
| Presidio sidecar Python | Melanggar image Go-only (N-17); opsi lanjut saja (O-11/D-4) |
| Websocket | SSE cukup; lebih sederhana di belakang proxy |
| Library mapping editor / spreadsheet JS | D-8: buat sendiri, komponen kecil |

Catatan revisi: **CopilotKit DIKELUARKAN dari daftar "tidak dipakai"** (2026-11-03)
— sebelumnya dicoret karena topik lama memasangkannya dengan ACP/Jobmanager yang
kompleks; kini dengan proxy OpenAI-compatible, CopilotKit sebagai UI chat menjadi
kandidat realistis (lihat 2.2 poin 9, O-16).

## 4. Ringkasan dependensi baru yang akan masuk go.mod

- `github.com/aliengiraffe/deidentify` (satu-satunya dependensi Go baru yang wajib).
- (opsional kelak) SDK TypeSafe — HANYA bila D-3 direvisi owner.

Frontend: **tidak ada dependensi npm baru** yang wajib.

## 5. Pertanyaan terbuka

- O-13: Provider LLM pertama mana? Usulan: apa pun yang OpenAI-compatible
  (mis. OpenAI / DeepSeek / dll — base URL configurable), agar bisa diganti via env.
  (Mode proxy membuat pilihan ini nyaris bebas — hanya env.)
- O-14: Retensi file upload & payload deidentify di `internal/storage` (hapus setelah
  `extracted`? retensi berapa hari?) — terkait pertanyaan 3 di spec.md bagian 10.
- O-15: Proxy LLM — apakah boleh dipakai klien DI LUAR sesi ingest (mis. developer
  menguji prompt via proxy), atau dikunci khusus sesi ingest aktif? Usulan Manager:
  dikunci khusus sesi ingest aktif (scope ketat, audit bersih).
- O-16: UI chat = **CopilotKit** (React, endpoint ke proxy backend) ATAU klien chat
  sendiri (paket `openai` di atas proxy)? Usulan Manager: CopilotKit bila demo cepat
  & generative UI diinginkan; klien sendiri bila ingin kontrol penuh & dependensi
  minimum. Keduanya tetap di atas proxy — keputusan tidak memengaruhi backend.

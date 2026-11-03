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

### 2.2 LLM client (API eksternal) — abstraksi sendiri + MODE PROXY

- Antarmuka Go sendiri (mis. `services/llm.go`: `Complete(ctx, req) (resp, err)`),
  implementasi pertama: **OpenAI-compatible HTTP API** (chat completions) via
  `net/http` bawaan — tanpa SDK pihak ketiga (jaga image Go-only & kontrol payload).
- Payload yang dikirim SELALU sudah dideidentifikasi (D-2).
- Env baru: `INGEST_LLM_BASE_URL` (default api.openai.com), `INGEST_LLM_API_KEY`,
  `INGEST_LLM_MODEL`. Fail-closed bila tool membutuhkan LLM dan key kosong.
- **MODE PROXY (ide owner, 2026-11-03 — DITERIMA)**: backend juga berfungsi sebagai
  **proxy OpenAI-compatible**. Keputusan teknis:
  1. **API key TIDAK PERNAH sampai ke klien** — klien (UI chat/editor, atau agent
     apa pun) memanggil endpoint proxy di backend; backend menyuntikkan
     `Authorization: Bearer $INGEST_LLM_API_KEY` sebelum meneruskan ke provider.
  2. Titik injeksi: SATU titik (middleware/transport) sehingga tidak ada jalur
     keluar tanpa injeksi — dan setiap request proxy tercatat di audit.
  3. Guard proxy = `ingest.write` (bukan publik); tanpa izin → 403 sebelum
     menyentuh provider.
  4. **Gate deidentify tetap DI DEPAN proxy**: request yang melewati proxy wajib
     sudah dideidentifikasi; proxy menolak payload yang mengandung kolom PII yang
     belum di-deidentify (fail-closed, D-2/D-5) — proxy bukan celah untuk mem bypass
     lapisan privasi.
  5. Manfaat: kunci tidak tersebar di klien; provider bisa diganti via env tanpa
     rebuild UI; rate-limit/budget bisa ditegakkan di satu titik.
  6. Catatan: mode proxy ini jadi cikal-bakal antarmuka agent eksternal kelak
     (kaitan MCP ditunda, O-4) — kontrak proxy dibuat konsisten dengan
     OpenAI-compatible agar adopsi MCP/alat lain tidak menulis ulang.
  7. **Konsekuensi frontend (owner, 2026-11-03): karena proxy-nya OpenAI-compatible,
     di TypeScript (frontend) KITA BISA MEMAKAI BENTUK/SDK OpenAI-compatible** —
     mis. paket `openai` (openai-compatible client) diarahkan ke
     `baseURL: /api/admin/ingest/llm-proxy` dengan `apiKey: "proxy"` (dummy — key
     asli tetap disuntik backend). Artinya: streaming (SSE), format pesan, dan
     tool-calling mengikuti bentuk OpenAI di sisi klien tanpa menulis klien chat
     kustom — SATU dependensi npm yang kini RELEVAN (revisi catatan "tanpa
     dependensi npm baru": boleh 1, hanya client-compat, tanpa SDK provider lain).
  8. Batas: SDK TS HANYA bicara ke proxy internal (bukan langsung ke provider);
     `dangerouslyAllowBrowser`-style konfigurasi dievaluasi di WO terkait — prinsip
     tetap: tidak ada kunci di klien, semua lewat guard RBAC backend.
  9. **CopilotKit kini KEMUNGKINAN (owner, 2026-11-03)**: karena proxy backend
     OpenAI-compatible, **CopilotKit (React) bisa dipakai sebagai UI chat** —
     arahkan endpoint CopilotRuntime ke proxy backend, yang lalu diteruskan ke
     provider LLM. Rantai: CopilotKit UI → proxy (injeksi key + gate deidentify +
     audit) → provider. Catatan keputusan:
     - Manfaat: komponen chat siap-pakai (UI, streaming, generative UI), tidak perlu
       membangun klien chat dari nol; konsisten dengan topik lama 1793096602507 yang
       mencantumkan CopilotKit.
     - Biaya: dependensi baru di frontend (CopilotKit runtime + React paket);
       kontrak CopilotKit agak lebih dari sekadar OpenAI-compatible (runtime GraphQL
       `useCopilotChat`), jadi proxy perlu layer adaptasi kecil.
     - Prinsip tetap: strictness & gate TIDAK dijalankan oleh CopilotKit — tetap di
       backend; CopilotKit hanya lapisan UI. Tool-calling klien DILARANG menulis
       langsung; semua aksi tulis lewat tool API backend yang di-guard RBAC.
     - Status: kandidat KUAT untuk WO-08 (chat UI), keputusan final saat review SPEC
       (O-16). Alternatif: klien chat sendiri memakai paket `openai` di atas proxy.
- **Streaming chat**: pakai **Server-Sent Events (SSE)** dari backend ke UI
  (gin `c.Stream`) — tanpa websocket, cukup untuk chat satu-arah streaming;
  polling fallback bila SSE diblok proxy. Streaming proxy: backend meneruskan
  stream dari provider → SSE ke klien (buffer per chunk, tetap tanpa PII karena
  payload sudah dideidentifikasi).

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

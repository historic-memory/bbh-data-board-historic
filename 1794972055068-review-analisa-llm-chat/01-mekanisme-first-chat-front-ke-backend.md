---
title: Mekanisme first chat dari front ke backend
description: 'Analisa flow LLM chat ingest: sesi baru sampai submit message pertama (front → backend)'
status: complete
created: "2026-11-18"
updated: "2026-11-18"
tags:
    - ingest
    - llm
    - arsitektur
---
# Mekanisme first chat dari front ke backend

## Tujuan

Owner mulai belajar dari awal alur LLM chat plugin ingest — dari sesi baru
sampai submit message pertama — dengan fokus mekanisme **front → backend**.

## Sumber

- Topic induk: `1793711295286` (Ingest data interaktif via chat).
- Kode: `sub_app/webapp/` (SPA), `plugins/ingest/` (Go), `sub_app/ingest_runtime/` (Node).

## Temuan (2026-11-18)

### Arsitektur (D-11)
- **Go** = penegak state machine + satu-satunya penulis data bisnis (RBAC + audit).
- **Runtime Node** tipis: terjemahan model ↔ AG-UI; satu-satunya pemegang API key LLM.
- **Klien** hanya render; tidak mendefinisikan tool.

### 1. Buat sesi baru
- SPA → `POST /api/admin/ingest/sessions` (guard `ingest.write`) → baris `ingest_sessions`, state awal `uploaded`.
- Sesi bisa lahir implisit: pesan pertama memanggil `EnsureSession` (services/sessions.go) — tidak ada "sesi hantu".

test:
api/admin/ingest/sessions/31df95e2-ba62-492d-9b8c-1f59a4f382e4/uploads
api/admin/ingest/sessions/ce211140-b0ad-4b0d-b8f5-0f56018eecf7/uploads

### 2. Upload berkas
- Klien validasi cepat (hanya `.csv`/`.xlsx`, tolak `.xls`, batas ukuran) — **server tetap penentu** (fail-closed 413/415).
- File masuk `internal/storage` + metadata `ingest_uploads`; isi TIDAK diproses di tahap ini.

### 3. Submit message pertama (chat)
- Halaman `/ingest/:id/chat`: CopilotKit di-mount per halaman; `runtimeUrl=/api/copilotkit`.
- Header per request: `Authorization: Bearer <jwt>` + `X-Ingest-Session-Id`.
- Go `handlers.CopilotKit`: reverse proxy ke `INGEST_RUNTIME_URL` (8200), `FlushInterval=-1` untuk SSE/AG-UI, audit tiap request, guard `ingest.write`.
- Runtime Node: factory agent **async per request**; `resolveProvider()` via service-token ke Go (cache 30s sukses / 5s gagal); model dibangun per request (hindari race env global).
- Tool didaftarkan di Node, **EKSEKUSI di Go** (`callGoTool` memakai Bearer milik user) → RBAC + audit tetap di backend.
- Pesan persist via Go `AppendMessage`: idempoten `UNIQUE(session_id, client_id)`, batas 100 KiB, role valid `user|assistant|system|tool`.

### Pengajuan lanjutan
- State machine transition (`/state`, `/transition`).
- Safe-payload deidentify (jalur isi berkas keluar dari Go).

## Praktik debugging (2026-11-18)

- Owner mulai belajar flow lewat `npm run dev` (Vite) di `sub_app/webapp`, akses via LAN `http://192.168.110.103:5173`.
- **Gotcha #1 — URL login**: SPA memakai base `/app/` → halaman login = `/app/login`, BUKAN `/api/admin/login` (itu endpoint JSON POST).
- **Gotcha #2 — 403 saat submit login**: bukan kredensial; ini CORS middleware Go (`internal/app/bootstrap.go:96`). `CORS_ALLOWED_ORIGINS` belum memuat origin LAN → ditambahkan `http://192.168.110.103:5173` di `.env` + restart container `app`. Login LAN terverifikasi 200 via browser.
- **Gotcha #3 — `debugger;` Firefox "tidak dieksekusi"**: DevTools harus open SEBELUM statement jalan; `listIngestUploads` dipanggil di `useEffect` mount halaman chat → hard reload (Ctrl+Shift+R) dengan DevTools terbuka. Alternatif lebih andal: breakpoint manual di panel Debugger. `debugger;` dihapus otomatis pada production build.

---
title: 'WO-01: Muat riwayat chat saat halaman chat dibuka'
description: Halaman /app/ingest/:id/chat menampilkan chat kosong walau pesan tersimpan di ingest_messages — front perlu memuat history saat mount.
status: complete
created: "2026-11-18"
updated: "2026-11-18"
tags:
    - ingest
    - chat
    - frontend
    - copilotkit
---
# WO-01: Muat riwayat chat saat halaman chat dibuka

## Latar belakang

Pesan chat **sudah tersimpan di database** (tabel `ingest_messages` via
`POST /api/admin/ingest/sessions/:id/messages` → `AppendMessage`,
`plugins/ingest/services/sessions.go:57`, idempoten `UNIQUE(session_id, client_id)`).
Endpoint pembacaan juga sudah tersedia:
`GET /api/admin/ingest/sessions/:id/messages` → `ListMessages`
(`plugins/ingest/handlers/sessions.go:61`, route di `plugin.go:79`).

Namun halaman `/app/ingest/:id/chat` (`sub_app/webapp/src/pages/admin/ingest/IngestChatPage.tsx`)
memount `CopilotKitProvider` + `CopilotChat` **tanpa memuat riwayat** —
tidak ada fetch ke endpoint messages saat mount, sehingga chat selalu
tampil kosong seperti sesi baru. Ini gap front-end, bukan kehilangan data.

## Tujuan

Saat membuka halaman chat sesi existing, riwayat pesan (user/assistant)
yang tersimpan di DB tampil kembali di UI chat.

## Lingkungan kerja (scope)

- **Front** (`sub_app/webapp/src/pages/admin/ingest/IngestChatPage.tsx`):
  fetch `GET /api/admin/ingest/sessions/:id/messages` saat mount, lalu
  feed hasilnya sebagai initial/seed messages ke CopilotKit
  (`initialMessages` atau mekanisme set-message yang tersedia di
  `@copilotkit/react-core/v2`).
- **Backend**: tidak ada perubahan — endpoint sudah lengkap
  (`ListMessages` memakai `limit` default 100, urut `created_at DESC, id DESC`
  di services/sessions.go:105; perhatikan urutan tampil di UI harus
  kronologis naik).
- Klien TIDAK mendefinisikan tool; hanya render (D-11).

## Kriteria selesai

1. Buka `/app/ingest/<id>/chat` untuk sesi yang punya pesan lama →
   riwayat user & assistant tampil sesuai urutan kronologis.
2. Sesi tanpa pesan → tetap tampil welcome message, tanpa error.
3. Pesan baru yang dikirim tetap ter-persist (AppendMessage) dan tidak
   ter-duplikasi (client_id idempoten tetap berjalan).
4. Guard permission tetap: fetch memakai Bearer token; endpoint `read`
   (`ingest.read`) tetap di backend.

## Catatan teknis

- `ListMessages` default `limit=100` (maks 200) — putuskan UX jika pesan
  > 100 (pagination sederhana atau tampilkan N terakhir sudah cukup untuk
  iterasi ini).
- Urutan dari API adalah DESC; reverse di klien sebelum render.

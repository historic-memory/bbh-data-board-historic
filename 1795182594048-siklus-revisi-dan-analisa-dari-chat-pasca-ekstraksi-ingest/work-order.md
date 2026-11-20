---
title: Work Order — Siklus revisi & analisa dari chat pasca ekstraksi ingest
description: 'Daftar WO topic 1795182594048: tool transisi state, tool analisa, tombol revise UI, patch prompt agent'
status: progress
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - work-order
    - index
---
# Work Order — Siklus revisi & analisa dari chat pasca ekstraksi ingest

> Master index WO topic `1795182594048`. Semua WO `draft` — menunggu
> persetujuan SPEC oleh owner sebelum eksekusi.

## Daftar WO

| WO | Judul | Status | Commit | Catatan |
|---|---|---|---|---|
| wo-01 | Tool `transition_session` (revise/cancel) di runtime + handler Go | draft | — | proxy `svc.Transition`; hanya `revise|cancel`; audit ingest-tool |
| wo-02 | Tool `analyze_session` di runtime | draft | — | proxy `POST /sessions/:id/analyze`; param `row_no[]` opsional |
| wo-03 | Tombol revise UI + pesan error menyebut revise | draft | — | editor/kartu langkah berikutnya saat `extracted` |
| wo-04 | Patch prompt Kontrak: cek state sebelum simpulan + Kontrak transisi | draft | — | larang laporan bug tanpa uji; wajib `GET state` dulu |
| wo-05 | Bruno E2E siklus penuh + verifikasi | draft | — | extracted→revise→confirm→extract→analyze→fix |

## Konteks kasus nyata

- Sesi `d1f3d30b...` (Resident): terkunci di `extracted`; Manager buka
  via `POST /transition {action:"revise"}` — bekerja (09:25).
- Sesi `873db24a...` (Resident): pola sama; Manager buka + confirm +
  extract via API → batch committed, 49/49 ditolak (format berkas) —
  pembuktian ekstraksi tidak terblokir saat alur benar.
- Klaim "deadlock"/"flag mapping_confirmed tidak sinkron" dari agent
  chat: TERBUKTI SALAH (revise bekerja; kolom flag tidak ada di skema).

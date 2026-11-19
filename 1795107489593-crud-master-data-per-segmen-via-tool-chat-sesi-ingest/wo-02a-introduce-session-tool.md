---
title: "WO-02a — Tool introduce_session (chat murni dari uploaded)"
description: "Backend POST /admin/ingest/tools/introduce_session (pola strict: guard ingest.write + audit ingest-tool, validasi target_plugin fail-closed, hanya dari state uploaded else 409, panggil aksi introduce yang SAMA dengan SaveMapping) + tool Node introduce_session + kontrak prompt (agent WAJIB tanya modul tujuan, dilarang menebak)"
status: pending
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - backend
    - tool
    - chat
---
# WO-02a — Tool `introduce_session` (chat murni dari `uploaded`)

Sumber: temuan review WO-01 (2026-11-20) — gate `introduced` hanya bisa
ditempuh editor (`ActionIntroduce` di `SaveMapping`,
`plugins/ingest/services/mapping.go`), sehingga sesi baru dari
`uploaded` tidak bisa diselesaikan murni via chat. Owner memilih opsi
(a): tool kecil `introduce_session`. Status: **pending**. Dikerjakan
SETELAH WO-02 (file `copilot-runtime.ts` sama — hindari konflik).

## Lingkup

### A. Backend Go

1. `POST /admin/ingest/tools/introduce_session` — pola tool strict:
   guard `ingest.write` + audit `provider='ingest-tool'` (sama dengan
   tool lain WO-01 topic `1795083433686`).
2. Input `{session_id, target_plugin}`:
   - `target_plugin` wajib dari daftar plugin yang memang memuat
     ingest segmen (`visit`, `resident`, `sec`, `hsh` — sumber dari
     daftar target mapping yang sudah ada, fail-closed; di luar → 400).
   - Hanya valid dari state `uploaded` (else 409 dengan pesan state
     sekarang) — KONSISTEN state machine existing.
   - Panggil aksi introduce yang SAMA dengan `SaveMapping`
     (satu sumber kebenaran — JANGAN duplikasi transisi state machine;
     refaktor kecil bila perlu agar aksi bisa dipanggil dua jalur).
3. D-7 tetap: tool ini HANYA memajukan gate identitas/modul — tidak
   menyentuh mapping, profil, atau ekstraksi.

### B. Node (`sub_app/ingest_runtime/copilot-runtime.ts`)

1. Skema tool `introduce_session` — pola existing (JSON Schema →
   `convertJsonSchemaToZodSchema`, Bearer user, `X-Ingest-Session-Id`,
   tanpa logika bisnis). PERHATIAN gotcha WO-01: bila parameternya
   objek bebas, gunakan `loosenFreeFormFields`.
2. Deskripsi tool: gunakan BILA sesi masih `uploaded` dan user belum
   lewat editor; agent WAJIB bertanya modul tujuan (Kunjungan/Data
   Warga/Kebutuhan/Kebutuhan Sekunder) bila belum jelas — DILARANG
   menebak; setelah tool sukses, sambung ke alur mapping biasa.

### C. Prompt agent (`plugins/ingest/prompt/agent.md`)

Kontrak kecil tambahan: sesi dari `uploaded` → tanya modul → panggil
`introduce_session` → lanjut alur mapping; editor tidak lagi satu-satunya
pintu (tetap valid untuk persiapan manual).

## Acceptance Criteria

- AC-1 Sesi BARU: upload via UI → TANPA membuka editor, chat murni dari
  `uploaded` sampai `extracted` (introduce → mapping → confirm →
  extraction) — bukti end-to-end satu sesi uji.
- AC-2 `target_plugin` di luar daftar → 400; panggil saat state bukan
  `uploaded` → 409; tanpa `ingest.write` → 403; tanpa token → 401.
- AC-3 Audit `provider='ingest-tool'` tercatat per pemanggilan.
- AC-4 Editor path TIDAK berubah perilaku (regresi: simpan draft masih
  men-trigger introduce seperti sebelumnya).
- AC-5 `go build/vet/test` + `tsc` + `npm test` runtime + build webapp
  hijau; Bruno minimal (positif + state salah 409 + 401); Log
  Implementasi (Dev) bertahap.

## Non-tujuan

- Perubahan state machine/transisi lain; perubahan `run_extraction`/
  `confirm_mapping`; UI editor.

## Catatan Implementasi (Dev)

(belum ada — isi bertahap per bagian sesuai aturan pinned)

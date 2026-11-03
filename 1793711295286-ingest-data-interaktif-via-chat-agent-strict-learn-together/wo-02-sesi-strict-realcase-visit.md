---
title: WO-02 — Sesi strict + real case `visit` via chat (tanpa editor)
description: "CP-2: state machine sesi, system prompt agent strict, deidentify nyata, classifier gate, ekstraksi visit ke pipeline existing via chat saja"
status: pending
created: "2026-11-03"
---
# WO-02 — Sesi Strict + Real Case `visit` via Chat (Checkpoint 2)

Checkpoint: **CP-2**. Prasyarat: WO-01 (CP-1) selesai + temuan sandbox dipakai sebagai
masukan rancangan ini. Sumber: `spec.md` §4 (state machine), §5 (kontrak strict),
§6.1/§6.3 (komponen), §7 (keamanan); `spec-techstack.md` §2.

## Tujuan

Membuktikan alur **learn-together via chat SAJA** (tanpa mapping editor) pada real
case modul pilot `visit`: file nyata diimpor lewat percakapan dengan agent strict,
lalu hasil ekstraksi masuk pipeline deterministik yang sudah ada.

## Lingkup

### A. State machine sesi + persistensi (backend)

- Migrasi v2: `ingest_sessions` (state, target_plugin, file_id, timestamps) +
  `ingest_profiles` (mapping/rules/flags jsonb, version). `ingest_messages` sudah ada
  di WO-01.
- Transisi state ditandai oleh action API (backend menegakkan); agent tidak bisa
  melompati gate:
  `uploaded → introduced → mapping_draft → mapping_confirmed → extracting → extracted`
  (+ jalur clarify di dalam `mapping_draft`).
- API sesi: create/list/get + transisi (confirm, extract, cancel). Semua guard
  `ingest.*` + audit.

### B. Agent strict via chat

- System prompt agent = file template di plugin (`plugins/ingest/prompt/`) memuat
  5 kontrak (spec §5): learn-together, 5 gate, dispatcher-lite (tanpa editor →
  konfirmasi via chat saja di CP-2), batas wewenang, output terstruktur.
- Tebakan mapping awal dari LLM disajikan di chat sebagai tabel berlabel "belum
  dikonfirmasi"; setiap kolom wajib dikonfirmasi user (atau `skip`).
- Classifier gate (pola Jev, D-3): antarmuka `DecisionClassifier` + implementasi
  `deterministic` (ambang/aturan) untuk gate tool-call: konfirmasi ekstraksi,
  deteksi payload mismatch. Implementasi `llm-json` opsional bila butuh.

### C. Deidentifikasi nyata (D-2/D-4)

- Dependensi `github.com/aliengiraffe/deidentify` masuk go.mod.
- Secret key env baru (pola AES-GCM, fail-closed bila kosong dan ada kolom butuh).
- Kolom ber-flag atau policy wajib → dideidentifikasi SEBELUM payload ke LLM;
  re-identifikasi balik lokal saat pembentukan payload final (deterministik).
- Gate di proxy WO-01 diaktifkan penuh (bukan stub): payload mengandung kolom PII
  yang belum dideidentify → ditolak.

### D. Ekstraksi ke `visit` (pilot, D-1)

- Payload final → pipeline existing plugin `visit`: validate → staging
  (`visit_import_batches/rows`) → import (dedup, `effective_at` history = waktu sumber).
- Pemanggilan via **service layer plugin `visit`** (bukan SQL lintas plugin).
- Laporan hasil di chat: created/updated/skipped/rejected + batch id.
- **Batasan dikenal dari topic visit**: 177 kunjungan tanpa `phone_e164` tidak punya
  kunci dedup — sesi strict harus menanyakan kebijakan duplikasi baris tanpa HP
  ke user (pertanyaan clarify), bukan memutus sendiri.

## Acceptance Criteria

- AC-1 File nyata (ekspor Notion kunjungan, data fiktif/sampel) berhasil diimpor
  end-to-end via chat: laporan created/updated/skipped/rejected tampil + batch id.
- AC-2 Gate identitas: modul target belum dinyatakan user → agent menolak lanjut
  (tertahan state machine, bukan hanya prompt).
- AC-3 Gate mapping: kolom tidak dikonfirmasi → tidak diekstrak (di-skip), bukan ditebak.
- AC-4 Gate aturan: pengecualian validasi hanya aktif bila dinyatakan eksplisit user.
- AC-5 Gate konfirmasi: tanpa pernyataan setuju → aksi ekstraksi terkunci (409/400).
- AC-6 Deidentify: payload ke provider TIDAK memuat nilai PII asli (uji: log/
  mock provider); re-identifikasi mengembalikan nilai asli di DB.
- AC-7 Proxy menolak payload PII yang belum dideidentify (gate WO-01 aktif).
- AC-8 Re-impor file sama → idempoten (dengan kebijakan dedup yang dikonfirmasi user).
- AC-9 RBAC: tanpa `visit.write` → ekstraksi ditolak; audit terisi untuk semua aksi.
- AC-10 `go vet`/`go test` hijau; swagger berisi route sesi; koleksi Bruno untuk
  endpoint sesi (chat e2e manual).

## Non-tujuan

- Mapping editor (CP-3); integrasi editor ↔ chat (CP-4); konfirmasi via editor.

## Risiko & catatan

- Kualitas tebakan LLM untuk header campur ID/EN (bulan, HP kacau) — justru bahan uji
  utama gate; jangan melonggarkan strictness agar "lulus".
- Kebijakan duplikasi tanpa HP wajib diklarifikasi user di sesi (lihat D bagian batasan).
- Payload LLM berisi ±5 baris sample SAJA (bukan seluruh file) selama clarify —
  hemat token & privasi.

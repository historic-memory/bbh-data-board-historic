---
title: WO-02 — Sesi strict + real case `visit` via chat (tanpa editor) [REVISI CP-1]
description: "CP-2 (REVISI): pemisahan runtime (O-19), state machine sesi di Go, agent strict via CopilotKit (D-11), deidentify nyata, classifier gate, compact O-18, ekstraksi visit via service layer"
status: pending
created: "2026-11-03"
---
# WO-02 — Sesi Strict + Real Case `visit` via Chat (Checkpoint 2) — REVISI

Checkpoint: **CP-2**. **CP-1 complete** (WO-01a/b/c — commit `e4f4ea9..397fbff`).
Dokumen ini **DIREVISI 2026-11-04** untuk memakai pembelajaran & keputusan CP-1:
D-11 final (runtime Node = backend chat AI; Go = data service), pola tool WO-01b,
keputusan MCP tidak dipakai, O-18 (compact buat sendiri), O-19 (pemisahan runtime).
Sumber: `spec.md` §4/§5/§6/§7; `spec-techstack.md` §2.2 (revisi); notes O-17/O-18/O-19.

> **⚠ REVISI (2026-11-04).** Bagian yang menyebut "proxy WO-01"/"gate di proxy"
> digantikan arsitektur D-11. Teks lama di bagian C diganti sesuai kenyataan.

## Tujuan

Membuktikan alur **learn-together via chat SAJA** (tanpa mapping editor) pada real
case modul pilot `visit`: file nyata diimpor lewat percakapan dengan agent strict,
lalu hasil ekstraksi masuk pipeline deterministik yang sudah ada.

## Lingkup

### 0. Persiapan struktur (O-19) — kerjakan PERTAMA

- Pindahkan runtime CopilotKit dari `sub_app/webapp/runtime/` ke
  **`sub_app/ingest_runtime/`** (npm workspaces, root `sub_app/`, satu lockfile —
  versi CopilotKit react-core/runtime tetap sinkron).
- Dockerfile.runtime hanya meng-install deps workspace runtime; compose
  `ingest-runtime` tidak berubah perilaku; perbaiki komentar header compose
  ("IDLE, DEV-ONLY spike" usang).
- Verifikasi E2E chat + tool + upload tetap hijau setelah pindah (regresi nol).

### A. State machine sesi + persistensi (backend GO — sumber kebenaran)

- Migrasi v2 (plugin `ingest`): `ingest_sessions` (state, target_plugin, file_id,
  timestamps) + `ingest_profiles` (mapping/rules/flags jsonb, version).
  `ingest_messages` & `ingest_uploads` sudah ada (WO-01).
- Transisi state ditandai oleh **action API Go** (backend menegakkan); agent/Node
  tidak bisa melompati gate:
  `uploaded → introduced → mapping_draft → mapping_confirmed → extracting → extracted`
  (+ jalur clarify di dalam `mapping_draft`).
- API sesi: create/list/get + transisi (confirm, extract, cancel). Semua guard
  `ingest.*` + audit. **Node TIDAK pernah menulis profil** — hanya meneruskan
  tool-call ke API Go (pola WO-01b).

### B. Agent strict via CopilotKit (runtime Node, D-11)

- System prompt agent = file template di plugin (`plugins/ingest/prompt/`) memuat
  5 kontrak (spec §5): learn-together, 5 gate, dispatcher-lite (tanpa editor →
  konfirmasi via chat saja di CP-2), batas wewenang, output terstruktur.
- Tool agent = tool API Go (pola `sandbox_ping`): `get_mapping_guess`,
  `confirm_mapping`, `run_extraction` — didaftarkan di **Go**, Node meneruskan
  (factory per-request), eksekusi di Go dengan guard RBAC + audit. TANPA MCP
  (keputusan WO-01c: tool MCP tidak lewat RBAC/audit → ditolak).
- Tebakan mapping awal disajikan di chat berlabel "belum dikonfirmasi"; kolom
  wajib dikonfirmasi user atau `skip`.
- Classifier gate (pola Jev, D-3): antarmuka `DecisionClassifier` di **Go** +
  implementasi `deterministic` (ambang/aturan) untuk gate tool-call: konfirmasi
  ekstraksi, deteksi payload mismatch.
- **Pertahanan sisi Node**: tambalan `provider-compat.ts` (index `tool_calls`)
  DIPERTAHANKAN sampai gateway diperbaiki — catat rencana penghapusannya
  (jangan jadi tambalan abadi).
- **Compact O-18** (implementasi runtime Node): ukur usage kumulatif per thread;
  threshold 70% via `INGEST_LLM_CONTEXT_LIMIT` (fail-closed default konservatif);
  ringkas pesan lama via LLM; summary disimpan sebagai pesan khusus di
  `ingest_messages` VIA GO; **keputusan terkunci TIDAK ikut diringkas** — profil
  dari Go disuntikkan ulang tiap request; compact tercatat di `ingest_audit`.

### C. Deidentifikasi nyata (D-2/D-4) — di GO (bukan Node)

- Dependensi `github.com/aliengiraffe/deidentify` masuk go.mod (plugin `ingest`).
- Secret key env baru (pola AES-GCM, fail-closed bila kosong dan ada kolom butuh).
- **Alur D-11**: sebelum prompt berisi data dikirim (Node → provider), Node meminta
  Go menyiapkan **payload aman**: Go mengambil sample dari file, menjalankan
  deidentify pada kolom PII (tipe eksplisit dari mapping), mengembalikan payload
  yang sudah anonim; re-identifikasi balik lokal di Go saat pembentukan payload
  final (deterministik). Secret key TIDAK pernah keluar dari Go.
- Gate (pengganti "gate di proxy WO-01" yang sudah dihapus): di titik
  `prepare_safe_payload` Go menolak kolom PII wajib yang belum termapping/
  dideidentify (fail-closed).

### D. Ekstraksi ke `visit` (pilot, D-1)

- Payload final → pipeline existing plugin `visit`: validate → staging
  (`visit_import_batches/rows`) → import (dedup, `effective_at` history = waktu sumber).
- Pemanggilan via **service layer plugin `visit`** (bukan SQL lintas plugin).
- Laporan hasil di chat: created/updated/skipped/rejected + batch id.
- **Batasan dikenal dari topic visit**: 177 kunjungan tanpa `phone_e164` tidak punya
  kunci dedup — sesi strict harus menanyakan kebijakan duplikasi baris tanpa HP
  ke user (pertanyaan clarify), bukan memutus sendiri.

## Acceptance Criteria

- AC-0 Pemisahan O-19 selesai: chat/tool/upload E2E tetap hijau dari struktur baru
  `sub_app/ingest_runtime` (regresi nol).
- AC-1 File nyata (ekspor Notion kunjungan, data fiktif/sampel) berhasil diimpor
  end-to-end via chat: laporan created/updated/skipped/rejected tampil + batch id.
- AC-2 Gate identitas: modul target belum dinyatakan user → agent menolak lanjut
  (tertahan state machine Go, bukan hanya prompt).
- AC-3 Gate mapping: kolom tidak dikonfirmasi → tidak diekstrak (di-skip), bukan ditebak.
- AC-4 Gate aturan: pengecualian validasi hanya aktif bila dinyatakan eksplisit user.
- AC-5 Gate konfirmasi: tanpa pernyataan setuju → aksi ekstraksi terkunci (409/400).
- AC-6 Deidentify: payload ke provider TIDAK memuat nilai PII asli (uji: log/
  mock provider); re-identifikasi mengembalikan nilai asli di DB; secret key
  hanya di Go.
- AC-7 Gate deidentify (di Go, titik `prepare_safe_payload`) menolak payload PII
  yang belum dideidentify (fail-closed).
- AC-8 Re-impor file sama → idempoten (dengan kebijakan dedup yang dikonfirmasi user).
- AC-9 RBAC: tanpa `visit.write` → ekstraksi ditolak; audit terisi untuk semua aksi
  (termasuk tool-call dari Node & event compact).
- AC-10 Compact O-18 bekerja: sesi panjang melewati threshold → summary dibuat,
  disimpan via Go, keputusan terkunci tetap utuh di prompt berikutnya.
- AC-11 `go vet`/`go test` hijau; `tsc`/`npm run build` hijau; swagger berisi route
  sesi; koleksi Bruno untuk endpoint sesi (chat e2e manual).

## Non-tujuan

- Mapping editor (CP-3); integrasi editor ↔ chat (CP-4); konfirmasi via editor;
  MCP (ditolak, WO-01c).

## Risiko & catatan

- **DATA NYATA**: berbeda dari CP-1 (fiktif) — pastikan deidentify hidup SEBELUM
  file asli pertama disentuh; sample ke LLM hanya ±5 baris selama clarify.
- Kualitas tebakan LLM untuk header campur ID/EN (bulan, HP kacau) — justru bahan uji
  utama gate; jangan melonggarkan strictness agar "lulus".
- Kebijakan duplikasi tanpa HP wajib diklarifikasi user di sesi (lihat D bagian batasan).
- Tambalan `provider-compat.ts` adalah ketergantungan eksternal (perilaku gateway)
  — pantau; hapus begitu gateway benar.

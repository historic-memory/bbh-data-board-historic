---
title: "WO-01 — Backend snapshot analisa + tool T-1/T-2/T-4 + fail-closed P-8"
description: "Endpoint analyze (dry-run full scan → snapshot jsonb), tool get_rejected_rows/propose_fixes/submit_analysis (pola agent_tools.go), fail-closed P-8 + batas O-21; listing UI & apply menyusul di WO-02/03"
status: progress
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - backend
    - gate
---
# WO-01 — Backend: snapshot analisa + tool T-1/T-2/T-4 + fail-closed P-8

Sumber: SPEC `spec.md` topic `1795083433686` (disetujui owner
2026-11-19). Status: **progress**. Prasyarat: tidak ada (mulai dari
WO ini).

## Lingkup

### A. Snapshot analisa per sesi (O-25 — persisten ringan)

1. `POST /admin/ingest/sessions/:id/analyze` — dry-run full scan via
   penegak existing (`EnforceMappingRules`) atas seluruh berkas (bukan
   sample 5 baris); input opsional `row_no[]` (scope checkbox P-7).
2. Snapshot = jsonb di plugin ingest (keputusan bentuk: profil baru
   `analysis` atau kolom jsonb per sesi — tanpa tabel relasional baru,
   P-6); isi: daftar pelanggaran per `RowNo`+`Violates`+`Reason`,
   dikompres kelompok per rule id; + flag default checkbox P-8.
3. Snapshot persisten (reload tetap ada) & dibaca ulang oleh T-1.
4. Guard `ingest.write` + audit.

### B. Tool endpoint (pola `handlers/agent_tools.go`)

1. `POST /admin/ingest/tools/get_rejected_rows` (T-1) — snapshot +
   nilai baris terkait AMAN (deidentify) + katalog exceptions segmen +
   flag default P-8. Dua konsumen: agent & UI listing.
2. `POST /admin/ingest/tools/propose_fixes` (T-2) — batch proposal
   `{kind:rule|row}`; rule wajib katalog (`ErrUnknownRule` → 400);
   disimpan sebagai proposal (BELUM menerapkan apa pun) + audit;
   row-fix field wajib kosong tanpa rule → ditolak (P-8 fail-closed).
3. `POST /admin/ingest/tools/submit_analysis` (T-4) — penutup putaran:
   `summary` + `remaining` (ditandai "perlu keputusan manusia");
   snapshot diperbarui; pesan sistem ke chat (pola O-18).
4. Batas O-21: max 500 baris per petikan proposal → 400 di atas itu.
5. Max 3 putaran analisa per sesi → putaran ke-4 wajib submit.

### C. Node

Registrasi skema 4 tool di `copilot-runtime.ts` — pola existing
(JSON Schema → `convertJsonSchemaToZodSchema`, Bearer user, tanpa
logika bisnis). Apply (T-3) ikut WO-03.

## Acceptance Criteria

- AC-1 `analyze` menghasilkan snapshot yang KONSISTEN dengan pesan gate
  (contoh: 144 lenient + 12 required dari berkas uji).
- AC-2 Snapshot persisten — reload/baca ulang T-1 tetap ada (O-25).
- AC-3 `propose_fixes` TIDAK menerapkan apa pun (profil/berkas tidak
  berubah); proposal ter-audit.
- AC-4 Fail-closed: rule di luar katalog 400; row-fix field wajib kosong
  tanpa rule ditolak; >500 baris 400; putaran ke-4 ditolak wajib submit.
- AC-5 Guard RBAC `ingest.write` + audit `provider='ingest-tool'` di
  semua tool; tanpa token 401.
- AC-6 `go build/vet/test` hijau; Bruno untuk 4 endpoint; Node tsc hijau.
- AC-7 **Log Implementasi (Dev)** di dok ini, bertahap per bagian.

## Non-tujuan

- UI listing (WO-02), apply/CSV perbaikan/re-scan (WO-03), blok chat
  (WO-03), perubahan state machine/katalog rule.

## Catatan Implementasi (Dev)

(belum ada — isi bertahap per bagian sesuai aturan pinned)

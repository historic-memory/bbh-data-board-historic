---
title: "WO-01 — Backend snapshot analisa + tool T-1/T-2/T-4 + fail-closed P-8"
description: "Endpoint analyze (dry-run full scan → snapshot jsonb), tool get_rejected_rows/propose_fixes/submit_analysis (pola agent_tools.go), fail-closed P-8 + batas O-21; listing UI & apply menyusul di WO-02/03"
status: complete
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - backend
    - gate
---
# WO-01 — Backend: snapshot analisa + tool T-1/T-2/T-4 + fail-closed P-8

Sumber: SPEC `spec.md` topic `1795083433686` (disetujui owner
2026-11-19). Status: **complete** (dikerjakan 2026-11-19, log di bawah —
lihat "Deviasi" untuk 3 keputusan yang perlu diketahui reviewer).
Prasyarat: tidak ada (mulai dari WO ini).

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

Semua kode fitur ada di `plugins/ingest/**` (tidak menyentuh `internal/`,
aturan proyek). Bukti verifikasi: `go build ./...` ✓, `go vet
./plugins/ingest/...` ✓, `go test ./plugins/ingest/...` ✓, `tsc --noEmit`
(ingest_runtime) ✓, Bruno 58–71 ✓ (36 test), audit & pesan chat diperiksa di DB.

Commit kode: `ba7853c`.

### A. Snapshot analisa

- `plugins/ingest/migrations/postgres/2026_11_20_100000_add_ingest_sessions_analysis.up.sql:1`
  — tambah `ingest_sessions.analysis JSONB` (nullable) + `analysis_rounds
  INTEGER NOT NULL DEFAULT 0` (+ COMMENT). **Kenapa:** O-25/P-6 — snapshot
  persisten ringan tanpa tabel relasional baru; NULL = sesi belum dianalisa.
  Down: `...analysis.down.sql` (drop kebalikan).
- `plugins/ingest/model/session.go:138` — field `Analysis json.RawMessage`
  (tag gorm column `analysis`, type `jsonb`) + `AnalysisRounds`; snapshot
  di-`json:"-"` supaya tidak bocor ke payload sesi umum.
- `plugins/ingest/model/analysis.go:1` — bentuk dokumen snapshot
  (`AnalysisSnapshot/Violation/Group/AnalysisFix/AnalysisProposal/
  AnalysisRemaining`), kelas P-8 `format|completeness`, keputusan
  `rejected|skipped`, `Proposals` (riwayat usulan) + `Summary/Remaining/
  SubmittedAt` (T-4). **Kenapa:** satu dokumen = satu jsonb, dipakai bersama
  T-1/T-2/T-4 (P-6) dan UI listing WO-02.
- `plugins/ingest/services/analysis.go:77` `AnalyzeSession` — dry run SELURUH
  berkas dengan sumber yang SAMA seperti gate (`buildMappingView` + `viewDoc` +
  `EnforceMappingRules`). **Kenapa:** AC-1 — angka snapshot identik dengan
  pesan gate; sengaja tidak ada jalur penegakan kedua yang bisa menyimpang.
  O-21 (scope >500 → `ErrAnalysisTooManyRows`) diperiksa sebelum parse; P-7
  (`rounds >= 3` → `ErrAnalysisRoundLimit`) sebelum menyimpan.
- `plugins/ingest/services/analysis.go:126-146` — analisa ulang MEMPERTAHANKAN
  `Proposals/Summary/Remaining/SubmittedAt` dari snapshot sebelumnya.
  **Kenapa:** T-2/T-4 memakai dokumen yang sama; tanpa ini usulan agent hilang
  saat putaran ke-2 (melanggar P-3: usulan menunggu keputusan manusia).
  Diuji di Bruno 66.
- `plugins/ingest/services/analysis.go:149` `buildAnalysisSnapshot` — total
  `rejected_total/skipped_total` selalu dari SELURUH berkas, `scope` (P-7)
  hanya memfilter daftar baris yang diserahkan ke agent. **Kenapa:** scope
  checkbox user tidak boleh mengubah kebenaran angka gate (AC-1).
- `plugins/ingest/services/analysis.go:274` `analysisKindForRule` — P-8:
  `required`/`warn_when_empty` → `completeness` + default ☐; lainnya → `format`
  + default ☑. `analysis.go:228` `groupAnalysisViolations` mengompresi per
  `violates` dengan batas `MaxAnalysisGroupRows = 200` (risiko SPEC §6:
  jsonb tidak membengkak untuk berkas besar).
- `plugins/ingest/services/analysis.go:300` `saveAnalysis` — nilai jsonb
  dikirim sebagai `string`, bukan `[]byte`. **Kenapa:** pgx memakai codec JSON
  apa adanya (bytea akan ditolak/di-escape), diverifikasi di
  `pgtype/json.go` pada module cache.

### B. Tool T-1 / T-2 / T-4

- `plugins/ingest/services/analysis_tools.go:64` `BuildRejectedRows` (T-1) —
  snapshot + nilai baris AMAN + katalog exceptions + flag P-8; `max_rows`
  default 50, dibatasi 500. Ada guard eksplisit `ErrSegmentRequired` bila
  `target_plugin` sesi kosong. **Kenapa:** `MappingOptions("")` punya default
  legacy `visit` untuk editor; jalur tool TIDAK boleh memakai default itu
  (fail-closed).
- `plugins/ingest/services/analysis_tools.go:183` `safeRowValues` — nilai baris
  dibaca ulang dari berkas ASLI lalu dideidentifikasi (`DeidentifyColumns`),
  kind dari dokumen profil; header yang belum diputuskan user dimask sebagai
  `PIIKindGeneric` (fail-closed, P-5). **Kenapa:** snapshot tidak menyimpan
  PII (tidak ada salinan PII kedua di DB) tetapi agent tetap bisa melihat baris
  yang bermasalah.
- `plugins/ingest/services/analysis_tools.go:283` `ProposeFixes` (T-2) —
  validasi: `rule` wajib ada di katalog (`model.ErrUnknownRule` → 400),
  `action` wajib diizinkan entri katalog (`model.ErrInvalidRule` → 400),
  `row` butuh `row_no` + `values` dengan ref field segmen yang dikenal; batch
  >500 → 400 (O-21). Hasilnya `status: "proposed"` yang disimpan ke snapshot —
  **tidak menerapkan apa pun** (P-3). Diuji Bruno 62/63/64/65 + 63 (versi
  profil tidak berubah).
- `plugins/ingest/services/analysis_tools.go:470` `emptySourceTargets` +
  `analysis_tools.go:343-355`/`:409` — P-8 diperkuat: "field wajib kosong"
  dihitung dari ISI BERKAS (ref target kolom yang selnya kosong) dan digabung
  dengan field katalog. **Kenapa:** katalog menyebut field VALIDASI
  (`visit_visitors.phone_e164`) sedangkan row-fix menyasar field SUMBER
  (`visit_visitors.phone_raw`); tanpa pembacaan berkas, agent bisa
  menyelundupkan nilai karangan lewat nama field yang berbeda. Dijaga unit test.
- `plugins/ingest/services/analysis_tools.go:539` `SubmitAnalysis` (T-4) —
  menyimpan `summary` + `remaining`, `submitted_at`, mereset `analysis_rounds`
  ke 0 (siklus baru), lalu menulis pesan `system` ke chat
  (`appendAnalysisSummaryMessage`, pola O-18) dengan `client_id` deterministik
  (SHA-1 isi) sehingga submit ulang idempoten (UNIQUE session_id+client_id).

### C. HTTP + guard + Node

- `plugins/ingest/handlers/analysis.go:45/85/140/199` — 4 handler dengan
  anotasi swagger, audit `provider='ingest-tool'`, dan error→status via
  `toolStatus`.
- `plugins/ingest/plugin.go:150-153` — 4 rute di balik guard yang sudah ada
  (`RequirePermission(authSvc, "ingest.write")`). **Kenapa:** AC-5 (RBCB):
  semuanya butuh hak tulis; viewer 403/401.
- `plugins/ingest/handlers/agent_tools.go:237-265` — pemetaan error baru:
  `ErrAnalysisNotFound` → 404, `ErrAnalysisRoundLimit` → 409, sisanya
  (unknown/invalid rule, unknown target, batas baris, proposal kosong, bentuk
  fix, P-8, submit kosong) → 400.
- `sub_app/ingest_runtime/copilot-runtime.ts:252/273/320` (+ registrasi di
  `:648/660/672`) — skema 3 tool (`get_rejected_rows`, `propose_fixes`,
  `submit_analysis`) didaftarkan di `createAgent(...).tools` memakai pola
  existing (`convertJsonSchemaToZodSchema`, Bearer user, `X-Ingest-Session-Id`),
  tanpa logika bisnis (kebijakan tetap di Go).
- `plugins/ingest/prompt/agent.md:104` — "Kontrak 6": loop terbatas (maks 3
  putaran, 500 baris/batch, `violates` hanya dari katalog, nilai sudah
  dideidentifikasi, putaran ke-4 wajib `submit_analysis`).

### Deviasi / keputusan yang perlu diketahui reviewer

1. **Node mendaftarkan 3 tool, bukan 4.** `apply_fixes` (T-3) milik WO-03,
   sedangkan `analyze` sengaja BUKAN tool agent: P-7 menyatakan scope baris
   adalah keputusan manusia (checkbox), jadi analisa dipicu UI/Bruno, bukan
   dipilih agent.
2. **Angka contoh di AC-1 (144 lenient + 12 required)** berasal dari berkas
   produksi; berkas uji Bruno dibuat sendiri (6 baris, 2 rejected + 1 skipped)
   karena alasan PII — konsistensinya yang diuji, bukan angkanya.
3. **Snapshot = satu dokumen per sesi** (bukan satu baris per putaran); riwayat
   putaran hanya disimpan sebagai angka `round`. Cukup untuk AC, tetapi bila
   WO-02 perlu riwayat per putaran, bentuk ini yang harus ditinjau.
4. **`AnalysisRounds` tidak di-reset oleh analisa ulang biasa**, hanya oleh
   `submit_analysis` — sesuai P-7 ("putaran ke-4 wajib submit").

### Berkas uji

- `api-tests/fixtures/visit-analysis-sample.csv:1` — header template visit
  dengan 6 baris: HP valid (`+65…`, `08…`), HP berantakan (`06-40961768`,
  landline → lenient), HP kosong (required), tanggal tak terparse
  (`31/13/2026`), waktu kosong (warn_when_empty) → menghasilkan format DAN
  completeness sekaligus.
- `api-tests/ingest/58…71` — rantai: unggah → simpan mapping+rule (allow
  lenient, skip warn_when_empty) → analyze → get_rejected_rows → propose_fixes
  (sukses + rule asing 400 + row-fix P-8 400) → bukti profil tidak berubah →
  putaran 2 & 3 → putaran 4 = 409 → submit_analysis (+ pesan sistem) →
  tanpa token 401 → viewer 403.

### Catatan operasional

- Migrasi dijalankan: `docker compose exec app go run ./cmd/console migrate
  --plugin ingest up` (applied `2026_11_20_100000_add_ingest_sessions_analysis`).
- Server container adalah `bin/server` manual (container idle): setelah
  `go build -o bin/server ./cmd/server`, server dijalankan ulang lalu snapshot
  dibaca lagi (persist ✓ — round 3, rejected 2, skipped 1, proposals 1).


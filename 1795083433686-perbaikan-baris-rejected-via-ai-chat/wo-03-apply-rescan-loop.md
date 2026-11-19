---
title: "WO-03 — Apply fixes + CSV perbaikan + re-scan loop + blok chat rejected_listing"
description: "apply_fixes (T-3): rule → profil append-only; row-fix → CSV perbaikan menggantikan berkas sesi (P-1); re-scan dry-run otomatis; blok terstruktur rejected_listing di chat (pola editor_link WO-04); uji berkas asli 144 lenient end-to-end"
status: complete
created: "2026-11-19"
updated: "2026-11-20"
tags:
    - ingest
    - backend
    - chat
    - loop
---
# WO-03 — Apply fixes + CSV perbaikan + re-scan loop + blok chat

Sumber: SPEC `spec.md` topic `1795083433686`. Prasyarat: WO-01 & WO-02
selesai — TERPENUHI (`ba7853c` + `fc8edbe`, review Manager LULUS).
Status: **complete** (mulai 2026-11-19, selesai 2026-11-20 — lihat
"Catatan Implementasi (Dev)" §8 untuk 6 deviasi yang perlu diketahui
reviewer).

## Lingkup

### A. `apply_fixes` (T-3) — penerapan, dipicu USER

1. Endpoint `POST /admin/ingest/tools/apply_fixes`; `confirmed_by_user:
   true` WAJIB eksplisit (hilang → 400 fail-closed); guard
   `ingest.write` + audit.
2. Per kind:
   - `rule` → validasi ulang katalog (`RuleSet.Validate`) → tambah ke
     profil append-only, `StatedBy=<admin>`, versi baru.
   - `row` → Go bentuk **CSV perbaikan** dari berkas ASLI + nilai baru
     (dari proposal, termasuk `edited_by: user`) → menggantikan berkas
     sesi (mekanisme replace existing; invarian satu berkas — P-1).
3. Setelah apply: **re-scan dry-run otomatis** → snapshot ter-update →
   sisa pelanggaran tampil di listing (O-25) → loop bisa diulang
   (checkbox → Analisa lagi) sampai gate hijau.
4. Batas loop tetap: max 3 putaran analisa (putaran ke-4 wajib submit).

### B. Blok terstruktur chat

1. `rejected_listing` — blok terstruktur dibuat BACKEND Go (pola
   `editor_link` WO-04; bukan teks LLM): link/tombol menampilkan/
   menyorot baris di listing editor.
2. Pesan sistem hasil analisa masuk `ingest_messages` VIA Go (pola
   summary O-18) — agent tidak menulis sendiri.
3. Dua arah: listing → chat (konteks baris) & chat → listing (link).

### C. Node

Skema tool `apply_fixes` di `copilot-runtime.ts` (pola existing,
tanpa logika bisnis). Regresi: 3 tool WO-01 tidak berubah perilaku.

### D. Uji berkas asli end-to-end

Kasus uji pertama: berkas kunjungan owner (144 `phone_raw.lenient_format`
+ 12 `phone_e164.required` + warning) — full loop: analisa → approve →
apply → re-scan → confirm → extract.

## Acceptance Criteria

- AC-1 `apply_fixes` tanpa `confirmed_by_user` → 400; dengan → terapkan
  + audit.
- AC-2 Rule yang di-apply muncul di profil append-only dengan StatedBy
  benar (regresi gate Aturan O-20).
- AC-3 Row-fix yang di-apply menghasilkan CSV perbaikan yang MENGHANTI
  berkas sesi (bukan edit staging DB) — berkas asli tetap ada di riwayat
  storage; snapshot diperbarui.
- AC-4 Re-scan otomatis memperlihatkan pengurangan pelanggaran; loop
  berulang sampai gate hijau; berhenti ≤3 putaran.
- AC-5 Blok `rejected_listing` tampil di chat & link dua arah berfungsi.
- AC-6 Uji berkas asli end-to-end lulus (kasus 144 lenient).
- AC-7 Regresi 4 segmen (visit/resident/sec/hsh) + 3 tool WO-01 tidak
  berubah perilaku; `go build/vet/test` + `tsc` + `npm run build` hijau;
  swagger regenerasi bila ada endpoint baru; Bruno end-to-end.
- AC-8 **Log Implementasi (Dev)** di dok ini, bertahap per bagian.

## Non-tujuan

- Perubahan state machine/katalog rule/tabel relasional; auto-apply
  sistem; retensi snapshot lanjutan (menunggu WO-10 topic induk).

## Catatan Implementasi (Dev)

### 1. Model & asal usulan (§A.2, catatan review WO-02)

`plugins/ingest/model/analysis.go`

- `Origin` (L78–L108) kini **kolom nyata** pada `AnalysisFix` (L126) dan
  `AnalysisProposal` (L138) berisi `agent` | `user`. `NormalizeOrigin()` (L103)
  memetakan nilai bebas dan tetap menerima `"edited_by: user"` sebagai alias
  untuk proposal lama (kompatibilitas data WO-02) — tetapi sumber kebenaran
  baru adalah kolomnya, bukan string di `note`.
- Status proposal (L92–L98): `proposed` → `partially_applied` (sebagian fix
  diterapkan) → `applied` (seluruh fix). `AppliedAt`/`AppliedBy`/`AppliedRows`
  (L142–L144) mencatat siapa menerapkan dan baris apa saja, supaya persetujuan
  bertahap (setujui baris 5 dulu, baris 9 kemudian) terlacak.

### 2. T-3 `apply_fixes` (§A.1–A.3, AC-1/AC-2/AC-3/AC-4)

`plugins/ingest/handlers/analysis_apply.go` (baru) + `plugin.go` L158–L162

- Route `POST /admin/ingest/tools/apply_fixes` di-guard `ingest.write` dan
  diaudit `provider='ingest-tool'` (pola tool lain).
- `applyFixesRequest` memakai `ConfirmedByUser *bool` (L29–L33): pointer supaya
  "tidak dikirim" bisa dibedakan dari "dikirim false", walau keduanya ditolak
  dengan pesan yang sama.
- `services.ErrApplyNotConfirmed` (L49) → 400 fail-closed. Tidak ada jalur
  auto-apply: satu-satunya pemanggil adalah permintaan user (tombol Setujui
  atau perintah chat yang dinyatakan setuju).

`plugins/ingest/services/analysis_apply.go` (baru, 691 baris)

- `ApplyFixes()` (L116) menjalankan urutan: pilih fix (`selectFixes` L250) →
  rule (`applyRuleFixes` L279) → row-fix (`applyRowFixes` L469) → tandai
  proposal (L198–L209) → **re-scan** (L227) → blok chat (L237).
- `selectFixes()` (L250) menegakkan persetujuan per baris/per kategori;
  himpunan kosong = seluruh fix proposal. Bila tidak ada yang cocok →
  `ErrApplyNothingToApply` (L58), bukan diam-diam sukses.
- **Rule** (`applyRuleFixes` L279–L380): divalidasi ULANG ke katalog segmen
  (`RuleSet.Validate`) — id di luar katalog/action yang tak diizinkan ditolak
  (`ErrUnknownRule`/`ErrInvalidRule`), bukan diterapkan mentah. Lalu ditulis
  sebagai **versi baru** dokumen `rules` profil (append-only; versi lama tidak
  diubah — regresi O-20). `StatedBy` = admin dari token (`applyRuleNote`
  L382), `rule_id` deterministik dari `violates` (`ruleIDFor` L400).
- **Row-fix** (`applyRowFixes` L469–L588): membaca berkas **ASLI** dari
  storage, membangun CSV perbaikan dari baris MENTAH seluruh berkas
  (`rawTableRows`), lalu menggantikan berkas sesi lewat `replaceSessionFile`
  (L648). Bukan edit staging DB.
  - Peta kolom diturunkan dari **dokumen mapping** (source ↔ target,
    L505–L516) — **tidak** dari `values[].target` T-1 yang kosong di praktik
    (catatan review WO-02). Ref tanpa kolom sumber → fail-closed
    `ErrAnalysisRepairFailed` (L437) supaya nilai usulan tidak hilang diam-diam.
  - Baris MENTAH dipakai, bukan `table.Rows`: tabel analisa dibatasi
    `maxAnalysisScanRows` (O-21) dan kolomnya dipotong origin — memakai tabel
    itu akan menghilangkan baris/kolom berkas pengganti.
  - `repairFilename()` (L612) melepas sufiks `-perbaikan` lama sebelum
    menambahkannya lagi; apply berulang tidak lagi menumpuk nama
    (`...-perbaikan-perbaikan.csv`). Unit test: `TestRepairFilename`.
  - `repairClientID()` (L591) = hash(proposal + baris) sehingga apply bertahap
    tidak bertabrakan UNIQUE (session_id, client_id) sementara apply identik
    tetap idempotent.
  - Batas 10 MiB (`ErrRepairTooLarge`) menjaga berkas perbaikan tidak melewati
    batas unggah.
- `plugins/ingest/services/analysis_repair_csv.go` (baru): `rawTableRows`,
  `rawHeaderIndex`, `rawDataIndexes` — pembacaan CSV/XLSX mentah dengan aturan
  header yang sama dengan parser pipeline (preamble/baris kosong
  dipertahankan).

### 3. Re-scan tanpa memakan putaran (§A.3, AC-4)

`plugins/ingest/services/analysis.go`

- `RescanAnalysis(sessionID, plugin, round)` (L340) menjalankan ulang dry-run
  penuh berkas memakai **sumber yang sama dengan gate** (`buildMappingView` →
  penegak), lalu menyimpan snapshot baru dengan `round` yang **diteruskan**
  dari pemanggil (L309: `analysis_rounds` sengaja **tidak** disentuh).
  Alasannya di komentar L330–L339: jatah 3 putaran (P-7) adalah jatah **agent
  menganalisa**, sedangkan re-scan adalah tindakan sistem setelah keputusan
  manusia. Konsekuensi yang diuji: `round` tetap sama sebelum/sesudah apply
  (Bruno #73/#75).
- Snapshot lama dipertahankan bagian yang bukan hasil scan (proposals,
  repairs, summary) — `saveAnalysisRaw` (L683) menulis snapshot gabungan,
  sehingga usulan yang belum diterapkan tidak hilang setelah re-scan.

### 4. Blok chat `rejected_listing` (§B.1–B.3, AC-5)

`plugins/ingest/services/analysis_chat.go` (baru) + handler L90–L128

- `buildRejectedListingBlock()` (L64) menyusun blok dari snapshot: angka
  penegak, kelompok + label, `editor_path`, `applied_rules`, `applied_rows`,
  `repair_file_id`. **Dibuat Go, bukan teks LLM** (pola `editor_link` WO-04) →
  angka di chat tidak bisa berbeda dari angka penegak.
- `renderRejectedListingMessage()` (L102) menulis pesan sistem ke
  `ingest_messages` **via Go** (pola O-18), dipanggil `writeRejectedListingBlock`
  (L34) setelah apply.
- `GET /sessions/:id/rejected-listing` (handler L117) mengembalikan blok
  terakhir untuk kartu UI; `GET /sessions/:id/analysis` (handler L90) mengirim
  snapshot + blok + proposal dalam satu tampilan (tanpa nilai sel — P-5).
- Arah dua arah: chat → listing lewat `editor_path` + `?violates=`/`?row=`;
  listing → chat lewat tombol "perbaiki via chat" (WO-02) yang kini memakai
  konteks yang sama.

### 5. Node (§C)

`sub_app/ingest_runtime/copilot-runtime.ts`

- Skema `apply_fixesParameters` (L360–L390): `proposal_id` + wajib
  `confirmed_by_user`, opsional `row_no[]`/`violates[]` untuk persetujuan
  sebagian. Deskripsi tool (L723–L732) menegaskan "HANYA setelah pengguna
  menyatakan setuju" — model tidak boleh mengisi `true` sendiri.
- Tanpa logika bisnis: `callGoTool` meneruskan Bearer user ke Go, jadi RBAC +
  audit tetap di backend (D-11).

### 6. UI (§D, AC-5)

- `sub_app/webapp/src/api/ingest.ts`: `applyIngestFixes()`,
  `getAnalysisView()`, `getRejectedListingBlock()`, tipe
  `ApplyFixesResult`/`AnalysisChatBlock`; `isManualProposal()` kini membaca
  kolom `origin` lebih dulu (fallback `note` hanya untuk data lama).
- `RejectedRowsPanel.tsx`: tombol **Setujui** aktif (per baris / per kategori
  / semua) → `runApply()` mengelompokkan usulan pending per proposal dan
  memanggil T-3 dengan `confirmedByUser: true`; hasil apply menampilkan
  "Diterapkan: N rule · M baris … Sisa pelanggaran: …" dari snapshot re-scan.
  Edit manual kini mengirim `origin: 'user'` (bukan hanya `note`).
- `IngestChatPage.tsx`: kartu `rejected_listing` (L144+) merender blok Go —
  angka, badge `N baris diperbaiki`, tautan kembali ke editor per kategori.

### 7. Verifikasi

- `go build ./...`, `go vet ./plugins/ingest/...`, `go test ./plugins/ingest/...`
  hijau (`analysis_apply_test.go` mencakup: persetujuan wajib, seleksi fix,
  rule id deterministik, nama berkas perbaikan termasuk anti-penumpukan,
  merge baris, normalisasi origin, blok listing, round-trip JSON blok,
  pembacaan baris mentah, penolakan ref tanpa kolom sumber / baris luar
  jangkauan, offset kolom).
- `bash build.sh` di container kanonik: build + swagger OK ("tidak ada
  perubahan isi" untuk docs yang sudah ada; endpoint baru terdokumentasi).
- `npx tsc --noEmit` + `npm run build` hijau.
- Smoke API end-to-end sesi `wo03-smoke-1795104921` (fixture
  `visit-analysis-sample.csv`): analyze (3 ditolak) → propose rule+row →
  apply 400 tanpa konfirmasi → apply 200 (rule masuk profil versi 2, CSV
  perbaikan menggantikan berkas) → re-scan 2 ditolak → apply row-fix → 1 →
  apply rule → **0 ditolak**; berkas `*-perbaikan.csv` berisi nilai koreksi
  dan baris lain utuh.
- Smoke browser Tailscale: panel menampilkan "setujui" per baris + "Setujui
  semua (N)"; klik "Setujui kategori" → notice "Diterapkan: 1 rule ke profil ·
  0 baris… Sisa 1 ditolak"; klik "setujui" baris → "2 baris di berkas
  perbaikan · Sisa 0 ditolak"; kartu chat menampilkan "0 ditolak · 0 dilewati
  · 1 baris diperbaiki" + "Gate aturan sudah hijau".
- **AC-6 (kasus owner 144 + 12)** — fixture owner tidak tersedia di repo, jadi
  dibangun ekuivalen sintetis dengan bentuk sama (144 HP lokal tak
  ternormalisasi + 12 HP kosong, 156 baris): analyze → 156 ditolak (144
  `lenient_format` + 12 `phone_e164.required`) → propose+apply rule
  `lenient_format` → **12** → propose+apply rule `phone_e164.required` → **0**
  (gate hijau) → confirm → extract `dedup_policy=time_only` →
  `created 156, rejected 0`. Deviasi dicatat di §8.
- **Bruno**: 7 request baru (#72–#78). Seluruh koleksi: **176 request, 158
  lulus, 16 gagal, 2 skip** — sama dengan baseline `HEAD` (169 request, 151
  lulus, 16 gagal, 2 skip) setelah diff assertion-per-assertion: **tidak ada
  kegagalan baru**; 16 kegagalan itu sudah ada sebelumnya (butuh token
  viewer/seed dashboard, sesi `bruno-*` yang sudah terhapus, dan batas putaran
  yang di-reset `submit_analysis`). Regresi 4 segmen (visit/residents +
  ingest-resident/sec/hsh) bersih, 3 tool WO-01 tetap lulus.

### 8. Deviasi / catatan penting

- **(a) Berkas owner asli tidak ada di repo.** AC-6 diuji dengan berkas
  sintetis yang mereproduksi bentuk keluhan (144 `lenient_format` + 12
  `phone_e164.required`) dan lulus penuh; berkas asli owner perlu diuji manual
  bila nanti tersedia. Fixture tidak di-commit (dibuat di `/tmp`).
- **(b) Tabel relasional baru: tidak ada.** Semua data tetap di snapshot jsonb
  (O-25) + `ingest_profiles` (append-only) + `ingest_uploads` (berkas aktif +
  riwayat). Tidak ada perubahan state machine maupun katalog rule.
- **(c) Nama berkas perbaikan tidak menumpuk.** Ditemukan saat smoke UI
  (`...-perbaikan-perbaikan-perbaikan.csv`); diperbaiki dengan melepas sufiks
  lama sebelum menambah, plus unit test regresi.
- **(d) `origin` backward-compatible.** `NormalizeOrigin` masih menerima
  `"edited_by: user"` supaya data WO-02 (yang hanya punya string di `note`)
  tetap terbaca sebagai usulan user. UI juga masih memakai `note` sebagai
  fallback terakhir.
- **(e) Batas putaran tetap per siklus.** Bruno #67/#68 gagal karena
  `submit_analysis` me-reset `analysis_rounds`; ini perilaku yang sudah ada
  sejak WO-01 dan bukan perubahan WO-03 (kegagalan yang sama ada di baseline).
  Re-scan apply sengaja **tidak** menyentuh `analysis_rounds`.
- **(f) Temuan environment (bukan kode).** Runtime Node perlu
  `docker restart bbb-data-board-ingest-runtime-1` agar skema `apply_fixes`
  termuat (tsx tidak watch di container dev); server Go perlu di-restart
  setelah `build.sh` agar biner baru dipakai.

### 9. Artefak uji (DB dev)

- `wo03-smoke-1795104921` — fixture visit-analysis-sample, 2× apply (rule +
  2 row-fix), gate hijau.
- `wo03-owner-1795105261` — kasus 144+12; extract `created 156, rejected 0`.
- `wo03-name-1795106117` — verifikasi nama berkas perbaikan tidak menumpuk.

### 10. Berkas yang diubah/dibuat

Backend (baru): `plugins/ingest/services/analysis_apply.go`,
`plugins/ingest/services/analysis_repair_csv.go`,
`plugins/ingest/services/analysis_chat.go`,
`plugins/ingest/handlers/analysis_apply.go`,
`plugins/ingest/services/analysis_apply_test.go`.

Backend (ubah): `plugins/ingest/model/analysis.go`,
`plugins/ingest/services/analysis.go`,
`plugins/ingest/services/analysis_tools.go`,
`plugins/ingest/handlers/agent_tools.go`,
`plugins/ingest/handlers/analysis.go`, `plugins/ingest/plugin.go`,
`plugins/ingest/prompt/agent.md`.

Node: `sub_app/ingest_runtime/copilot-runtime.ts`.

UI: `sub_app/webapp/src/api/ingest.ts`,
`sub_app/webapp/src/pages/admin/ingest/RejectedRowsPanel.tsx`,
`sub_app/webapp/src/pages/admin/ingest/IngestChatPage.tsx`.

Bruno: `api-tests/ingest/72..78-*.bru`.

## Review Manager

**LULUS** (2026-11-20). Spot-check kode `18998f7` (23 berkas, +2.691):

1. AC-1 fail-closed ✓ — `ConfirmedByUser *bool` (pointer: tidak dikirim ≠
   dikirim false, keduanya ditolak); `ErrApplyNotConfirmed` → 400;
   rute di-guard `ingest.write` + audit `provider='ingest-tool'`.
2. AC-2 O-20 ✓ — rule divalidasi ULANG ke katalog (`RuleSet.Validate`)
   sebelum masuk profil append-only sebagai VERSI BARU; `StatedBy` = admin
   dari token; rule_id deterministik dari `violates`.
3. AC-3 P-1 ✓ — row-fix membaca berkas ASLI, CSV perbaikan dari baris
   MENTAH (`rawTableRows`, bukan tabel analisa terpotong O-21), peta kolom
   dari dokumen mapping (bukan `values[].target` yang kosong), ref tanpa
   sumber → fail-closed; `replaceSessionFile` menggantikan berkas sesi.
4. AC-4 ✓ — `RescanAnalysis` TIDAK menyentuh `analysis_rounds` (jatah
   3 putaran = jatah agent; re-scan = tindakan sistem setelah keputusan
   manusia) dan mempertahankan proposals/repairs yang belum diterapkan.
5. AC-5 ✓ — blok `rejected_listing` dibuat Go (pola `editor_link`), pesan
   sistem via Go (O-18), `GET /rejected-listing` untuk kartu UI; Node hanya
   skema + Bearer user; deskripsi tool menegaskan model tidak boleh
   mengisi `confirmed_by_user` sendiri.
6. Kedua catatan review WO-02 dieksekusi ✓ — kolom `origin` nyata
   (`agent|user`, alias lama tetap terbaca) dan peta kolom dari dokumen
   mapping.
7. Temuan & perbaikan sendiri (`repairFilename` anti-penumpukan + unit
   test) disetujui. Deviasi terdokumentasi diterima: AC-6 dengan fixture
   sintetis ekuivalen (156 baris → 0 ditolak → extract 156/0) — berkas
   asli owner diuji manual bila tersedia; Bruno 176 req tanpa kegagalan
   baru vs baseline; env: restart runtime/server sebagai operasi dev.

**Topic `1795083433686` SELESAI** — WO-01..WO-03 complete
(`ba7853c` + `fc8edbe` + `18998f7`). Sisa tindak lanjut: uji berkas asli
owner manual; 2 file kotor pre-existing (`AdminLayout.tsx`,
`src/styles/index.css`) masih perlu kategorisasi (O-22 leftover topic
induk).

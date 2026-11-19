---
title: "WO-02 — Listing UI: tabel rejected + checkbox scope + edit manual + tombol Analisa"
description: "Panel Baris Rejected di IngestMappingPage: checkbox default P-8, edit manual inline, tombol Analisa (N) memicu loop agent, Setujui per baris/kategori/semua, link perbaiki via chat; persisten O-25"
status: complete
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - frontend
    - listing
---
# WO-02 — Frontend: listing UI + checkbox scope + edit manual + tombol Analisa

Sumber: SPEC `spec.md` topic `1795083433686`. Prasyarat: WO-01
(snapshot + tool T-1/T-2/T-4 hidup) — SELESAI (`ba7853c`, review
Manager LULUS). Status: **complete** (mulai 2026-11-19, selesai
2026-11-19 — lihat "Catatan Implementasi (Dev)" §6 untuk 7 catatan/deviasi
yang perlu diketahui reviewer).

## Lingkup

### A. Panel "Baris Rejected" di `IngestMappingPage`

1. Tabel: `☑` checkbox (default dari flag P-8 backend — format ☑ /
   kelengkapan ☐) · RowNo · kolom bermasalah · nilai terdeidentify ·
   `violates` · usulan AI · status (menunggu/diusulkan/diedit user/
   disetujui/perlu keputusan manusia).
2. **Edit manual inline** per baris → masuk proposal yang sama
   (`edited_by: user`) — jalur manual & AI bermuara ke satu proposal store.
3. Tombol **"Analisa (N)"** → simpan scope row_no terpilih ke `analyze`
   → tutup/keluar mapping editor → auto-submit SATU pesan basa-basi ke
   chat AI (mis. "Tolong analisa baris rejected yang barusan
   disimpan") — TANPA data di pesan: session id & snapshot sudah ada
   di sesi, AI mengambil via tool (P-9); loop agent berjalan atas
   baris terpilih (P-7). Pesan single-shot: refresh tidak mengirim ulang.
4. Tombol **"Setujui"** per baris / per kategori / semua →
   `apply_fixes` (T-3, WO-03) dengan `confirmed_by_user`.
5. Tombol **"perbaiki via chat"** per baris/kelompok → buka chat dengan
   konteks baris (blok terstruktur backend, pola `editor_link` WO-04).
6. Regenerasi listing setiap apply/edit (re-scan otomatis, WO-03) —
   snapshot persisten (O-25) → reload tetap ada.
7. PermissionGate `ingest.write` untuk tombol mutasi (pola editor
   existing); RBAC tetap.

## Acceptance Criteria

- AC-1 Listing tampil dari snapshot WO-01 dan persisten (reload tetap).
- AC-2 Checkbox mengontrol scope Analisa; default sesuai flag P-8.
- AC-3 Edit manual inline masuk proposal (`edited_by: user`) dan tampil
  di kolom usulan/status.
- AC-4 Tombol "Analisa (N)" menyimpan scope (analyze), menutup editor,
  lalu chat otomatis terkirim SATU pesan basa-basi (single-shot, tidak
  terkirim ulang saat refresh) dan loop agent berjalan atas baris
  terpilih; hasil proposal muncul di kolom usulan AI.
- AC-5 Tombol Setujui memanggil `apply_fixes` dengan
  `confirmed_by_user` (penerapan diuji penuh di WO-03).
- AC-6 Link "perbaiki via chat" membuka chat dengan konteks baris.
- AC-7 RBAC: viewer tanpa `ingest.write` hanya melihat (tombol mutasi
  tersembunyi); tanpa token 401.
- AC-8 `tsc --noEmit` + `npm run build` hijau; Playwright smoke via
  Tailscale (`http://100.125.175.34:5173`); **Log Implementasi (Dev)**.

## Non-tujuan

- Penerapan proposal (T-3, CSV perbaikan, re-scan) — WO-03.
- Blok chat `rejected_listing` — WO-03 (link tombol boleh dibuat
  statis dulu, disempurnakan saat WO-03).

## Catatan Implementasi (Dev)

### 1. API client — tipe + 3 fungsi tool/analyze (§A.1–A.3, AC-1/AC-3/AC-4)

`sub_app/webapp/src/api/ingest.ts`

- `AnalysisViolation` / `AnalysisGroup` / `AnalysisFix` / `AnalysisProposal` /
  `AnalysisRemaining` / `AnalysisSnapshot` (L577–L661): cermin persis bentuk
  JSONB snapshot O-25 (`groups[]`, `violations[]`, `proposals[]`, `remaining[]`,
  `summary`, `scope`, `round`, `full_scan`). Snapshot tetap **satu dokumen per
  sesi** — UI tidak menormalkan apa pun, hanya membaca.
- `SafeColumnValue` / `RejectedRow` / `RejectedRowsView` (L663–L696): bentuk
  balikan tool T-1 `get_rejected_rows`. `RejectedRowsView` sekaligus membawa
  `exceptions[]` (dipakai untuk label `violates` manusiawi) dan `snapshot`.
- `analyzeIngestSession(sessionId, rowNos = [])` (L698) → `POST
  /sessions/:id/analyze` body `{ row_no }`; `row_no` kosong = seluruh berkas
  (perilaku backend, tidak diakali di UI).
- `getRejectedRows(sessionId, { targetPlugin, maxRows, rowNos })` (L716) → tool
  T-1. **Panel memakai tool yang sama dengan agent** (bukan endpoint UI khusus)
  supaya angka yang dilihat user = angka yang dilihat agent (P-7/P-9).
  `maxRows` dikirim `500` = `MaxAnalysisRows` backend.
- `proposeIngestFixes(sessionId, fixes, note?)` (L739) → tool T-2
  `propose_fixes`.
- `MANUAL_EDIT_NOTE = 'edited_by: user'` (L757) + `isManualProposal()`
  (L760) — lihat deviasi (a).

### 2. Jembatan editor → chat satu-shot (§A.3, P-9, AC-4)

`sub_app/webapp/src/pages/admin/ingest/analysisHandoff.ts` (baru, 100 baris)

- Penanda `ingest:pending-analysis` di **`sessionStorage`**, bukan state React
  (`PENDING_ANALYSIS_KEY` L16): harus bertahan melewati navigasi
  `/map` → `/chat` yang me-remount halaman, tanpa menyentuh backend/DB (tidak
  ada tabel/state machine baru — batasan WO).
- `analysisChatMessage(rowCount)` (L24): pesan basa-basi **tanpa data** —
  tanpa nomor baris, nama kolom, atau nilai sel (P-5 tanpa PII). Angka scope
  hanya untuk kalimat manusia; agent mengambil detail via `get_rejected_rows`.
- `queueAnalysisChat()` (L43): gagal `setItem` (private mode/kuota) tidak
  memblokir navigasi — analisa sudah tersimpan di backend, user tetap bisa
  mengetik manual.
- `takePendingAnalysis()` (L69): **mengambil sekaligus menghapus** penanda dan
  memvalidasi `session_id`; penanda milik sesi lain / rusak tetap dibuang.
  Konsumsi sekali pakai inilah yang membuat refresh tidak mengirim ulang
  (AC-4).

### 3. Panel "Baris Rejected" (§A.1–A.6, AC-1/AC-2/AC-3/AC-6/AC-7)

`sub_app/webapp/src/pages/admin/ingest/RejectedRowsPanel.tsx` (baru, 764 baris)

- **Sumber data & persistensi (AC-1, O-25).** `load()` (L100) memanggil T-1
  `getRejectedRows(..., maxRows: 500)` lalu membaca `snapshot` dari respons —
  tidak ada state listing di frontend. Reload halaman = baca ulang snapshot
  yang sama ⇒ daftar tetap ada. `404` (`ErrAnalysisNotFound`) → `phase='none'`,
  empty-state "belum ada analisa" + tombol Analisa; tidak dianggap error.
- **Checkbox default P-8 (AC-2).** `load()` L114–L120: default per baris
  diambil dari `snapshot.violations[].default_check` — `true` untuk pelanggaran
  **format** (bisa diperbaiki mekanis), `false` untuk **kelengkapan** (butuh
  keputusan manusia). Nilai default datang dari backend, tidak di-hardcode di
  UI. Checkbox per baris `data-testid="ingest-rejected-check-<rowNo>"` (L590),
  plus checkbox "pilih semua" di header grup (L364–L380) untuk seleksi per
  kategori.
- **Scope (AC-2, AC-4).** `checkedRows` (L150) adalah satu-satunya sumber
  scope; `runAnalyze()` (L241) mengirim tepat himpunan itu ke `analyze` — bukan
  "semua baris". Label tombol "Analisa (N)" mengikuti `checkedRows.length`.
- **Kolom tabel** sesuai §A.1: RowNo · kolom bermasalah (`violation.field` →
  header asli lewat `targetHeader`, L81) · nilai terdeidentify (`row.values[]`
  sebagai daftar `Kolom: anon-…`) · `violates` (chip + label exception via
  `violatesLabel`, L91) · usulan AI · status · aksi.
- **Edit manual inline (AC-3).** Form inline per baris (`data-testid=
  "ingest-rejected-edit-<rowNo>"`, L680; simpan `…-save-<rowNo>`, L734).
  `saveManualEdit()` (L257) → T-2 `propose_fixes` dengan `kind:'row'` dan
  `note: 'edited_by: user'`, lalu `load('reload')` supaya baris langsung
  menampilkan usulan terbaru. **Tidak** menerapkan apa pun — usulan menunggu
  WO-03 (juga dinyatakan di teks form: "Usulan TIDAK langsung diterapkan").
  Field yang bisa diedit (`editableRefs`, L541) = `violations[].field` ∩
  `targets[].ref` + ref yang memang ada di `row.values` — lihat deviasi (e).
- **Satu proposal store (AC-3).** `rowProposals` / `ruleProposals` (L161,
  L181) diindeks dari `snapshot.proposals[]` yang sama; usulan AI dan usulan
  manual tampil di kolom yang sama, dibedakan badge lewat `isManualProposal`.
  Fix per baris bersifat *last-wins* (entri terakhir = paling baru), konsisten
  dengan urutan append backend.
- **Usulan level rule.** T-2 juga menerima `kind:'rule'` yang tidak punya
  `row_no`; karena itu `ruleProposals` dipetakan per `violates` dan
  ditampilkan di chip grup ("usulan AI: allow · layout …", L364+) serta sebagai
  *fallback* di kolom "Usulan AI" baris yang `violates`-nya punya rule proposal
  (L645). Tanpa ini, usulan rule AI tidak akan terlihat di UI.
- **Status baris.** `statusOfRow()` (L198): ada proposal manual → `diedit user`;
  ada proposal AI (baris atau rule) → `diusulkan`; ada di `snapshot.remaining[]`
  → `perlu keputusan manusia`; sisanya `menunggu`. `disetujui` disiapkan di
  `STATUS_TONES` (L49) tapi belum dipakai karena penerapan baru ada di WO-03.
- **"Setujui" (AC-5).** Tombol "Setujui kategori" (chip grup) dan "Setujui
  semua (N)" (footer) **dirender tapi `disabled`** dengan jejak eksplisit:
  endpoint `apply_fixes` menyala di WO-03. Batasan WO: "JANGAN buat
  apply/… (WO-03)" — jadi yang dipenuhi sekarang hanya kesiapan UI.
- **"perbaiki via chat" (AC-6).** Link per baris `?row=<rowNo>` dan per
  kelompok `?violates=<code>` ke `/ingest/:id/chat`; parameter sudah disiapkan
  sebagai konteks untuk blok chat WO-03 (Non-tujuan: blok dibuat di WO-03).
- **RBAC (AC-7).** `canMutate = hasPermission(INGEST_WRITE)` (L69) menyembunyikan
  /menonaktifkan semua kontrol mutasi di dalam panel; server tetap sumber
  kebenaran (tanpa token → 401, viewer → 403). Lihat deviasi (d) untuk
  interpretasi gate halaman.
- **Pesan error.** `analysisFailureMessage()` (L757) memprioritaskan teks
  `error` dari server (mis. `ErrAnalysisRoundLimit` 409, `ErrAnalysisRowFixNeedsRule`
  P-8 400) alih-alih teks generik → batas/kendala backend terlihat apa adanya.

### 4. Wiring (AC-1/AC-4)

- `IngestMappingPage.tsx` L26 (import) + L776–L784: panel dirender setelah
  bagian mapping, **hanya bila `view.target_plugin` ada** — sesi tanpa segmen
  belum bisa dianalisa (backend `400 ErrSegmentRequired`), jadi panel tidak
  ditampilkan ketimbang menampilkan error yang tak bisa ditindak. Dibungkus
  `data-testid="ingest-rejected-panel"` untuk smoke. Gate halaman tetap
  `PermissionGate permission={INGEST_WRITE}` (L80) — perilaku lama tidak
  diubah.
- `IngestChatPage.tsx` L25 (import) + L78–L134: `ChatHistoryLoader`
  mengerjakan dua hal sekarang — (1) memuat riwayat lama, menandai
  `historyReady` di `.finally` (L110); (2) **kickoff P-9** (L126–L134) hanya
  setelah `historyReady` supaya pesan analisa berada di **ekor** percakapan
  (urutan kronologis benar). `kicked` ref mencegah dobel-kirim dalam satu
  mount; `takePendingAnalysis()` mencegah kirim ulang antar-mount/refresh.
  Pengiriman memakai pola yang sama dengan chat manual:
  `agent.addMessage({id, role:'user', content})` lalu `copilotkit.runAgent({agent})`.

### 5. Verifikasi

- `npx tsc --noEmit` bersih; `npm run build` hijau (`✓ built in 21.96s`, hanya
  peringatan lama chunk >500 kB).
- Smoke API (Tailscale `http://100.125.175.34:8082`, sesi
  `wo02-smoke-1795097024`): `analyze` 200; T-1 200 (`rejected_total=2`,
  `full_scan=true`, 1 baris dalam scope, grup `default_check=true` format /
  `false` kelengkapan); T-2 manual 200 dan proposal muncul di
  `snapshot.proposals[]` dengan `note: "edited_by: user"`; tanpa token → 401.
- Smoke browser (`http://100.125.175.34:5173/app/ingest/wo02-smoke-1795097024/map`):
  panel tampil dengan ringkasan + chip grup yang kini menampilkan
  `usulan AI: allow · layout 2006-01-02 15:04`; checklist baris 9 → "Analisa (1)";
  edit manual → notice "Perbaikan manual baris 9 tersimpan sebagai usulan",
  baris berubah ke `diedit user` + nilai `2026-11-21 10:00`. Setelah
  `POST /analyze`, editor ditinggalkan dan chat menerima **tepat satu** pesan
  "Tolong analisa 1 baris rejected yang barusan disimpan", agent memanggil
  `get_rejected_rows` → `propose_fixes` → `submit_analysis`.
- Single-shot (AC-4) diverifikasi: reload halaman chat → penanda
  `ingest:pending-analysis` = `null`, pesan hanya ada 1 kali (dari riwayat),
  tidak ada pengiriman baru.

### 6. Deviasi / catatan penting

- **(a) `edited_by` tidak ada di backend.** `AnalysisProposal` Go hanya punya
  `created_by` (id admin, **sama** untuk usulan AI dan manual dari sudut
  pandang tool). Karena batasan WO melarang backend baru, penanda
  `edited_by: user` dibawa di `note` proposal (`MANUAL_EDIT_NOTE`) dan UI
  mendeteksinya lewat `isManualProposal()`. Konsekuensi: penanda bersifat
  konvensi string — bila WO-03 butuh pembeda yang tegas, tambahkan kolom
  `origin` di backend saat apply.
- **(b) Tidak ada suite Playwright di repo** (tidak ada config/spec). Smoke
  dilakukan via HTTP + browser Tailscale manual dan didokumentasikan di §5,
  mengikuti preseden log WO-08.
- **(c) "Setujui" hanya UI.** Dirender `disabled` (AC-5 disiapkan, penerapan
  penuh = WO-03), sesuai batasan "JANGAN buat apply/CSV/re-scan".
- **(d) Interpretasi AC-7.** Halaman mapping sudah lama digerbangi
  `INGEST_WRITE` (§A.7 "pola editor existing; RBAC tetap"), sehingga viewer
  mendapat 403 pada seluruh halaman — bukan hanya tombolnya. Di dalam panel
  setiap mutasi tetap dicek `hasPermission(INGEST_WRITE)`. Server tetap
  menegakkan 401 (tanpa token) / 403 (viewer) — tercakup Bruno #70/#71.
- **(e) `rows[].values[].target` kosong.** `plugins/ingest/deidentify.go`
  (`safeRowValues`) membangun `ColumnInput` tanpa `Target`, jadi field
  `target` pada `values[]` praktis kosong. Panel tidak bergantung padanya:
  ref yang bisa diedit diturunkan dari `violations[].field` ∩ `targets[].ref`
  (plus `values` yang memang membawa target). Tidak diubah di WO-02 (backend
  milik WO-01/tersendiri); dicatat agar WO-03 tahu.
- **(f) Batas putaran bersifat per siklus.** `analysis_rounds` di-reset saat
  `submit_analysis` menutup siklus, sehingga `409 ErrAnalysisRoundLimit`
  ("batas 3 putaran") berlaku per siklus, bukan per sesi. UI tidak menyimpulkan
  sendiri: pesan server ditampilkan verbatim.
- **(g) Temuan environment (bukan perubahan kode).** Saat smoke pertama, pesan
  chat tidak terpersist karena container `bbb-data-board-ingest-runtime-1`
  berjalan sejak sebelum commit WO-01 (`ba7853c`) yang menambahkan
  `attachPersistence`; bind-mount ada di `/src` tetapi `tsx` tidak watch.
  `docker restart bbb-data-board-ingest-runtime-1` → persistensi jalan. Hanya
  tindakan runtime dev, tidak ada perubahan kode WO-02.

### 7. Artefak uji (DB dev)

- Sesi `wo02-smoke-1795097024` (state `uploaded`, `target_plugin=visit`,
  `analysis_rounds=1`).
- Proposal: AI rule `01a1f42b-…`, manual baris `01a1f41f-…` (nilai
  `2026-11-20 10:00`), manual baris `01a1f431-…` (nilai `2026-11-21 10:00`,
  `note: Perbaikan manual baris 9 (edited_by: user)`).

### 8. Berkas yang diubah/dibuat

- `sub_app/webapp/src/api/ingest.ts` (ubah: tipe + `analyzeIngestSession`,
  `getRejectedRows`, `proposeIngestFixes`, `MANUAL_EDIT_NOTE`,
  `isManualProposal`)
- `sub_app/webapp/src/pages/admin/ingest/analysisHandoff.ts` (baru)
- `sub_app/webapp/src/pages/admin/ingest/RejectedRowsPanel.tsx` (baru)
- `sub_app/webapp/src/pages/admin/ingest/IngestMappingPage.tsx` (ubah: render panel)
- `sub_app/webapp/src/pages/admin/ingest/IngestChatPage.tsx` (ubah: kickoff P-9)

---
title: WO-07 — Penanganan sesi `extracting` stale (cancel + delete)
description: Sesi yang state-nya macet di `extracting` (server restart/run terputus) tidak bisa dihapus karena gate fail-closed; sediakan aksi cancel dari `extracting` + guard stale agar delete bisa jalan
status: complete
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - session-lifecycle
---
# WO-07 — Penanganan sesi `extracting` stale

Sumber: temuan owner 2026-11-19 (screenshot sesi `01a1a5d6-…` "CP2 uji
sintetis", state `extracting`, dibuat 04 Nov, tombol Hapus/Hapus permanen
nonaktif) + aturan WO-04a §A ("sesi `extracting` → delete DITOLAK 409 —
jangan menghapus sesi yang sedang bekerja"). Owner menyetujui: **harus ada
penanganan, karena sesi sudah stale**.

## Masalah

- Gate delete fail-closed WO-04a melarang delete dari `extracting` — tepat
  untuk run yang benar-benar berjalan, tetapi **tidak ada jalan keluar** bila
  state macet: run terputus (server restart/dimatikan di tengah ekstraksi)
  dan tidak pernah kembali ke `extracted`.
- Sesi stale seperti itu tidak bisa di-cancel, tidak bisa di-delete,
  selamanya menganggur di daftar. `cancel` di `model/session.go` sudah
  "boleh dari semua state kecuali `extracted`", tapi aksi ini tidak
  terekspose untuk kasus `extracting` di UI (dan backend perlu pembeda
  run-aktif vs stale).

## Lingkup

### A. Backend (plugin `ingest`)

1. **Tanda run** — ekstraksi mencatat waktu mulai run aktif (mis. kolom
   `extracting_at`/`run_started_at` di `ingest_sessions` atau tabel run;
   pilih yang paling sedikit mengubah model — invarian: hanya SATU run
   aktif per sesi). Saat run selesai/gagal, tandanya dikosongkan.
2. **Deteksi stale** — sesi `extracting` dianggap stale bila TIDAK ada run
   aktif (tanda kosong, kasus: server restart) ATAU tanda lebih tua dari
   ambang (mis. 30 menit; nilai bisa dikonfigurasi). Status stale
   diekspos di daftar sesi/state API (UI bisa menampilkan "ekstraksi
   terhenti").
3. **Aksi `cancel` dari `extracting`** — endpoint/bagian state machine:
   `cancel` sah dari `extracting` **hanya bila sesi terdeteksi stale**
   (fail-closed: run benar-benar aktif → tetap ditolak 409). Transisi
   `extracting → cancelled`.
4. **Delete setelah cancel** — sesi `cancelled` bisa di-soft/hard delete
   seperti biasa (WO-04a); tidak perlu aturan baru, cukup dipastikan gate
   tidak menghalangi `cancelled`.
5. Guard `ingest.write` + audit untuk aksi cancel-stale; klien/tool LLM
   tetap tidak bisa (403).

### B. Frontend (webapp, halaman daftar `/ingest`)

1. Aksi **"Batalkan ekstraksi"** pada baris sesi `extracting` yang stale
   (ikon/teks + konfirmasi ringan); sesi run aktif tetap tanpa aksi cancel.
2. Setelah cancel sukses, tombol Hapus/Hapus permanen aktif kembali dan
   badge state berubah `cancelled`.
3. Indikator "ekstraksi terhenti" (stale) di kartu sesi, dari data backend
   — bukan tebakan UI.

## Acceptance Criteria

- AC-1 Sesi `extracting` tanpa run aktif (tanda kosong) terdeteksi stale
  oleh backend; status tampil di API daftar/state.
- AC-2 Sesi `extracting` dengan run aktif (tanda segar) → cancel ditolak
  409; delete juga tetap ditolak 409 (perilaku WO-04a tak berubah).
- AC-3 `cancel` dari `extracting` stale → state `cancelled` (audit +
  RBAC); sesi uji `01a1a5d6-…` bisa dibersihkan dengan jalur ini.
- AC-4 Delete soft/hard pada sesi `cancelled` sukses (regresi WO-04a).
- AC-5 UI: tombol cancel hanya muncul untuk sesi stale; badge "terhenti"
  dari backend; Hapus aktif kembali setelah cancel.
- AC-6 Restart server tidak menyisakan state `extracting` yang tak
  tertangani: setelah restart, run dianggap berakhir (tanda kosong) —
  tidak ada "run aktif" fiktif.
- AC-7 `go build/vet/test` hijau; `tsc --noEmit` + `npm run build` hijau;
  Bruno untuk cancel-stale (200) & cancel-run-aktif (409); swagger.

## Non-tujuan

- Mengubah gate WO-04a untuk delete dari `extracting` langsung — delete
  tetap harus lewat `cancel` dulu (fail-closed dipertahankan).
- Ekstraksi berulang/re-queue otomatis setelah cancel (koreksi manual via
  `revise`/alur normal).
- Resumability run ekstraksi (lanjut dari titik putus).

## Risiko & catatan

- **Definisi stale harus memakai data, bukan tebakan UI** — UI hanya
  merender hasil deteksi backend.
- Ambang waktu stale (mis. 30 menit) cukup longgar untuk run besar
  (berkas asli BBH ±1.600 baris); jangan sampai run sah dicap stale.
- Invarian satu-run-aktif harus tetap berlaku; penanda run jangan bisa
  ganda/berlebihan setelah restart.
- Sesi `01a1a5d6-…` dipakai sebagai kasus uji nyata (bukan fixture).

---

## Rancangan Manager — 2026-11-19 (disetujui owner "harus ada penanganan itu")

| Bagian | Isi | Pemilik dominan |
|---|---|---|
| §A | Penanda run + deteksi stale + cancel dari `extracting` + jalur delete | Dev backend |
| §B | Aksi cancel, indikator terhenti, tombol hapus kembali aktif | Dev frontend |

Urutan eksekusi: §A → §B. Effort kasar: §A ±1 hari; §B ±0,5 hari.

---

## Progres Dev — 2026-11-19

Status: **implementasi §A + §B selesai**, verifikasi hijau. Belum di-commit push
(sesuai instruksi), binari `bin/server` di container sudah dibangun ulang.

### §A Backend (`plugins/ingest`)

- **Migrasi** `2026_11_19_100000_add_ingest_sessions_run_started_at.up.sql`:
  kolom `ingest_sessions.run_started_at TIMESTAMPTZ` + index parsial
  `idx_ingest_sessions_extracting` (state='extracting'). Sudah diterapkan ke DB
  dev (`console migrate up --plugin ingest`, versi 7).
- **Penanda run** (`services/session_state.go`): transisi yang berakhir di
  `extracting` mengisi `run_started_at = now()` — dan HANYA bila proses ini
  memegang run (diklaim `RunExtraction`). State yang meninggalkan `extracting`
  (complete/cancel/rollback/dry-run) mengosongkan penanda.
- **Registry run in-process** (`services/extraction_run.go`): `claimExtractionRun`
  menjamin invarian SATU run aktif per sesi (dua run paralel → 409), dilepas
  lewat `defer`.
- **Deteksi stale** (fungsi murni `model.EvaluateExtraction`, `model/stale.go`):
  `extracting` + penanda kosong → stale (`no_marker`, kasus restart); penanda
  lebih tua dari ambang → stale (`expired`). Ambang dari env
  `INGEST_EXTRACTION_STALE_MINUTES` (default 30 menit, nilai invalid → default).
  Fail-closed: penanda segar = run aktif.
- **Cancel dari `extracting`** (`Transition`): sah HANYA bila stale; run aktif →
  `ErrExtractionRunActive` → 409. `AllowedActionsFor` tidak menawarkan `cancel`
  saat run aktif (UI tidak menampilkan aksi yang pasti ditolak).
- **Sweep boot** (`SweepStaleExtractionRuns`, dipanggil `plugin.RegisterServices`):
  mengosongkan penanda milik proses sebelumnya → tidak ada "run aktif" fiktif
  setelah restart (AC-6). Best-effort (tabel belum dimigrasi → hanya log).
- **API**: daftar sesi & `GET /sessions/:id/state` membawa `extraction`
  (`{stale, reason, run_started_at, threshold_minutes}`). Audit aksi cancel dari
  `extracting` memakai kelas `cancel_stale_extraction`; guard tetap `ingest.write`.
- Delete dari `extracting` TETAP 409 (gate WO-04a tidak dilonggarkan); setelah
  `cancelled`, soft/hard delete jalan normal.

### §B Frontend (`sub_app/webapp`)

- `api/ingest.ts`: tipe `IngestExtractionStatus`; `transitionIngestSession` +
  `cancelIngestExtraction`; `IngestSession.extraction`.
- `IngestSessionListPage.tsx`: badge `ekstraksi terhenti` (danger) hanya dari
  `extraction.stale`; tombol **Batalkan ekstraksi** hanya untuk sesi stale +
  modal konfirmasi ringan; tombol Hapus/Hapus permanen tetap nonaktif selama
  `extracting` (fail-closed) dan aktif lagi setelah `cancelled`.

### Bukti verifikasi

- `go vet ./...` hijau; `go test ./...` hijau (termasuk
  `model.TestEvaluateExtraction`, `services.TestStaleThresholdDefault`,
  `services.TestAllowedActionsForCancelGate`).
- `tsc -b` + `npm run build` (webapp) hijau; swagger di-regenerate
  (`build.sh`, ada `ExtractionStatus`, `sessionStateResponse`).
- Bruno (bru 4.2.1, server lokal):
  - `54` state extracting stale → `extraction.stale=true` (`no_marker`),
    `allowed_actions` memuat `cancel`; `55` cancel → 200 `cancelled`;
    `56` delete sesi cancelled → 200 (AC-1/AC-3/AC-4).
  - `57` cancel dengan penanda SEGAR → **409** `run ekstraksi masih aktif` (AC-2).
- AC-6 diuji langsung: sesi `extracting` ber-penanda segar → restart server →
  penanda dikosongkan saat boot → `stale=true` (`no_marker`).
- Audit: baris `ingest_audit` kelas `cancel_stale_extraction` (200) dan
  `transition_cancel` (409) tercatat.
- UI (browser, Tailscale `http://100.125.175.34:8082/app/ingest`): badge
  "ekstraksi terhenti" + tombol "Batalkan ekstraksi" tampil; klik → modal →
  konfirmasi → sesi hilang dari badge terhenti (state `cancelled`).
- Kasus nyata `01a1a5d6-7780-7369-9754-0c8f9dbc7edd` ("CP2 uji sintetis"):
  dibatalkan lewat `cancel` (state `cancelled`) — sebelumnya macet di
  `extracting` sejak 04 Nov.

### Catatan / batasan

- Invarian satu-run memakai registry memori → berlaku untuk deployment satu
  instance (compose: satu container `app`). Multi-instance perlu kolom
  kepemilikan di DB (belum diperlukan sekarang).
- Transisi `extract` langsung lewat API tanpa run nyata tidak mengisi penanda →
  sesi seperti itu langsung terbaca stale (bukan run aktif fiktif).
- Sesuai instruksi Manager: status dok → `review`, belum push.


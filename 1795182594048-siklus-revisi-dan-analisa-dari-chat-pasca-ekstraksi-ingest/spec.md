---
title: SPEC — Siklus revisi & analisa dari chat pasca ekstraksi ingest
description: 'SPEC draft: tool transition_session (revise/cancel) + analyze_session + tombol revise UI + prompt Kontrak supaya siklus extracted→revise→confirm→extract→analisa bisa dituntaskan murni dari chat (kasus nyata sesi 873db24a & d1f3d30b)'
status: progress
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - ai-chat
    - state-machine
    - spec
---
# SPEC — Siklus revisi & analisa dari chat pasca ekstraksi ingest

> **Status: DRAFT — menunggu review owner.** Belum ada persetujuan;
> WO belum boleh dieksekusi.

Topic `1795182594048`. Sumber: kasus nyata sesi resident
`873db24a-596c-47a8-b824-08c906919909` dan `d1f3d30b-9561-47de-bd89-a41d8cd47f61`
(2026-11-20), ditangani manual oleh Manager via API karena agent chat
tidak punya tool transisi state / pemulai analisa.

## 1. Masalah

Setelah ekstraksi selesai (state `extracted`), agent chat TIDAK punya
jalur resmi untuk:

1. **Membuka kembali mapping** — tool `confirm_mapping(confirm:false)` /
   PUT profil TIDAK memindahkan state; hanya transisi `revise`
   (`extracted → mapping_draft`) yang bisa, tapi tidak diekspos sebagai
   tool. Dua sesi nyata terkunci dan pemilik harus minta bantuan API
   manual.
2. **Memulai analisa baris rejected** — endpoint
   `POST /sessions/:id/analyze` ada, tapi tidak ada tool; Kontrak 6
   mengasumsikan user menekan "Analisa (N)" di editor. Chat tidak bisa
   memulai siklus analisa sendiri.
3. Kejadian turunan: agent chat menyimpulkan "deadlock state machine"
   (klaim salah — mesin state sehat, terverifikasi via API) dan
   membuat laporan bug palsu; prompt belum memerintahkan cek
   `GET /sessions/:id/state` sebelum menyimpulkan.

## 2. Solusi (proposisi — menunggu ok owner)

1. **Tool `transition_session`** — param `action: revise|cancel` SAJA
   (fail-closed; `confirm/extract/introduce/mapping_draft` sudah punya
   tool masing-masing — jangan buka dua pintu untuk aksi yang sama).
   Handler Go proxy ke `svc.Transition` yang sama dengan endpoint
   `POST /sessions/:id/transition` (sudah teruji). Audit otomatis via
   pola tool `provider='ingest-tool'`.
2. **Tool `analyze_session`** — param opsional `row_no[]` (scope baris,
   default semua). Proxy ke endpoint `analyze` existing. Hasil = snapshot
   persisten; agent lanjut Kontrak 6 (`get_rejected_rows` → klasifikasi
   → `propose_fixes` → `apply_fixes`).
3. **Patch UI** — tombol "Buka kembali mapping (revise)" di editor/kartu
   langkah berikutnya saat state `extracted` (backend sudah siap);
   pesan error `ErrMappingLocked` menyebut jalan keluar revise.
4. **Patch prompt agent** — Kontrak baru: wajib cek
   `GET /sessions/:id/state` (atau tool state) sebelum menyimpulkan
   "terkunci/deterministik gagal"; transisi state hanya lewat tool
   transisi, bukan PUT profil; larang membuat laporan bug atas
   deduksi tanpa uji.

## 3. Nontarget (out of scope)

- Tidak mengubah state machine `model/session.go` (sehat, fail-closed,
  terverifikasi).
- Tidak mengekspos `confirm/extract` lewat tool transisi.
- Rule aktif hilang saat simpan draft + peringatan kolom NIK teks di
  template = backlog terpisah.

## 4. Kriteria terima

- Dari chat murni: sesi `extracted` → `transition_session(revise)` →
  `confirm_mapping(true)` → `run_extraction` → `analyze_session` →
  row-fix → tuntas, TANPA bantuan API manual.
- UI menampilkan tombol revise saat `extracted`; error menyebut revise.
- Bruno: kasus positif + negatif (action di luar revise/cancel → 400).

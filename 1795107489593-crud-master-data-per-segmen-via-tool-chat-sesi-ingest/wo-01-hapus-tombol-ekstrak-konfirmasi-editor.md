---
title: "WO-01 — Hapus tombol Ekstrak & Konfirmasi dari editor mapping (aksi jalan via chat AI)"
description: "Hapus tombol 'Ekstrak sekarang' (kartu Ekstrak: dedup + dry run ikut hilang) dan 'Konfirmasi & kunci mapping' di IngestMappingPage — konfirmasi/ekstraksi cukup lewat tool chat AI (confirm_mapping/run_extraction); Simpan draft tetap; prasyarat data dedup/dry-run pindah ke alur chat"
status: pending
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - frontend
    - ux
---
# WO-01 — Hapus tombol Ekstrak & Konfirmasi dari editor mapping

Sumber: owner 2026-11-20 — "sesuai rencana kita pakai tools AI chat".
Topic `1795107489593` (extends `1795083433686`). Status: **pending**.
Patch frontend kecil (pola patch langsung, bukan fitur besar).

## Latar

Editor mapping (`IngestMappingPage.tsx`) masih memuat dua tombol aksi
jalur manual: **"Konfirmasi & kunci mapping"** (`confirm()` →
`confirmMapping`) dan **"Ekstrak sekarang"** (`extract()` →
`extractMapping` dengan pilih dedup + dry run). Sejak tool chat AI
(`confirm_mapping`/`run_extraction`) matang, dua jalur ini redundan dan
mempertahankan dua sumber kebenaran. Owner memutuskan: **hapus dua-duanya**
— aksi konfirmasi/ekstraksi cukup lewat chat AI. Tambahan owner (2026-11-20):
**tombol "Simpan draft" diberi modal prompt untuk kembali ke chat** —
jalur editor adalah persiapan, jalur eksekusi adalah chat.

## Lingkup

### A. Frontend (`sub_app/webapp/src/pages/admin/ingest/`)

1. Hapus tombol **"Konfirmasi & kunci mapping"** di footer
   `IngestMappingPage.tsx` (blok `Button ... confirm()`).
2. Hapus kartu **"Ekstrak"** di sidebar (judul, pilihan kebijakan dedup,
   checkbox dry run, tombol "Ekstrak sekarang") beserta state/handler yang
   hanya dipakai keduanya (`busy 'confirm'|'extract'`, `dedup`, `dryRun`,
   `confirm()`, `extract()`, `gateReady` yang hanya dipakai tombol).
3. **Simpan draft TETAP ADA** — jalur simpan mapping manual tetap dipakai
   (juga dipakai konfirmasi per baris tebakan WO-11).
4. **Modal prompt setelah Simpan draft (owner 2026-11-20)** — sukses simpan
   menampilkan modal (komponen `ui/Modal`, pola Modal hapus existing) dengan
   pilihan: **"Kembali ke chat"** → navigasi `/ingest/:id/chat`, atau
   **"Lanjut edit"** → tetap di editor (modal tutup). Notice success pindah
   isi ke modal (jangan dobel). Simpan draft TIDAK memicu navigasi otomatis
   selain lewat pilihan user (konsisten D-7: keputusan selalu manusia).
5. Perbaiki teks panduan di halaman yang menyebut tombol terkait (mis.
   "Tombol Ekstrak baru aktif setelah semua gate lolos" pada notice
   `confirm()` yang dihapus, dan subtitle kartu Ekstrak) → arahkan ke
   sesi chat AI ("konfirmasi & ekstraksi dilakukan lewat sesi chat").
6. Gate panel (daftar gate) TETAP TAMPIL — masih kebutuhan informasi.
   Panel "Pengecualian (aturan)" tetap (input rule manual tetap valid).

### B. Alur dedup / dry-run (perhatian khusus)

- Pilihan **kebijakan dedup** dan **dry run** sebelumnya ada di kartu
  Ekstrak yang dihapus. Verifikasi alur chat: tool `run_extraction`
  (Node `copilot-runtime.ts` + backend `RunExtraction`) menerima
  `dedup_policy`/`dry_run` dari argumen tool — agent menanyakan user lalu
  mengirim nilai. **Prasyarat**: agent prompt (`agent.md`) sudah
  menginstruksikan konfirmasi kebijakan dedup (checklist topic induk
  AC-4 gate Aturan: 177 baris tanpa HP → agent WAJIB bertanya). Jika
  ternyata belum ada jalur itu, TIDAK BOLEH diabaikan: catat temuan dan
  tambahkan di prompt (kecil), bukan restore tombol.

### C. Verifikasi

- `npx tsc -b` + `npm run build` hijau; tidak ada referensi mati
  (`confirmMapping`/`extractMapping` import dihaluskan — bila tidak dipakai
  lagi di file lain, hapus import).
- Smoke browser Tailscale `http://100.125.175.34:5173`: editor mapping tanpa
  dua tombol; daftar gate + panel rejected + Simpan draft tetap ada; alur
  konfirmasi → ekstraksi lewat chat AI tetap jalan end-to-end (sesi uji
  baru); dry-run via chat terbukti.
- Regresi: alur upload → tebakan → Setujui (WO-11) → analisa (WO-02) tidak
  berubah.

## Acceptance Criteria

- AC-1 Tombol "Konfirmasi & kunci mapping" dan kartu "Ekstrak" (dengan
  dedup + dry run + tombolnya) tidak ada lagi di editor mapping.
- AC-2 Simpan draft tetap ada dan berfungsi (regresi WO-11: Setujui
  tebakan → PUT mapping jalan); sukses simpan menampilkan modal pilihan
  "Kembali ke chat" (navigasi) / "Lanjut edit" (tetap); tidak ada
  navigasi otomatis tanpa pilihan user.
- AC-3 Konfirmasi & ekstraksi end-to-end hanya via chat AI (tool
  `confirm_mapping` + `run_extraction`) — terbukti satu sesi uji penuh.
- AC-4 Kebijakan dedup & dry run tetap bisa dipilih user DI CHAT (agent
  menanyakan; bukan default diam-diam) — bila belum ada, tambah ke prompt.
- AC-5 Gate panel + panel rejected + panel aturan tetap tampil.
- AC-6 `tsc -b` + `npm run build` hijau; smoke browser Tailscale;
  **Log Implementasi (Dev)** di dok ini.
- AC-7 Modal baru memakai komponen `ui/Modal` existing (tanpa dependensi
  baru); keyboard Escape/click-luar menutup modal = "Lanjut edit".

## Non-tujuan

- CRUD master data per segmen (tool M-1..M-4) — WO-02/topic ini, desain
  draf di `notes.md`.
- Perubahan backend/endpoint (`confirm_mapping`/`run_extraction` tetap).
- Menghapus halaman editor atau panel lain.

## Catatan Implementasi (Dev)

(belum ada — isi bertahap per bagian sesuai aturan pinned)

---
title: "WO-03 — Apply fixes + CSV perbaikan + re-scan loop + blok chat rejected_listing"
description: "apply_fixes (T-3): rule → profil append-only; row-fix → CSV perbaikan menggantikan berkas sesi (P-1); re-scan dry-run otomatis; blok terstruktur rejected_listing di chat (pola editor_link WO-04); uji berkas asli 144 lenient end-to-end"
status: pending
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - backend
    - chat
    - loop
---
# WO-03 — Apply fixes + CSV perbaikan + re-scan loop + blok chat

Sumber: SPEC `spec.md` topic `1795083433686`. Prasyarat: WO-01 & WO-02
selesai. Status: **pending**.

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

(belum ada — isi bertahap per bagian sesuai aturan pinned)

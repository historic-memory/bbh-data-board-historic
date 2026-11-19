---
title: "Notes — perbaikan baris rejected via AI chat"
description: "Catatan awal topic turunan: analisa gate rejected ingest + pertanyaan owner (upload/save, listing rejected jadi baris perbaikan via chat AI) — SPEC menyusul setelah klarifikasi"
status: draft
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - gate
    - ai-chat
---
# Notes — Perbaikan baris rejected via AI chat

Topic turunan (extends) dari `1793711295286` — dibuat karena topic induk
sudah terlalu banyak WO (WO-01..11 + WO-04 induk). Analisa & temuan di sini
bersumber dari review Manager/Visual Agent 2026-11-19.

## Latar (temuan owner, 2026-11-19)

Owner menemui **gate rejected** di mapping editor (contoh nyata sesi uji
segmen visit): pesan gate menyebut per-baris + rule id — `baris 279:
visit_visitors.phone_e164.required`, `... dan 11 lainnya`, plus warning
`scheduled_at.warn_when_empty`. Baris melanggar **ditolak** sampai rule
pengecualian ditambahkan (fail-closed, O-20/WO-03).

## Pertanyaan owner (2026-11-19)

1. **Q1 — upload sudah tersimpan di server lalu di-load?**
   **Terjawab (analisa Visual Agent):** ya. `plugins/ingest/services/uploads.go`:
   file divalidasi (fail-closed: hanya `.csv`/`.xlsx`, ≤10 MiB, `.xls`/`.xlsm`/
   `.xlsb` ditolak) → disimpan ke object storage dengan key acak
   `ingest/uploads/<session_id>/<id>.<ext>` (nama asli tidak pernah jadi path
   disk) → metadata di `ingest_uploads` (`stored_key`, size, admin, timestamp;
   metadata gagal → objek dihapus lagi). File **tidak diparse saat upload** —
   parsing & validasi gate membaca ulang file tersimpan saat fase mapping/
   ekstraksi. Sesi satu berkas, state machine `uploaded→…→extracted`.
2. **Q2 — bisa listing rejected jadi baris perbaikan via AI chat (gaya
   vibecoding)?** **Feasible — fondasi ±80% sudah ada:**
   - Rejected rows sudah terdata per-baris di staging
     `*_import_batches`/`*_import_rows` (`row_no`, `valid`, detail field)
     + endpoint audit `GET /admin/<plugin>/import-batches/:id`; pesan gate
     juga membawa row_no + rule id.
   - Chat sesi + agent + provider LLM sudah hidup (topic induk, D-11);
     pola yang pas = **propose–approve**: agent MENGUSULKAN, user
     MENYETUJUI. Gate Aturan fail-closed: rule wajib merujuk
     `exceptions.yaml` plugin target + `StatedBy/StatedAt` = pernyataan
     user — agent TIDAK boleh menciptakan rule sendiri.
   - State `revise` (`mapping_confirmed/extracted → mapping_draft`) sudah
     mendukung siklus perbaiki → confirm ulang.
   - Safety net: dry run + dedup sudah ada.

## Keputusan desain yang harus terjaga (usulan Manager)

1. **Pemisahan dua kategori perbaikan:**
   (a) perbaikan **data** (mis. format HP → E.164) = edit nilai baris staging;
   (b) **pengecualian** (mis. HP memang kosong) = tambah rule
   (`allow`/`skip`) yang mengubah kebijakan gate. AI harus tahu mana yang
   mana; (b) selalu butuh persetujuan user eksplisit.
2. **Auto-fix oleh AI TIDAK menulis langsung** — semua penulisan lewat
   action API Go (RBAC + audit, D-11); Node/klien tidak pernah menulis
   profil/master data.
3. **Dry run wajib** sebelum apply sungguhan (safety net existing).
4. UX usulan: daftar rejected sebagai baris/grup per rule id → tombol
   "perbaiki via chat" → konteks (rule id + row_no + nilai) ke sesi chat →
   AI usulkan nilai baru ATAU rule pengecualian → user approve → apply →
   gate dievaluasi ulang.

## Status

- SPEC & WO **menyusul** (topic baru dibuat khusus agar topic induk tidak
  membesar). Belum ada keputusan owner di atas usulan di atas.
- Terkait tapi terpisah: 144 pelanggaran `phone_raw.lenient_format`
  (isu data, bukan UX) — kandidat kasus uji pertama fitur ini.

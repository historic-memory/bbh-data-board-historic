---
title: 'WO-03: UI edit lengkap + kelola dokumen'
description: Form edit resident menampilkan & mengirim seksi Ibu/Kelahiran/Admisi (prefill + delta), serta aksi hapus/edit dokumen di halaman detail — permission-aware.
status: planned
created: "2026-10-28"
updated: "2026-10-28"
tags:
    - work-order
    - resident
    - ui
    - spa
---
# WO-03: UI edit lengkap + kelola dokumen

## Referensi
- SPEC: A-10..A-12, AC-8, GAP-3.
- `sub_app/webapp/src/api/residents.ts` (tipe `PATCHResidentPayload`, catatan baris ~231-247).
- `sub_app/webapp/src/pages/admin/residents/ResidentFormPage.tsx` (build payload create vs edit).
- `sub_app/webapp/src/pages/admin/residents/ResidentDetailPage.tsx`, `DocumentModal.tsx`.
- WO-01 & WO-02 (kontrak backend) — **selesai lebih dulu**.

## Tugas
1. **`api/residents.ts`**
   - Perluas `PATCHResidentPayload` dengan `mother?: MotherPayload`, `birth?: BirthPayload`,
     `admission?: AdmissionPayload` (mengikuti semantik parsial A-2).
   - Tambah `deleteResidentDocument(id, docId)` (`DELETE .../documents/:docId`) dan
     `updateResidentDocument(id, docId, payload)` (`PATCH`). Perbarui komentar kontrak (hapus catatan lama).
2. **`ResidentFormPage.tsx`** (mode edit)
   - Tampilkan & prefill seksi **Ibu**, **Kelahiran**, **Admisi** dari data resident existing.
   - Build payload **delta**: kirim hanya field yang berubah (hindari menimpa dengan nilai sama).
     Mengosongkan field teks → kirim `""` (backend → `NULL`, A-2).
   - Tombol simpan disabled tanpa `resident.write` (A-11).
   - Tangani `409` (NIK mother duplikat) & `400` dengan pesan ramah (`lib/apiError.ts`).
3. **`ResidentDetailPage.tsx`**
   - Tabel dokumen: kolom aksi **Edit** (pakai `DocumentModal` mode edit → PATCH) & **Hapus**
     (konfirmasi → `deleteResidentDocument`), hanya bila `resident.write`.
   - Setelah sukses: toast + refresh daftar dokumen.

## Catatan UI
- Bahasa Indonesia; identifier kode Inggris.
- Jangan menampilkan URL backend absolut (stateless SPA).
- Komponen mengikuti pola existing (`Modal`, `Button`, `Toast`).

## Kriteria Selesai
- AC-8 terpenuhi.
- `npm run build` (HOST) & `tsc -b` lulus.
- Aksi dokumen tercermin dari API nyata (bukan mock).

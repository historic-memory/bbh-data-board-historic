---
title: 'WO-03: UI edit lengkap + kelola dokumen'
description: Form edit resident menampilkan & mengirim seksi Ibu/Kelahiran/Admisi (prefill + delta), serta aksi hapus/edit dokumen di halaman detail — permission-aware.
status: complete
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

## Catatan Implementasi (2026-10-28)

- `api/residents.ts`: `UpdateResidentPayload` diperluas (`mother`/`birth`/`admission`), tipe
  `UpdateMotherPayload`/`UpdateBirthPayload`/`UpdateAdmissionPayload`, plus
  `updateResidentDocument` (PATCH) & `deleteResidentDocument` (DELETE). Komentar kontrak lama
  ("backend belum punya jalur update") dihapus.
- `ResidentFormPage.tsx`: seksi Ibu/Kelahiran/Admisi dirender untuk create **dan** edit; prefill dari
  agregat (`mother`, `birth`, `admissions[0]` = baris terbaru). Delta dibangun per sub-entitas:
  - ibu ter-link → hanya field berubah; `full_name` kosong **tidak dikirim** (NOT NULL, server
    mengabaikan); ibu belum ada → payload dikirim hanya bila ada isian (nama wajib, divalidasi klien).
  - kelahiran ter-link → blob `birth` hanya bila ada delta; angka kosong tidak dikirim (tidak bisa
    di-NULL-kan); teks kosong dikirim `""` (clear).
  - admisi ter-link → `admitted_at`/`exited_at`/`note`; `exited_at` kosong → `""` (clear).
    Belum ada baris → wajib `admitted_at` (validasi klien + server).
  - Seksi **Status awal tetap create-only** (riwayat append-only, SPEC A-14).
- `DocumentModal.tsx`: prop `document` → mode ubah (judul "Ubah dokumen", submit PATCH diff-only,
  tanpa perubahan = 0 request). `ResidentDetailPage.tsx`: tombol **Ubah**/**Hapus** per dokumen
  (hanya `resident.write`), modal konfirmasi hapus, toast + reload.
- Verifikasi:
  - `tsc -b` + `npm run build` (HOST) lulus, bundle stateless (grep `:8082`/`:5173` kosong).
  - Uji browser nyata terhadap server `:8082` (login root, resident fixture dengan ibu/kelahiran/
    admisi/dokumen):
    - form edit ter-prefill semua seksi; mengubah No. HP ibu → kosong, Ditolong oleh → BIDAN,
      Tanggal keluar → 2026-04-10 menghasilkan **tepat** satu PATCH berisi
      `{"mother":{"phone":""},"birth":{"attendant_type_code":"BIDAN"},"admission":{"exited_at":"2026-04-10"}}`
      (diff-only, tanpa field lain) → detail + API menampilkan nilai baru, phone jadi NULL.
    - modal Ubah dokumen → PATCH `{doc_type,url,label}` → daftar diperbarui; Hapus → konfirmasi →
      DELETE 204 → "0 dokumen" dan API `{"items":[],"total":0}`.
  - Jalur 403 (aktor tanpa `resident.write`) belum teruji end-to-end (H-8, aktor root `*`); guard
    tombol + guard server terpasang.
- Kode: `dd6c753`.

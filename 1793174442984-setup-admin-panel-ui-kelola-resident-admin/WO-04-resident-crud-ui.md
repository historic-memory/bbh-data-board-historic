---
title: "WO-04: UI Kelola Resident (list, detail, create, edit, status, dokumen)"
description: "Halaman resident: tabel dengan pencarian/filter/paginasi, halaman detail (status history + dokumen), form create/edit berbasis mirror fields.yaml, aksi ubah status append-only, dan tambah dokumen link Drive."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - ui
  - resident
---

# WO-04: UI Kelola Resident

## Referensi
- SPEC: A-5, A-11, A-12, A-13, AC-3, AC-4, AC-6.
- Routes: `GET/POST /api/admin/residents`, `GET/PUT/PATCH/DELETE /api/admin/residents/:id`,
  `GET/POST /api/admin/residents/:id/status`,
  `GET/POST /api/admin/residents/:id/documents`,
  `GET /api/admin/mothers`.
- Payload: `CreateResidentInput` / `UpdateResidentInput` (partial, `nil` = unchanged),
  `StatusInput{status_code, effective_at, note}`, `DocumentInput{doc_type, url, label}`.
- Aggregate: `ResidentFull{resident, mother, birth, admissions[], current_status, documents[], branch}`.
  **Catatan:** kunci status agregat = `current_status` (bukan `status`).
- Enum: `plugins/resident/schema/fields.yaml`.

## Langkah

### T-1 — Tipe & mirror schema
1. `src/features/resident/types.ts`: `Resident`, `ResidentFull`, `Mother`,
   `BirthRecord`, `Admission`, `CurrentStatus`, `ResidentDocument`, `BranchRef`,
   input types. Semua sesuai swagger.
2. `src/features/resident/fields.ts`: mirror `fields.yaml` (enums gender, religion,
   blood_type, delivery_method, attendant_type, status, document_type) + label
   Indonesia. Header komentar: **"mirror dari plugins/resident/schema/fields.yaml —
   update bersamaan"** (mitigasi A-12).

### T-2 — List (`/residents`)
- Kolom: `resident_code` (mono), `full_name`, gender, `birth_date` (umur turunan),
  `current_status` (Badge), `branch.label ?? branch.key`, aksi.
- Filter: `q` (debounce 300ms), `branch_key` (dari `GET /api/admin/facilities`
  atau daftar cabang — pakai sumber yang tersedia), `status` (WO-01 B-2).
- Paginasi: page size 25, prev/next berbasis `items.length`; tampilkan `total`
  bila tersedia (A-11).
- State: loading skeleton, empty state ("Belum ada data resident"), error alert
  dengan tombol "Coba lagi".
- Tombol "Tambah Resident" hanya bila `hasPermission('resident.write')`.

### T-3 — Detail (`/residents/:id`)
- Kartu ringkas: kode (read-only), nama, gender, tanggal lahir, cabang.
- Kartu status: status terkini + tombol "Ubah Status".
- Tabel riwayat status (append-only, urut terbaru) + catatan.
- Tabel dokumen: `doc_type` (badge), label, URL Drive (tautan `target="_blank"`,
  `rel="noreferrer"`), tombol "Tambah Dokumen".
- Kartu data ibu, kelahiran, admisi (jika ada).
- Tombol Edit (`resident.write`) & Hapus (`resident.delete`) dengan modal konfirmasi.
- 404 → tampilan "Resident tidak ditemukan" + tautan kembali.

### T-4 — Form create/edit
- Section: Data Bayi (wajib: `full_name`, `gender`, `birth_date`, `branch_key`),
  Data Ibu (opsional), Kelahiran (opsional), Admisi (opsional), Status awal
  (create saja), Dokumen (create: opsional).
- `resident_code`: kosongkan = server generate (WO-10 lama); tampilkan hint
  "Kosongkan agar sistem membuat kode otomatis"; tidak bisa diedit setelah dibuat.
- Validasi client: wajib isi, format tanggal `YYYY-MM-DD` (input `type="date"`),
  NIK 16 digit bila diisi, enum dari `fields.ts`.
- Edit memakai `PATCH` **hanya field yang berubah** (diff vs nilai awal); field
  yang dikosongkan dikirim `""` hanya bila diizinkan backend — verifikasi, kalau
  tidak, nonaktifkan pengosongan.
- Setelah sukses: toast + kembali ke detail/list (refetch).

### T-5 — Ubah status & tambah dokumen
- Modal Ubah Status: `status_code` (select enum), `effective_at` (default sekarang),
  `note` (wajib untuk `TRANSFER` — isi tujuan, mis. "JCH"; lihat D-6).
- Modal Tambah Dokumen: `doc_type`, `url` (wajib validasi `https://`), `label`.
- Refetch detail setelah sukses.

## Kriteria Penerimaan
- [AC-1] List menampilkan data nyata dari API; pencarian `q` dan filter
  `branch_key`/`status` bekerja; paginasi prev/next konsisten.
- [AC-2] Create berhasil dengan field wajib minimum; kode otomatis tampil
  (mis. `BBH-26-P-00x`); kode duplikat → 409 ditampilkan ramah.
- [AC-3] Detail menampilkan `current_status`, riwayat status, dan dokumen.
- [AC-4] Ubah status menambah baris riwayat (append-only) tanpa menghapus yang lama.
- [AC-5] Tambah dokumen menampilkan link Drive yang bisa dibuka.
- [AC-6] PATCH mengirim hanya field berubah; respons 400 (enum/format) tampil
  sebagai pesan per-field bila memungkinkan.
- [AC-7] Viewer tanpa `resident.write`/`resident.delete` tidak melihat tombol
  mutasi; 403 server tetap ditangani.
- [AC-8] Hapus → konfirmasi → sukses → list ter-refresh.
- [AC-9] `npm run build` lulus; tidak ada error TS.

## Bukti
Alur create → detail → ubah status ×2 → tambah dokumen → edit → hapus (screenshot
+ catatan respons API). Uji viewer (403/403) dan error validasi.

## Risiko
- Drift `fields.ts` vs `fields.yaml` (A-12) → wajib komentar + checklist review.
- `UpdateResidentInput` tidak mendukung ubah `resident_code` (memang read-only).

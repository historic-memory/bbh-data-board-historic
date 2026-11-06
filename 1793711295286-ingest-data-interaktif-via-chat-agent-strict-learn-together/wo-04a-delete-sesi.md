---
title: WO-04a — Delete sesi ingest (soft/hard/bulk) — dipindah dari WO-04
description: "Prioritas owner: soft delete default, hard delete ber-konfirmasi, restore, checkbox multi-select + bulk delete di daftar /ingest"
status: pending
created: "2026-11-06"
---
# WO-04a — Delete Sesi Ingest (Checkpoint: patch prioritas sebelum CP-4)

**Latar:** keputusan owner 2026-11-06 — fitur delete sesi dibutuhkan SEKARANG
(daftar sesi mulai penuh saat uji berkas asli). Bagian ini DIPINDAHKAN dari
WO-04 §D dan dieksekusi lebih dulu sebagai patch prioritas. Sisa WO-04
(integrasi editor↔chat, retensi O-14) tetap menjadi CP-4.

Sumber keputusan: percakapan owner 2026-11-06 (soft/hard/restore/checkbox
bulk-delete) — sudah tertulis di WO-04 §D + AC-10; dokumen ini memindahkannya
menjadi lingkup eksekusi mandiri.

## Tujuan

Memberi admin kontrol atas daftar sesi: hapus (soft default), pulihkan, dan
hapus permanen dengan konfirmasi berlapis — tanpa membuka jalur yang
melewati RBAC/audit.

## Lingkup

### A. Backend (plugin `ingest`)

- Migrasi: kolom `deleted_at` (nullable) di `ingest_sessions` + index untuk
  filter daftar.
- **Soft delete** (default): `DELETE /api/admin/ingest/sessions/:id` — set
  `deleted_at`; sesi terhapus hilang dari daftar default (`list` mengecualikan
  `deleted_at IS NOT NULL`).
- **Hard delete**: `DELETE /api/admin/ingest/sessions/:id?mode=hard` (atau aksi
  terpisah — pilih satu, konsisten) — hapus fisik baris `ingest_sessions` +
  `ingest_messages` + `ingest_uploads` + berkas di `internal/storage` milik
  sesi. HANYA dari UI dengan konfirmasi eksplisit.
- **Restore**: `POST /api/admin/ingest/sessions/:id/restore` — kosongkan
  `deleted_at`.
- **Bulk delete (soft)**: `POST /api/admin/ingest/sessions/bulk-delete` — body
  daftar id; respons per-sesi: sukses / gagal / ditolak (state terkunci, mis.
  `extracting` sedang berjalan).
- Semua aksi: guard `ingest.write` + audit (`provider='ingest-delete'` pola
  existing). Klien/tool-calling LLM TIDAK bisa menghapus (403 + audit).
- Sesi ber-state `extracting` → soft/hard delete DITOLAK (409) — jangan
  menghapus sesi yang sedang bekerja.

### B. UI (halaman daftar `/ingest`)

- Checkbox multi-select per baris + aksi "Hapus terpilih" → modal konfirmasi
  menampilkan jumlah sesi terpilih → bulk soft delete; hasil per-sesi
  ditampilkan (sukses/gagal/ditolak).
- Aksi per sesi: "Hapus" (soft, konfirmasi ringan) + "Hapus permanen" (hard,
  modal berlabel RISIKO — khusus sesi `extracted`: peringatan kehilangan
  profil learn-together).
- Sesi terhapus dapat dilihat kembali lewat filter eksplisit ("Tampilkan
  terhapus") + tombol restore.
- Checkbox HANYA di halaman daftar — tidak muncul di chat/editor.

## Acceptance Criteria

- AC-1 Soft delete: sesi hilang dari daftar default, muncul di filter
  "terhapus", restore mengembalikan dengan state utuh.
- AC-2 Hard delete: baris + pesan + upload + berkas storage hilang fisik;
  hanya lewat konfirmasi eksplisit; sesi `extracted` menampilkan peringatan
  risiko profil.
- AC-3 Bulk delete: multi-select → satu endpoint batch; hasil per-sesi
  dilaporkan; sesi `extracting` ditolak 409 tanpa mengganggu sisanya.
- AC-4 RBAC: viewer (tanpa `ingest.write`) → 403 untuk semua aksi delete/
  restore; tanpa token 401; audit terisi untuk setiap aksi (termasuk bulk,
  per sesi).
- AC-5 Tool-calling LLM tidak punya akses delete (tidak ada route tool);
  percobaan → tidak ada jalur.
- AC-6 Regresi nol: daftar/aksi kontekstual WO-01 tidak berubah perilaku;
  `go build`/`vet`/`test` hijau; `tsc`/`npm run build` hijau; swagger memuat
  route baru; Bruno untuk soft/hard/restore/bulk (+ negatif 403/409).
- AC-7 Catatan Implementasi (Dev) diisi sesuai aturan 2026-11-06
  (±10 entri, kolom Kenapa merujuk keputusan).

## Non-tujuan

- Retensi otomatis/job cleanup (O-14) — tetap di WO-4 (CP-4); nanti WAJIB
  memakai jalur penghapusan yang sama (soft → hard policy) supaya tidak
  saling melanggar.
- Integrasi editor↔chat, blok editor_link, generative UI — tetap di WO-04.
- Delete di halaman chat/editor.

## Risiko & catatan

- Hard delete menghapus berkas fisik — pastikan path dari `stored_key` yang
  benar (storage key tidak pernah dikirim ke klien sejak WO-01b).
- Jangan biarkan `list` lama (cache/paginasi) menampilkan sesi terhapus —
  satu query filter di satu tempat.
- Modal konfirmasi hard delete untuk sesi `extracted` WAJIB menyebut profil
  (jangan hanya "yakin?").

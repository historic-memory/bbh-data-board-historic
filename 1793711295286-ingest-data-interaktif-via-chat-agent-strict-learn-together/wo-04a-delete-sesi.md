---
title: WO-04a — Delete sesi ingest (soft/hard/bulk) — dipindah dari WO-04
description: "Prioritas owner: soft delete default, hard delete ber-konfirmasi, restore, checkbox multi-select + bulk delete di daftar /ingest"
status: complete
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

## Progres Implementasi — 2026-11-06 (Dev)

### Bukti AC-1..AC-7

- **AC-1 (soft delete/restore)**: host functional test — soft 200, soft-2x 409,
  sesi hilang dari list default, `include_deleted=1` menampilkan `deleted_at`,
  restore 200, restore-2x 409. Browser: Hapus → notice "dihapus (bisa
  dipulihkan lewat filter terhapus)" → badge `terhapus · <waktu>` + tombol
  "Pulihkan" → notice "dipulihkan".
- **AC-2 (hard delete)**: hard 200 dengan DB 0 baris (sessions/messages/
  uploads/profiles) dan 0 berkas storage; id tidak ada → 404. Browser: modal
  "Hapus permanen sesi" — blok RISIKO "sesi sudah diekstrak … profil
  learn-together" hanya muncul untuk sesi `extracted` (terverifikasi
  `72be67c3-…` 5 pesan extracted), sesi `uploaded` mendapat peringatan
  permanen standar tanpa blok risiko.
- **AC-3 (bulk delete)**: bulk `{deleted:2, rejected:2}` — sesi `extracting`
  ditolak 409 dan duplikat ditolak tanpa mengganggu yang lain. Browser:
  multi-select A+B → "Hapus terpilih (2)" → modal jumlah → confirm → notice
  "Bulk delete (2 sesi): 2 dihapus." + ringkasan per-sesi; bug UI (checkbox
  tidak ter-clear sesudah bulk) ditemukan saat verifikasi browser dan sudah
  diperbaiki (`setSelected(new Set())`).
- **AC-4 (RBAC + audit)**: 401 tanpa token untuk semua jalur; 403 semua aksi
  (delete/restore/bulk) untuk viewer hanya ber-`ingest.read`; audit
  `provider='ingest-delete'` terisi per aksi termasuk bulk (class
  `bulk_delete_<status>` per sesi).
- **AC-5 (tool LLM)**: tidak ada route delete di `/api/admin/ingest/tools/*`
  (plugin.go hanya daftar read/write route); chat/editor tidak punya jalur
  delete.
- **AC-6 (regresi)**: `go build ./...` + `go vet ./...` + `go test
  ./plugins/ingest/...` hijau; `npx tsc --noEmit` + `npm run build` hijau;
  swagger (gitignored `docs/`) diregenerasi di container kanonik via
  `build.sh` — memuat `/{id}` DELETE, `/restore`, `/bulk-delete`,
  `include_deleted`; Bruno suite baru `20–28` 14/14 PASS.
- **AC-7**: Catatan Implementasi (Dev) di bawah ini.

### Catatan Implementasi (Dev)

| Berkas:baris | Yang dilakukan | Kenapa |
| --- | --- | --- |
| `plugins/ingest/migrations/postgres/2026_11_06_100000_add_ingest_sessions_deleted_at.up.sql:12` | Migrasi `deleted_at TIMESTAMPTZ` + index partial `idx_ingest_sessions_active` / `idx_ingest_sessions_deleted` | Keputusan Manager: filter daftar cukup satu kolom nullable + index partial, tanpa enum status baru. |
| `plugins/ingest/model/session.go` (field `DeletedAt *time.Time`) | Tambah `DeletedAt` ke model + summary service | Keputusan Manager: sesi legacy tanpa baris `ingest_sessions` tidak boleh hilang dari daftar — list menyiapkan fallback. |
| `plugins/ingest/services/sessions.go` (`ListSessions(includeDeleted)`) | Filter `deleted_at IS NULL` default, mode `include_deleted` untuk filter "Tampilkan terhapus" | Keputusan Manager: satu query filter di satu tempat (risiko "list lama menampilkan terhapus" tertutup). |
| `plugins/ingest/services/delete.go:74` | `DeleteSession` soft default / `?mode=hard`, sentinel `ErrSessionDeleted/ErrSessionNotDeleted/ErrSessionExtracting` | Keputusan Manager: soft default, hard hanya dari UI ber-konfirmasi, `extracting` → 409. |
| `plugins/ingest/services/delete.go:155` | `hardDelete`: storage dulu (fail-first), pre-check eksistensi → 404, lalu tx hapus uploads/messages/profiles/session | Keputusan Manager: hard = fisik DB + storage; temuan uji (hard id tidak ada sebelumnya 200) diperbaiki agar 404 jujur. |
| `plugins/ingest/services/delete.go:258` | `BulkDeleteSessions` per-sesi (dedup, guard, status per-id) | Keputusan Manager: bulk soft per-sesi — `extracting`/duplikat ditolak tanpa menggagalkan sisanya. |
| `plugins/ingest/handlers/delete.go:33,61,90` | HTTP handler delete/restore/bulk; audit `provider='ingest-delete'`, max 200 id bulk | Keputusan Manager: guard `ingest.write` + audit tiap aksi; bulk diaudit per sesi agar rantai tanggung jawab utuh. |
| `plugins/ingest/handlers/session_state.go` (`sessionErrorStatus`) | Petakan sentinel baru ke 409 | Keputusan Manager: konflik state (sudah terhapus / belum terhapus / extracting) apa adanya 409, bukan 500/404. |
| `plugins/ingest/plugin.go:85-87` | Registrasi 3 route delete; NOL route di `/tools/*` | Keputusan Manager + D-11: tool LLM tidak punya akses delete — route sengaja tidak dibuat. |
| `sub_app/webapp/src/api/ingest.ts:68,80,103` | `deleteIngestSession/restoreIngestSession/bulkDeleteIngestSessions` + `include_deleted` | Keputusan Manager: UI hanya lewat API write; checkbox hanya di daftar (tidak di chat/editor). |
| `sub_app/webapp/src/pages/admin/ingest/IngestSessionListPage.tsx` (rewrite) | Checkbox multi-select, modal bulk ber-jumlah, Hapus/Hapus permanen per sesi, blok RISIKO khusus `extracted`, filter "Tampilkan terhapus" + "Pulihkan" | Keputusan Manager: konfirmasi berlapis + peringatan profil wajib untuk sesi `extracted`; seleksi di-clear setelah bulk (bug ditemukan via verifikasi browser). |
| `api-tests/ingest/20–28-*.bru` (9 berkas, 14 request) | Bruno: soft/restore/hard/bulk + negatif 401/403 + restore-2x 409 | Keputusan Manager: bukti AC-1..AC-6 harus bisa diulang otomatis, bukan hanya screenshot. |

---

## Review Manager — 2026-11-06 (laporan dev diproses, commit kode `9040b39`)

Laporan Dev Agent diterima dan di-review. **WO-04a: COMPLETE** — AC-1..AC-7
semua ✅, verifikasi dua lapis (host functional test + browser), Bruno 14/14,
gate build/test hijau, swagger memuat 4 kontrak baru.

**Catatan review:**
1. **Keputusan desain yang disetujui saat eksekusi** (tercantum di log sebagai
   "Keputusan Manager") dikonfirmasi SAH: index partial, sentinel error →
   409, storage fail-first di hard delete, audit bulk per-sesi, max 200 id
   bulk, NOL route di `/tools/*`.
2. **Hard delete 404 jujur** — perbaikan dari temuan uji (id tidak ada
   sebelumnya balas 200) tepat; fail-first storage sebelum tx DB mencegah
   baris hilang tapi berkas menggantung.
3. **Bug UI bulk (seleksi tidak ter-clear)** ditemukan via verifikasi browser
   sendiri dan diperbaiki — pola verifikasi dua lapis ini yang kita inginkan.
4. **Blok RISIKO khusus `extracted`** (kehilangan profil learn-together)
   terverifikasi tampil hanya untuk sesi itu; `uploaded` dapat peringatan
   standar — sesuai AC-2.
5. **Cleanup sesi uji** (hard-delete via API, cancel sesi extracting, hapus
   file temp repo) — baik; daftar produksi tidak tertinggal sampah uji.
6. Catatan kecil (non-blocking): `include_deleted` di list — pastikan UI
   filter terhapus tidak ikut masuk paginasi default pengguna baru; sudah
   tertutup karena default `deleted_at IS NULL`.

**Konsekuensi master index:** WO-04a complete; patch prioritas ditutup.
Sisa menuju penyelesaian fitur ingest: **WO-04 (CP-4)** — prasyarat tinggal
keputusan O-14 (retensi) + verifikasi ulang percakapan UI end-to-end
(catatan review §0.1).

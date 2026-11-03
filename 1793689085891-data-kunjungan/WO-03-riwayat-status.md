---
title: WO-03 — Riwayat status kunjungan (append-only)
description: Endpoint riwayat status visit append-only + status terkini, pola resident_status_history.
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-03 — Riwayat status kunjungan (append-only)

**Prasyarat:** WO-02.

## Lingkup

1. `POST /api/admin/visit/visits/:id/status` (guard `visit.write`) — append baris `visit_status_history` (status_code dari enum, note, changed_by = aktor). Transaksional dengan update `visits.status_code`.
2. `GET /api/admin/visit/visits/:id/status` (guard `visit.read`) — daftar riwayat urut `effective_at` asc.
3. `visits.status_code` = status TERAKHIR (pola LATERAL/terbesar `effective_at`) — konsisten dgn `current_status` resident.
4. Aturan transisi: tidak ada pembatasan ketat di MVP, tapi `note` wajib untuk `CANCELLED` (alasan) — validasi 400 bila kosong (paralel dgn aturan TRANSFER resident).
5. Test DB: append 3 status → urutan & status terkini benar; note CANCELLED wajib.

## Acceptance Criteria

- AC-1: curl — tambah 2 status → history 3 baris (termasuk awal), `status_code` visit = terakhir.
- AC-2: status tidak ada di enum → 400; CANCELLED tanpa note → 400.
- AC-3: viewer write → 403.

## Bukti

- Kode commit `fd932bd` (5 file, +343).
- Endpoint: `POST /api/admin/visit/visits/:id/status` (guard `visit.write`, 201 `model.StatusHistory`) dan `GET /api/admin/visit/visits/:id/status` (guard `visit.read`, `{items,total}`, urut `effective_at ASC`). Visit tak dikenal → 404.
- Append dalam satu transaksi; setelah insert, `visits.status_code` disinkronkan ke baris dengan `effective_at` terbesar (bukan sekedar status yang baru dikirim) — backfill bertanggal lama TIDAK menggeser status terkini.
- `changed_by` diisi dari admin login (`authmiddleware.Admin(c)`), `effective_at` opsional (RFC3339 / `YYYY-MM-DD`, default sekarang).
- AC-1: `DONE` (note) lalu `CANCELLED` (note) → `GET status` total 3: `SCHEDULED` → `DONE` → `CANCELLED`; `GET visit` → `status_code=CANCELLED`, `current_status=CANCELLED`; DB 3 baris.
- AC-2: `CANCELLED` tanpa note → 400 (`note wajib untuk status CANCELLED`); `TIDAK_DATANG` → 400 (`status tidak dikenal`); note blank (`"  "`) juga 400.
- AC-3: viewer `GET status` 200, `POST status` 403; tanpa token 401.
- Test DB RUN: 3 baris urut + status terkini, `changed_by` tersimpan, backdated row tidak menang, validasi enum/note tidak menulis baris tambahan, visit tak dikenal → ErrNotFound.
- Data uji dibersihkan (0 visitors/0 visits/0 history); akun viewer uji dihapus. **Catatan untuk WO-06**: importer harus menulis baris history awal dengan `effective_at` = waktu sumber (bukan waktu impor), kalau tidak status historis akan tertimpa "sekarang".
- Tidak ada push.

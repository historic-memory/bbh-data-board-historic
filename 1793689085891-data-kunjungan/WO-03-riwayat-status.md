---
title: WO-03 — Riwayat status kunjungan (append-only)
description: Endpoint riwayat status visit append-only + status terkini, pola resident_status_history.
type: work-order
status: pending
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

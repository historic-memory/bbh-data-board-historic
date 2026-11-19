---
title: "WO-04 — MergeVisitors di plugin visit + endpoint admin"
description: "Fungsi service baru plugins/visit/services MergeVisitors(survivorID, mergedIDs[]) — SATU transaksi pindahkan visit_visits + riwayat status ke survivor, hapus visitor lain, jaga invarian phone_e164 unik partial; endpoint admin POST /api/admin/visit/visitors/merge (guard visit.write + audit); kandidat survivor default = data paling lengkap"
status: pending
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - visit
    - backend
    - merge
    - dedup
---
# WO-04 — `MergeVisitors` di plugin visit + endpoint admin

Sumber: SPEC `spec.md` topic `1795107489593` (W-4) + desain
`notes.md` §"Satu tulisan backend plugin-target". Prasyarat: tidak
menggantung WO-02/03 (dipakai WO-03 saat eksekusi merge). Status:
**pending**. Kode di `plugins/visit/` — SATU-SATUNYA tulisan backend
plugin-target di topic ini.

## Lingkup

### A. Service `MergeVisitors(survivorID, mergedIDs[])`

1. `plugins/visit/services` — fungsi baru (mis.
   `visitor_merge.go` + test DB):
   - Input: `survivorID` + `mergedIDs[]` (1..200; survivor tidak boleh
     termasuk mergedIDs → 400; semua ID wajib ada → 404 jujur).
   - **SATU transaksi**:
     a. Update `visit_visits.visitor_id` → survivor (semua visit milik
        merged).
     b. Update riwayat status bila tabel mengacu visitor (periksa
        skema; yang mengacu visit sudah ikut (a)).
     c. Hapus visitor merged (hard delete).
   - Jaga invarian **`phone_e164` unik partial**: bila survivor dan
     merged sama-sama punya E.164 sama → tidak mungkin (duplikat memang
     akar masalah) — survivor menang; bila E.164 merged berbeda dan
     sudah dipakai visitor lain di luar transaksi → 409 dengan pesan
     jelas (fail-closed, bukan tebar data).
   - Audit per aksi (survivor, merged, visits moved, deleted count) —
     provider audit plugin visit (pola audit existing).
2. **Kandidat survivor default** (dipakai agent untuk usulan):
   helper read-only `SuggestSurvivor(groupIDs[])` — skor = HP valid +
   jumlah visit terbanyak + data terlengkap; KONSULTATIF saja —
   keputusan tetap user di kartu approve.

### B. Endpoint admin

1. `POST /api/admin/visit/visitors/merge` — guard `visit.write` +
   audit (pola endpoint visit existing); body `{survivor_id,
   merged_ids[]}`; dry-run preview param `?dry_run=1` (jumlah visit
   yang pindah/hilang, tanpa menulis) — dipakai WO-03 (W-3).
2. Swagger anotasi (`@Router`, `@Success`) — dokumen regenerasi via
   `build.sh` (pola WO-09 visit).

### C. Kompatibilitas

- TIDAK mengubah endpoint/CRUD visit existing (regresi wajib).
- Tidak ada soft delete (W-2): hard delete + audit + preview wajib.

## Acceptance Criteria

- AC-1 Transaksi atomik: bila langkah mana pun gagal → rollback penuh
  (tidak ada visit pindah setengah).
- AC-2 Invarian dedup: setelah merge, tidak ada dua visitor dengan
  `phone_e164` sama; E.164 bentrok di luar grup → 409, tanpa perubahan.
- AC-3 Dry-run preview (`?dry_run=1`) tidak menulis apa pun dan
  menampilkan jumlah terdampak yang benar.
- AC-4 RBAC: tanpa `visit.write` → 403; tanpa token → 401; audit
  tercatat (survivor/merged/visits/deleted).
- AC-5 `SuggestSurvivor` konsultatif — tidak pernah dipanggil untuk
  menulis; agent hanya mengusulkan.
- AC-6 Unit test + test DB (atomik, invarian, preview, 409 bentrok,
  ID tak ada) hijau; `go build/vet/test` hijau; swagger regen; Bruno
  merge visit (positif + bentrok + RBAC); regresi CRUD visit +
  ekstraksi; **Log Implementasi (Dev)**.

## Non-tujuan

- Merge tabel lain (resident/sec/hsh) — kandidat lanjutan, tidak di
  scope; soft delete; perubahan kunci dedup importer.

## Catatan Implementasi (Dev)

(belum ada — isi bertahap per bagian sesuai aturan pinned)

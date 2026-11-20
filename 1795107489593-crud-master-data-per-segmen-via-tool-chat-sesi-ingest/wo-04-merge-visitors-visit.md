---
title: "WO-04 — MergeVisitors di plugin visit + endpoint admin"
description: "Fungsi service baru plugins/visit/services MergeVisitors(survivorID, mergedIDs[]) — SATU transaksi pindahkan visit_visits + riwayat status ke survivor, hapus visitor lain, jaga invarian phone_e164 unik partial; endpoint admin POST /api/admin/visit/visitors/merge (guard visit.write + audit); kandidat survivor default = data paling lengkap"
status: complete
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

Entri berikut mencatat **keputusan** + alasan, bukan ringkasan aktivitas.
Format: `berkas:baris` — mengapa.

1. `plugins/visit/services/visitor_merge.go:37` (`maxMergeBatch=200`) — batas
   jumlah visitor per merge ditegakkan di service, bukan hanya handler, supaya
   jalur tool (apply_record_actions) dan jalur admin langsung tak bisa
   melewatinya; konsisten dengan `MaxRecordActions` WO-03 (W-1).
2. `plugins/visit/services/visitor_merge.go:79` (`normalizeMergeIDs`) —
   memusatkan empat guard sekaligus: survivor_id wajib, survivor ∉ merged_ids
   (400), dedupe + buang id kosong, dan batas 1..200. Semua jalur masuk
   (admin/tool) lewat sini sehingga aturan tak terduplikasi.
3. `plugins/visit/services/visitor_merge.go:111` (`loadMergeGroup`) — memuat
   survivor + seluruh merged di DALAM transaksi dan mengembalikan 404 **jujur**
   menyebut id yang hilang. Alternatif "abaikan yang tak ada" di tolak: itu
   sukses-sem­u yang menyesatkan user di chat (AC-1 fail-closed).
4. `plugins/visit/services/visitor_merge.go:145` (`checkPhoneConflicts`) —
   invarian `phone_e164` unik di jaga eksplisit SEBELUM menulis: bentrok di
   dalam grup dimenangkan survivor (memang akar duplikasi), bentrok dengan
   visitor di luar grup → `MergeConflictError` (409) tanpa perubahan apa pun
   (AC-2).
5. `plugins/visit/services/visitor_merge.go:64` (`MergeConflictError` +
   `Is`) — error khusus membawa `PhoneE164/OwnerID/MergedID` supaya handler
   bisa merender detail actionable (siapa pemilik nomor), bukan sekadar "409".
6. `plugins/visit/services/visitor_merge.go:180` (`MergeVisitors`) — seluruh
   langkah (a) pindah `visit_visits`, (c) hapus visitor merged, audit
   dibungkus `s.db.Transaction`. (b) "pindah riwayat status" **NO-OP**:
   `visit_status_history` hanya ber-FK ke `visit_id` (ON DELETE CASCADE),
   tak ada kolom visitor — jadi memindah visit otomatis memindah riwayatnya.
   Ini keputusan sadar setelah membaca skema, bukan lupa mengerjakan.
7. `plugins/visit/services/visitor_merge.go:214` + `:277`
   (`errDryRunRollback`) — preview `?dry_run=1` dijalankan lewat transaksi yang
   sama lalu **di-rollback** dengan sentinel error khusus. Pola
   rollback-as-control-flow: dampak (`visits_moved`) dihitung dari tulisan
   nyata, tetapi DB tak berubah (AC-3) — jauh lebih andal daripada menghitung
   dampak secara terpisah yang rawan menyimpang dari eksekusi.
8. `plugins/visit/services/visitor_merge.go:262` — `errDryRunRollback`
   dibedakan dari error asli tepat setelah transaksi: preview mengembalikan
   hasil sukses (`DryRun=true`), error lain diteruskan apa adanya.
9. `plugins/visit/services/visitor_merge.go:72` (`VisitorMerge` audit row) —
   audit plugin visit ditulis di tabel BARU `visit_visitor_merges` **dalam
   transaksi yang sama**. Keputusan W-8: audit tepercaya jalur chat tetap
   `ingest_audit` (provider `ingest-tool`); baris visit ini adalah catatan
   domain plugin (siapa survivor, berapa pindah/hapus) yang tak dimiliki
   ingest.
10. `plugins/visit/migrations/postgres/2026_11_21_110000_create_visit_visitor_merges.up.sql:1`
    — `CREATE TABLE ... + 2 index + COMMENT`; `.down.sql` hanya `DROP TABLE`.
    `merged_ids JSONB` menampung daftar utuh untuk audit/baca ulang, plus
    `changed_by` untuk menelusuri admin pelaku.
11. `plugins/visit/model/models.go:72` (`VisitorMerge`) — struktur GORM
    memetakan tabel audit; `MergedIDs string` (bukan slice) karena kolomnya
    JSONB dan tak perlu di-query sebagai relasi.
12. `plugins/visit/services/visitor_merge.go:296` (`SuggestSurvivor`) —
    helper KONSULTATIF read-only (AC-5): tak ada jalur tulis di fungsi ini.
    Skor = HP valid +50, kunjungan ×2 (maks 40), +5 per origin/note/phone_raw;
    diurutkan menurun. Menambah sinyal kelengkapan data agar agent mengusulkan
    survivor yang paling "kaya" — keputusan tetap di kartu approve user.
13. `plugins/visit/handlers/visitor_merge.go:47` (`MergeVisitors`) — handler
    memakai `dryRunOn(c)` + `adminID(c)` yang SUDAH ada di `handlers/import.go`
    (tak dideklarasikan ulang) → `changed_by` mengalir ke audit.
14. `plugins/visit/handlers/visitor_merge.go:111` (`writeMergeError`) —
    `errors.As` ke `MergeConflictError` memetakan ke 409 dengan
    `error/detail/phone_e164/owner_id/merged_id`; error lain jatuh ke
    `writeError` (peta sentinel baku). Kontrak 409 yang kaya ini yang dibaca
    mobile/chat.
15. `plugins/visit/plugin.go:80-81` — `POST /merge` memakai guard `write`,
    `POST /suggest-survivor` memakai guard `read`. Route statis `/merge`
    berdampingan aman dengan `/visitors/:id` (gin mendukung sibling
    statis+param, sudah diverifikasi). Path `/merge` di pilih PERSIS karena
    itu yang sudah dipanggil `apply_record_actions` (WO-03) — chat langsung
    hidup tanpa perubahan sisi ingest.
16. `plugins/ingest/services/record_apply.go:351` — cabang eksekusi merge
    mengirim payload sebenarnya `{survivor_id, merged_ids}` ke
    `visit/visitors/merge`. Sebelumnya body kosong (`action.Body`) sehingga
    endpoint pasti 400 → ini **bug nyata** yang di temukan saat integrasi.
17. `plugins/ingest/services/record_apply.go:296` (`previewMergeImpact`) —
    pratinjau W-3 kini memanggil endpoint merge `?dry_run=1` dan mem-parse
    `visits_moved`, sehingga kartu approve menampilkan **dampak nyata** bukan
    sekadar daftar ID. Kegagalan self-call tidak menggagalkan preview
    (fallback ke daftar ID) supaya UX tetap jalan.
18. `plugins/ingest/services/record_apply.go:239/283` — dampak dry-run
    disimpan di `mergeMoved` terpisah lalu dipakai untuk `preview.Note`. Ini
    memperbaiki bug urutan: teks generik "Visitor digabungkan..." yang di set
    di akhir **menimpa** catatan dampak ("N kunjungan akan pindah...").
19. `plugins/visit/services/visitor_merge_db_test.go` — `TestMergeVisitorsPhoneConflict`
    melepas unique index partial dalam test lalu memasangnya kembali SETELAH
    baris duplikat dibersihkan (urutan cleanup LIFO); tanpa itu mustahil
    menyuntik dua `phone_e164` sama karena index-nya sendiri yang mencegah
    keadaan tersebut.
20. `plugins/visit/services/visitor_merge_db_test.go` (`TestMergeVisitorsRollsBackOnFailure`)
    — kegagalan langkah (a) di suntik lewat `CHECK (false) NOT VALID` pada
    `visit_visits` (baris lama tak divalidasi ulang) untuk membuktikan AC-1:
    rollback penuh, tanpa baris audit tertinggal.

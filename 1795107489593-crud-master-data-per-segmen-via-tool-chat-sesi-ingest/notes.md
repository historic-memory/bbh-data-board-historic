---
title: "Desain — CRUD master data per segmen via tool chat sesi ingest"
description: "Draf desain (owner request 2026-11-20): tool chat per sesi untuk CRUD master data SEGMENT sesi (visit/resident/sec/hsh) — list/find duplicates/merge/delete/edit; P-3 propose→approve→Go eksekusi; advanced search menyusul; topic extends 1795083433686"
status: draft
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - crud
    - ai-chat
    - desain
---
# Desain — CRUD master data per segmen via tool chat sesi ingest

Topic `1795107489593` (extends `1795083433686`, turunan topic induk
`1793711295286`). Owner request 2026-11-20: "tiap sesi punya tools untuk
manage CRUD sesi tersebut… pokoknya toolsnya lengkap". Status: **draf
desain — menunggu review owner** sebelum SPEC/WO.

## 1. Keputusan owner (terkunci 2026-11-20)

- **Objek = master data per SEGMENT** (bukan batas batch): sesi dengan
  `target_plugin=visit` → tool mengelola `visit_visitors`/`visit_visits`;
  demikian juga resident/sec/hsh. "crud per segment pastinya".
- **Kasus nyata pertama**: data kunjungan dev sekarang ada yang dobel
  (dampak re-impor dengan `time_only` — baris tanpa HP tidak punya kunci
  dedup; batasan tercatat WO-09 visit) → **merge visitor duplikat** =
  operasi utama pertama.
- **Advanced search**: kandidat lanjutan, "seiring waktu" — dicatat
  backlog, TIDAK di scope awal.
- **Penempatan**: topic baru extends `1795083433686` (bukan menambah WO
  di topic lama / topic induk yang sudah padat).

## 2. Fakta kode terverifikasi (2026-11-20)

1. **Lineage baris → master BELUM persisten.** `VisitorID`/`VisitID` per
   baris hanya ada di report in-memory; `visit_import_rows` (staging)
   tidak menyimpan id master. Karena scope = per segmen (bukan per
   batch), lineage TIDAK bloker — opsional lanjutan bila owner ingin
   "lihat hasil impor sesi ini saja".
2. **Pola tool sudah matang**: endpoint `/admin/ingest/tools/*`
   ber-guard `ingest.write` + audit `provider='ingest-tool'` (WO-01/
   WO-03); Node = skema + forward Bearer user (D-11); proposal →
   `confirmed_by_user` wajib → Go eksekusi (P-3, pola WO-03
   `apply_fixes`).
3. **Duplikat visit**: kunci dedup katalog = `phone_e164` + `scheduled_at`
   (`phone_then_time`) atau waktu saja (`time_only`). Data dev memuat
   duplikat lintas batch — deteksi duplikat harus di data master, bukan
   hanya batch sesi.
4. **Tidak ada soft delete** di `visit_visitors`/`visit_visits` —
   delete = hard delete + audit (kecuali owner putuskan lain, O-2).
5. **Merge harus di plugin visit**, bukan di ingest: ingest memanggil
   service layer plugin target (pola `target.go`/importer). Semua tulisan
   master lewat service layer = RBAC + audit + aturan plugin tetap
   berlaku.

## 3. Draf toolset (M-1..M-4)

Semua endpoint `POST /admin/ingest/tools/*`, pola tool strict existing
(Go eksekusi, Node skema, Bearer user, audit `provider='ingest-tool'`):

| Tool | Fungsi | Catatan |
|---|---|---|
| **M-1 `list_segment_records`** | Daftar record master segmen (visit: visitors + visits; resident/sec/hsh mengikuti) dengan filter dasar (kata kunci, tanggal, status) + paginasi | Nilai di chat **terdeidentify** (P-5); operasi by ID |
| **M-2 `find_duplicates`** | Cari kelompok duplikat dengan kunci dedup katalog segmen + kandidat survivor | Dry-run; keluaran = grup + ID, tanpa nilai mentah |
| **M-3 `propose_record_actions`** | Usulan batch: `merge` / `delete` / `update` (field tertentu) — HARUS merujuk ID nyata; disimpan sebagai proposal | TIDAK menerapkan apa pun (P-3) |
| **M-4 `apply_record_actions`** | Eksekusi proposal: `confirmed_by_user: true` wajib (400 bila hilang); eksekusi via service layer plugin target; audit per aksi | Dry-run preview (jumlah terdampak) otomatis dulu; hasil re-report |

### Semantik merge (visit — kasus pertama)

- User/agent pilih **survivor** dari grup duplikat.
- Go (plugin visit): pindahkan `visit_visits` + riwayat status ke survivor,
  hapus visitor lainnya, dalam SATU transaksi; jaga invarian dedup
  (`phone_e164` unik partial).
- Ter-audit per aksi (survivor, merged, visits moved).
- Kandidat survivor default = yang punya data paling lengkap (HP valid +
  kunjungan terbanyak) — agent mengusulkan, user menyetujui.

### Keamanan lintas tool

- **Guard**: `ingest.write` (pola O-22) — PLUS usulan fail-closed:
  wajib juga permission tulis plugin target (`visit.write`, `resident.write`
  dst.) karena operasi menyentuh master data (O-1, menunggu owner).
- **Bulk limit**: usulan max 200 ID per aksi (pola bulk delete WO-04a)
  (O-4, menunggu owner).
- **Delete**: usulan hard delete + audit (data visit belum punya
  soft delete) (O-2, menunggu owner).
- **P-5**: nilai master data tidak pernah mentah ke LLM/chat — tool
  bekerja by ID + nilai deidentified; kartu approval di chat pola
  `rejected_listing` (dibuat backend Go, WO-03).

## 4. Sisa terbuka (menunggu keputusan owner)

- **O-1 Guard permission**: cukup `ingest.write` (O-22) atau wajib plus
  permission plugin target (`visit.write` dst.)? Usulan Manager: plus.
- **O-2 Delete**: hard delete + audit (usulan) atau perlu soft delete?
- **O-3 Bentuk kartu approval di chat**: pola `rejected_listing` WO-03
  (usulan) — cukup atau perlu bentuk baru?
- **O-4 Bulk limit**: 200 ID/aksi (usulan) atau nilai lain?
- **Advanced search** (filter lanjutan, fuzzy, dll.): backlog — tidak
  di scope awal (keputusan owner "seiring waktu").

## 5. Kasus uji pertama

Data kunjungan dev sekarang (duplikat hasil re-impor owner): end-to-end
via chat — M-2 temukan grup duplikat → M-3 merge proposal → kartu
approval → M-4 apply → verifikasi master data bersih + audit lengkap +
regresi fitur existing (analisa rejected, ekstraksi, listing).

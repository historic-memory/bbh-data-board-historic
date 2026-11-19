---
title: "SPEC — CRUD master data per segmen via tool chat generik (Hybrid)"
description: "SPEC final: 2 tool generik segment_read (proxy whitelist GET + deidentify) & segment_write_propose (proposal→kartu approve→apply_record_actions) untuk SEMUA segmen + MergeVisitors plugin visit; guard whitelist per segmen + RBAC target; WO-01..WO-05"
status: progress
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - crud
    - ai-chat
    - spec
---
# SPEC — CRUD master data per segmen via tool chat generik (Hybrid)

> **Status: DISETUJUI owner 2026-11-20** ("ok aku setuju"). Menjadi
> acuan eksekusi WO-01..WO-05 topic ini. Status dok: `progress`.

Topic `1795107489593` (extends `1795083433686`, turunan induk
`1793711295286`). Sumber: `notes.md` (DESAIN FINAL owner ok 2026-11-20).
Status: **draft — menunggu review owner** sebelum WO dieksekusi.

## 1. Masalah

Hasil sesi ingest hanya bisa dibereskan lewat analisa rejected (topic
`1795083433686`); data master hasil ekstraksi (kunjungan dobel hasil
re-impor, data salah isi) harus dibetulkan manual di UI admin per modul.
Owner ingin agent chat punya tools CRUD lengkap untuk master data segmen
sesi — flexible, tanpa tool baru per segmen.

## 2. Solusi (Hybrid — owner ok 2026-11-20)

1. **`segment_read`** (M-1) — 1 tool baca generik: proxy GET-only ke
   API admin segmen sesi dengan whitelist prefix per segmen (fail-closed
   di Go) + **deidentify wajib** sebelum response ke LLM (P-5, ID tetap
   utuh). RBAC baca plugin target.
2. **`segment_write_propose`** (M-2) — 1 tool tulis generik: usulan
   create/update/delete/merge **TIDAK menerapkan apa pun** (P-3) —
   proposal tersimpan (snapshot sesi, O-25) + kartu approve di chat
   (backend Go, pola `rejected_listing` WO-03).
3. **`apply_record_actions`** (M-3) — eksekusi proposal:
   `confirmed_by_user` wajib (400 bila hilang); eksekusi = panggilan API
   admin asli (RBAC + audit endpoint tercatat) + audit ingest-tool;
   dry-run preview terdampak; max 200 aksi/proposal.
4. **`MergeVisitors`** di `plugins/visit/services` + endpoint admin —
   satu-satunya backend plugin-target baru (kasus pertama: duplikat
   kunjungan dev).
5. **Patch frontend** (WO-01, sudah dibuat): hapus tombol Ekstrak &
   Konfirmasi di editor + modal Simpan draft → chat — alur "semua via
   chat".

## 3. Keputusan terkunci (owner 2026-11-20)

- Arsitektur **Hybrid** (bukan tool semantik per segmen): 2-3 tool
  generik untuk SEMUA segmen — plugin baru ikut otomatis.
- **Guard = whitelist per segmen + RBAC target**: baca wajib permission
  baca plugin target (`visit.read` dst.); tulis wajib permission tulis
  plugin target + approve user. Larangan keras: `admins/groups/
  permissions/audit/ingest/providers/dashboard/endpoints`.
- **Approval = proposal store + kartu approve di chat** (bisa batch).
- P-3/P-5/D-11/O-25 pola topic induk TIDAK berubah.
- Delete = hard delete + audit (O-2; sec/hsh sudah 409 bila terpakai,
  resident CASCADE — preview jumlah terdampak wajib).
- Advanced search = backlog (filter API baru nanti; tool ikut otomatis).
- Node hanya skema (D-11) — whitelist di Go, bukan prompt.

## 4. Lingkup per WO (5)

### WO-01 — Patch frontend: hapus tombol Ekstrak & Konfirmasi + modal Simpan draft

SUDAH DIBUAT (`wo-01-hapus-tombol-ekstrak-konfirmasi-editor.md`,
pending). Hapus 2 tombol jalur manual di `IngestMappingPage`; Simpan
draft → modal "Kembali ke chat"/"Lanjut edit"; verifikasi dedup/dry-run
masih bisa dipilih di chat (AC-4). Prasyarat seluruh WO berikutnya.

### WO-02 — Backend `segment_read` + skema Node

- Endpoint `POST /admin/ingest/tools/segment_read`: input
  `{session_id, path, query?}` — path relatif segmen sesi, GET-only.
- Whitelist prefix per segmen (map segmen → prefix API, fail-closed
  `ErrSegmentPathForbidden`); RBAC baca plugin target (bukan cukup
  `ingest.write`).
- **Deidentify tunggal di Go** sebelum response ke LLM (pola
  `DeidentifyColumns`; field PII dimask, ID utuh).
- Node: skema 1 tool di `copilot-runtime.ts` (pola existing).
- AC: whitelist menolak di luar segmen; viewer baca tanpa permission
  target → 403; tanpa token 401; response terdeidentify; audit
  `provider='ingest-tool'`; Bruno; tsc hijau.

### WO-03 — Backend `segment_write_propose` + `apply_record_actions` + kartu approve

- Endpoint M-2: validasi keberadaan ID (400 bila tidak ada), method
  POST/PUT/PATCH/DELETE, path whitelist tulis, body endpoint read-only
  ditolak; proposal tersimpan snapshot + audit; TIDAK menerapkan apa
  pun.
- Endpoint M-3: `confirmed_by_user: true` wajib (400 fail-closed — pola
  WO-03); eksekusi via API admin asli (internal); dry-run preview
  terdampak (delete/merge); max 200 aksi/proposal; hasil re-report ke
  chat (pola O-18).
- Kartu approve `record_actions` di chat (backend Go, pola
  `rejected_listing` WO-03) + tombol Setujui per aksi/semua.
- RBAC: baca `ingest.write`; eksekusi wajib permission TULIS plugin
  target (`visit.write` dst.) — fail-closed.
- Node: skema 2 tool. AC: proposal tidak mengubah data; apply tanpa
  confirm → 400; audit ganda; regresi 4 tool existing (WO-01 topic
  lama) tidak berubah; Bruno; log implementasi.

### WO-04 — `MergeVisitors` di plugin visit + endpoint admin

- `plugins/visit/services` fungsi baru `MergeVisitors(survivorID,
  others[])`: SATU transaksi — pindahkan `visit_visits` + riwayat status
  ke survivor, hapus visitor lain, jaga invarian `phone_e164` unik
  partial; kandidat survivor default = data paling lengkap (usulan
  agent).
- Endpoint admin `POST /api/admin/visit/visitors/merge` (guard
  `visit.write`) + audit.
- Dry-run preview via M-3 (jumlah visit yang pindah/dihapus).
- AC: transaksi atomik (gagal separuh jalan → rollback); duplikat HP
  berbeda → survivor menang normalisasi E.164; audit lengkap; unit test
  + Bruno; regresi CRUD visit existing.

### WO-05 — Prompt agent + uji end-to-end + Bruno

- `agent.md` kontrak baru: alur CRUD (baca dulu → usulkan → tunggu
  approve → eksekusi), whitelist manual (panduan; penolakan tetap di
  Go), PII tidak pernah ditampilkan mentah, merge visitor = usulan
  survivor + alasan.
- Bruno: folder `api-tests/ingest-crud/` (segment_read whitelist+
  deidentify; propose→apply rantai; merge visit; negatif: path di luar
  whitelist 400, apply tanpa confirm 400, viewer 403, tanpa token 401).
- Uji kasus pertama: data kunjungan dev yang dobel — end-to-end via
  chat.
- AC: seluruh alur lulus; regresi fitur existing (analisa rejected,
  ekstraksi, listing WO-02 topic lama) tidak berubah; log
  implementasi.

## 5. Non-tujuan

- Advanced search / filter lanjutan (backlog).
- Soft delete master data (O-2 tetap hard + audit).
- Lineage baris → master (opsional, tidak bloker).
- Tool untuk modul di luar segmen (admins/groups/audit dsb.) — larangan
  keras.
- Perubahan state machine ingest / katalog rule / tabel relasional baru.

## 6. Risiko

- **Privilege escalation via proxy baca** → mitigasi: whitelist prefix
  fail-closed di Go + RBAC target; test negatif eksplisit.
- **PII bocor ke LLM via response API** → mitigasi: deidentify tunggal
  sebelum response; test deidentify per segmen.
- **Merge salah sasaran** → mitigasi: dry-run preview wajib, survivor
  diusulkan agent & disetujui user, transaksi atomik, audit.
- **Delete massal salah** → mitigasi: max 200 aksi, preview terdampak,
  hard delete hanya via kartu approve.

## Rencana bukti (per WO)

- WO-02: Bruno whitelist + deidentify + RBAC.
- WO-03: Bruno rantai propose→approve→apply + negatif + audit rows.
- WO-04: unit test transaksi + Bruno merge + regresi visit.
- WO-05: uji data kunjungan dobel dev end-to-end + regresi fitur
  existing.

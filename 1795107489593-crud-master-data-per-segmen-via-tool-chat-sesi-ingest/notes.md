---
title: "Desain — CRUD master data per segmen via tool chat sesi ingest"
description: "DESAIN FINAL (owner ok 2026-11-20): 2 tool generik untuk CRUD master data SEMUA segmen via chat — segment_read (proxy whitelist GET + deidentify) & segment_write_propose (proposal→approve→eksekusi API asli); whitelist per segmen + RBAC target; kasus pertama merge visitor duplikat visit; topic extends 1795083433686"
status: progress
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
manage CRUD sesi tersebut... pokoknya toolsnya lengkap". Status:
**DESAIN FINAL — owner ok 2026-11-20** (3 keputusan dikunci: Hybrid,
whitelist+RBAC target, proposal store + kartu approve). SPEC + WO
menyusul. WO-01 patch frontend (hapus tombol Ekstrak/Konfirmasi + modal
Simpan draft → chat) DIBUAT & SIAP di topic ini terlebih dahulu.

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
- **ARSITEKTUR = HYBRID (owner ok 2026-11-20)**: 1 tool generik baca
  (proxy whitelist GET + deidentify) + 1 tool generik tulis
  (proposal → approve → eksekusi API asli) untuk SEMUA segmen —
  lebih flexibel daripada tool semantik per segmen (opsi A "tools lebih
  ke api http request aja" → dibulatkan jadi C dengan guard ketat).
- **GUARD = whitelist per segmen + RBAC target**: GET cukup permission
  baca plugin target (`visit.read` dst.); tulis wajib permission tulis
  plugin target + approve user; agent TIDAK boleh akses endpoint di luar
  segmen (admins/groups/audit/ingest sendiri).
- **APPROVAL = proposal store + kartu approve di chat** (pola
  `rejected_listing` WO-03) — bisa approve batch sekaligus.

## 2. Fakta kode terverifikasi (2026-11-20)

1. **151+ fungsi service CRUD sudah ter-expose via HTTP API admin**
   (terverifikasi grep: visit `ListVisitors/GetVisitor/UpdateVisitor/
   DeleteVisitor/ListVisits/AddStatus`; resident `GetResident/
   ListResidents/.../DeleteResident` (CASCADE); sec & hsh
   `ListCategories/ListItems/DeleteItem/DeleteEntry/DeletePeriod` 409
   bila terpakai dst.) — API admin SIAP dipakai sebagai back-end tool;
   NOL endpoint baru per segmen.
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
5. **Semua tulisan master lewat API/service layer plugin target** —
   RBAC + audit + aturan plugin (409 in-use sec/hsh, CASCADE resident,
   dsb.) tetap berlaku. Plugin visit TAMBAHAN: `MergeVisitors` BELUM ada
   → satu fungsi service baru di `plugins/visit/services` diperlukan
   (satu-satunya tulisan backend plugin-target di scope ini).

## 3. ARSITEKTUR FINAL — 2 tool generik (HYBRID, owner ok 2026-11-20)

Semua endpoint `POST /admin/ingest/tools/*`, pola tool strict existing
(Go eksekusi, Node skema, Bearer user, audit `provider='ingest-tool'`):

### M-1 `segment_read` — baca generik (proxy whitelist + deidentify)

- Endpoint: `POST /admin/ingest/tools/segment_read`
- Input: `{session_id, path, query?}` — `path` RELATIF ke segmen sesi
  (contoh segmen visit: `visit/visitors`, `visit/visits`), method
  GET-only, ditolak selain itu.
- **Whitelist prefix per segmen** (fail-closed):
  - visit → `/api/admin/visit/*` (visitors, visits, status)
  - resident → `/api/admin/residents*`, `facilities*`
  - sec → `/api/admin/sec/*`; hsh → `/api/admin/hsh/*` + `needs/*`
  - LARANG (400 `ErrSegmentPathForbidden`): `admins`, `groups`,
    `permissions`, `audit`, `ingest/*`, `providers`, `dashboard`,
    `endpoints`, `resident-template` — apapun di luar daftar.
- **RBAC**: pemanggil wajib punya permission BACA plugin target
  (`visit.read`, `resident.read`, `sec.read`, `hsh.read`) — kecuali O-22
  (`ingest.write` saja) TIDAK dipakai untuk baca lintas modul.
- **Deidentify WAJIB sebelum ke LLM** (P-5): response JSON dari API asli
  dilewati satu jalur deidentify (pola `DeidentifyColumns` WO-01) —
  field yang ter-mapping PII dimask deterministik; response tetap
  terstruktur (ID tetap utuh — operasi by ID tetap mungkin).
- **Filter dasar** via query params asli API (`q`, `limit`, `offset`,
  `branch_key`, dsb.) — advanced search nanti = filter API baru,
  TIDAK butuh tool baru.

### M-2 `segment_write_propose` — tulis generik (proposal → approve)

- Endpoint: `POST /admin/ingest/tools/segment_write_propose`
- Input: `{session_id, actions[]}` dengan `action = {kind: create|update|
  delete|merge, path, method, body|ids}` — HARUS merujuk ID nyata
  (validasi keberadaan dilakukan saat propose; ID tak ada → 400).
- **TIDAK menerapkan apa pun** (P-3): disimpan sebagai proposal di
  snapshot sesi (pola WO-01), ter-audit; muncul sebagai **kartu approve
  di chat** (backend Go, pola `rejected_listing` WO-03).
- Kartu = ringkasan per aksi (kind, target ID, delta deidentified) +
  tombol **Setujui** (per aksi / semua) → `apply_record_actions`
  (M-3) dengan `confirmed_by_user`.
- Fail-closed: method selain POST/PUT/PATCH/DELETE → 400; path di luar
  whitelist segmen → 400; body yang menarget endpoint baca-only → 400.

### M-3 `apply_record_actions` — eksekusi proposal

- Endpoint: `POST /admin/ingest/tools/apply_record_actions`
- `confirmed_by_user: true` WAJIB (400 bila hilang — pola WO-03
  `ErrApplyNotConfirmed`).
- Eksekusi = **panggilan API admin asli** (internal, service layer
  plugin target) dengan admin id dari token → RBAC + audit endpoint
  asli tetap tercatat; ingest menambah audit sendiri
  `provider='ingest-tool'` (aksi CRUD per proposal).
- Bulk limit usulan: max 200 aksi/proposal (pola bulk delete WO-04a).
- Dry-run preview: jumlah record terdampak dulu (khusus delete/merge:
  daftar visit yang akan pindah/dihapus) sebelum eksekusi sungguhan.
- Hasil re-report ke chat (pola pesan sistem O-18).

### Satu tulisan backend plugin-target yang diperlukan

- **`MergeVisitors` di `plugins/visit/services`** (baru — tidak ada di
  API): survivor + daftar visitor lain → SATU transaksi memindahkan
  `visit_visits` + riwayat status ke survivor, hapus visitor lainnya,
  jaga invarian dedup (`phone_e164` unik partial); audit per aksi
  (survivor, merged, visits moved). Endpoint admin-nya juga dibuat
  (dipakai M-3). Kandidat survivor default = data paling lengkap —
  agent mengusulkan, user menyetujui.

### Keamanan lintas tool (LOCKED)

- **Node TIDAK berubah per segmen**: hanya 2-3 skema tool generik
  (D-11 tetap — tanpa logika bisnis di Node).
- **P-5**: nilai master data tidak pernah mentah ke LLM — jalur
  deidentify tunggal di Go sebelum response ke model/chat.
- **Fail-closed**: whitelist di Go, bukan di prompt (prompt hanya
  panduan; penolakan tetap 400 bila dicoba).
- **Audit ganda**: audit endpoint asli (RBAC) + audit ingest-tool per
  proposal/aksi (siapa approve, apa isinya).

## 4. Sisa terbuka (kecil, bisa diputuskan saat SPEC)

- **O-2 Delete**: hard delete + audit (usulan — data visit belum punya
  soft delete). sec/hsh sudah 409 bila terpakai; resident CASCADE —
  preview wajib tampilkan jumlah baris terdampak.
- **O-5 Baca lintas-sesi**: tool hanya aktif pada sesi dengan segmen
  (`target_plugin` wajib)? Usulan: ya (sama dengan WO-02 panel).
- **Advanced search**: backlog (owner: "seiring waktu") — saat itu,
  cukup tambah filter di API segmen, tool ikut otomatis.

## 5. Kasus uji pertama

Data kunjungan dev sekarang (duplikat hasil re-impor owner): end-to-end
via chat — M-1 baca visitors (deidentified) → temukan grup duplikat →
M-2 propose merge (survivor usulan agent) → kartu approve di chat →
M-3 apply (MergeVisitors) → verifikasi master data bersih + audit
ganda + regresi fitur existing (analisa rejected, ekstraksi, listing).

## 6. Daftar WO draf (disusun setelah desain final ini)

- WO-01 (SUDAH DIBUAT): hapus tombol Ekstrak & Konfirmasi editor +
  modal Simpan draft → chat (patch frontend, prasyarat alur "semua via
  chat").
- WO-02 (draf): backend `segment_read` (whitelist + deidentify) +
  skema Node.
- WO-03 (draf): backend `segment_write_propose` + `apply_record_actions`
  + proposal store snapshot + kartu approve chat (pola WO-03) + skema
  Node.
- WO-04 (draf): `MergeVisitors` di plugin visit + endpoint admin +
  dry-run preview + audit.
- WO-05 (draf): prompt agent (kontrak CRUD: user selalu approve,
  whitelist manual, PII tidak ditampilkan mentah) + Bruno end-to-end +
  uji duplikat kunjungan dev.

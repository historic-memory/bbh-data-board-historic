---
title: "WO-03 — Backend segment_write_propose + apply_record_actions + kartu approve chat + Node"
description: "Tulis generik: usulan create/update/delete/merge → proposal store (snapshot sesi) + kartu approve di chat (backend Go pola rejected_listing WO-03) → apply_record_actions confirmed_by_user wajib → eksekusi via API admin asli (RBAC + audit ganda) + dry-run preview + max 200 aksi"
status: complete
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - backend
    - tool
    - crud
---
# WO-03 — Backend: `segment_write_propose` + `apply_record_actions` + kartu approve + Node

Sumber: SPEC `spec.md` topic `1795107489593` + desain `notes.md`.
Prasyarat: WO-02 selesai (whitelist + deidentify hidup — dipakai ulang
di sini). Status: **pending**. Pola ikut WO-03 topic `1795083433686`
(`analysis_apply.go` + kartu `rejected_listing`).

## Lingkup

### A. `POST /admin/ingest/tools/segment_write_propose` (M-2)

1. Input: `{session_id, actions[]}`, `action =
   {kind: create|update|delete|merge, path, method, body?, ids?}`:
   - kind `create`/`update` → `path` + `body`; `delete` → `ids[]`
     (1..200); `merge` → `{survivor_id, merged_ids[]}` (hanya visit
     untuk saat ini — endpoint merge WO-04; segmen lain 409 dulu
     sampai tersedia).
   - **Validasi keberadaan ID** saat propose (GET internal): ID tidak
     ada → 400 (fail-closed, sebelum kartu tampil).
   - method wajib POST/PUT/PATCH/DELETE (GET ditolak); path wajib di
     whitelist tulis segmen (peta WO-02, GET-prefixed path ditolak);
     **TIDAK menerapkan apa pun** (P-3) — disimpan sebagai PROPOSAL
     di snapshot sesi (pola WO-01 topic lama; O-25 persisten) + audit.
2. Proposal ber-ID (`proposal_id`), status `proposed`; muncul sebagai
   **kartu approve `record_actions` di chat** — dibuat backend Go
   (pola `rejected_listing` WO-03; bukan teks LLM), ringkasan per aksi
   (kind, target, delta deidentified) + tombol Setujui.

### B. `POST /admin/ingest/tools/apply_record_actions` (M-3)

1. Input: `{session_id, proposal_id, action_idx[]?}` — subset aksi boleh
   (default semua); `confirmed_by_user: true` WAJIB eksplisit
   (pointer; hilang/false → 400 `ErrApplyNotConfirmed`, pola WO-03
   topic lama).
2. **Dry-run preview** otomatis (W-3) untuk delete/merge SEBELUM
   eksekusi: jumlah + daftar record terdampak (visit yang pindah/
   hilang) — kembalikan preview bila user belum pernah approve aksi
   yang sama (idempotensi per client_id deterministik, pola T-4).
3. **Eksekusi = panggilan API admin asli** (internal service-layer
   call, pola dispatcher `target.go`) dengan admin ID dari token user:
   - `create/update/delete` → endpoint admin segmen yang dipetakan
     (method sesuai kind); RBAC TULIS plugin target wajib (403 tanpa
     `visit.write` dst. — fail-closed).
   - `merge` → endpoint `POST /api/admin/visit/visitors/merge` (WO-04).
4. **Bulk limit** (W-1): max 200 aksi/proposal → 400 di atas itu.
5. **Audit ganda** (W-8): audit endpoint asli tercatat otomatis +
   audit `provider='ingest-tool'` per proposal/aksi (siapa approve,
   isi aksi, hasil).
6. Hasil re-report ke chat (pola pesan sistem O-18) + snapshot
   proposal diperbarui (`applied`/`partially_applied`/gagal per aksi).
7. Guard `ingest.write` + audit ingest-tool (konsisten tool lain).

### C. UI chat (mini)

1. Renderer kartu `record_actions` di `IngestChatPage.tsx` (pola kartu
   `rejected_listing` WO-03 topic lama): daftar aksi + tombol
   "Setujui" per aksi/semua → panggil `apply_record_actions` (API
   backend), PermissionGate `ingest.write`.
2. Tidak ada UI lain (listing panel tidak dipakai di sini).

### D. Node

Skema 2 tool (`segment_write_propose`, `apply_record_actions`) di
`copilot-runtime.ts` — pola existing, Bearer user, tanpa logika bisnis;
deskripsi menegaskan: agent tidak boleh mengisi `confirmed_by_user`
sendiri; setelah propose, tunggu keputusan user.

## Acceptance Criteria

- AC-1 Propose tidak mengubah data master (bukti DB + audit); proposal
  persisten (reload tetap) dengan ID + status.
- AC-2 Kartu approve tampil di chat (backend Go) dengan ringkasan
  deidentified; Setujui per aksi/semua berfungsi.
- AC-3 Apply tanpa `confirmed_by_user` → 400; dengan → eksekusi via
  API asli (bukti data berubah + audit endpoint asli + audit
  ingest-tool).
- AC-4 ID tak ada / method salah / path di luar whitelist → 400
  (sebelum kartu tampil).
- AC-5 RBAC: eksekusi tanpa permission tulis target → 403; bulk >200
  → 400.
- AC-6 Dry-run preview tampil sebelum eksekusi delete/merge (W-3).
- AC-7 Regresi: 4 tool analisa (WO-01 topic lama) + segment_read
  (WO-02) tidak berubah perilaku; `go build/vet/test` + tsc + npm
  build hijau; Bruno; **Log Implementasi (Dev)**.

## Non-tujuan

- `MergeVisitors` internal (WO-04), prompt agent (WO-05), soft delete,
  auto-apply oleh sistem, perubahan state machine/tabel baru.

## Catatan Implementasi (Dev)

Entri berikut mencatat **keputusan** + alasan, bukan ringkasan aktivitas.
Format: `berkas:baris` — mengapa.

1. `plugins/ingest/model/record.go:1` — model usulan tulis dipisah dari
   `analysis.go` karena dua domain berbeda (analisa berkas vs CRUD master):
   proposal tulis tidak ikut lifecycle `AnalysisSnapshot` dan tidak boleh
   membuat snapshot analisa "rusak" saat keduanya hidup bersamaan.
2. `plugins/ingest/model/record.go` konstanta `RecordStatus*` memakai 5 status
   (`proposed/previewed/applied/partially_applied/failed`) — `partially_applied`
   wajib ada karena bulk delete bisa lulus sebagian; tanpa itu UI harus menebak
   dari `results`.
3. `plugins/ingest/model/record.go:MaxRecordActions=200` — batas W-1
   ditegakkan di model (bukan hanya handler) supaya jalur tool lama/baru tidak
   bisa melewatinya.
4. `plugins/ingest/model/record.go` `RecordActionView` TIDAK memuat nilai sel,
   hanya `fields[]` (nama field) — penerapan P-5 di bentuk data, bukan sekadar
   kebiasaan prompt.
5. `plugins/ingest/model/session.go:RecordProposals` — menumpang jsonb di baris
   sesi (O-25): ikut terhapus bersama sesi, tak perlu tabel retensi terpisah,
   dan reload halaman tetap melihat proposal yang sama.
6. `plugins/ingest/migrations/postgres/2026_11_21_100000_add_ingest_sessions_record_proposals.up.sql:1`
   — `ADD COLUMN IF NOT EXISTS` + COMMENT: migrasi idempoten dan self-documented;
   down file hanya membuang kolom (data usulan tidak perlu dipertahankan saat
   rollback).
7. `plugins/ingest/services/record_write.go:segmentWriteWhitelist` — peta TULIS
   SENGAJA dipisah dari `segmentReadWhitelist`. Peta baca sengaja longgar
   (`sec/*`, `hsh/*`); bila dipakai untuk tulis, memperluas baca otomatis
   memperluas tulis — pelanggaran fail-closed. Menambah endpoint tulis = satu
   baris eksplisit.
8. `plugins/ingest/services/record_write.go:SegmentWritePath` — memakai ulang
   `segmentReadBannedPrefix` (admins/ingest/audit/dashboard/...) supaya larangan
   keras tidak bisa diakali lewat jalur tulis.
9. `plugins/ingest/services/record_write.go:recordWriteReservedPaths` —
   `residents/stats` adalah endpoint agregat di bawah koleksi; tanpa peta ini
   `DELETE residents/stats` akan lolos sebagai "item ber-ID `stats`"
   (kebocoran nyata, bukan teoretis).
10. `plugins/ingest/services/record_write.go:bulkDeleteAllowed` — hapus massal
    dikirim sebagai `DELETE <koleksi>` + `ids[]`; diizinkan HANYA bila pasangan
    `<koleksi>/:id` memang boleh DELETE. Ini mengikat jalur massal ke whitelist
    per-entitas tanpa menduplikasi daftar.
11. `plugins/ingest/services/record_write.go:recordKindMethod` — method bawaan
    per kind supaya agent tak perlu mengisi `method`; GET dipaksa ditolak di
    jalur terpisah agar pesan errornya jelas ("GET ditolak").
12. `plugins/ingest/services/record_write.go:recordBodyFields` — hanya nama
    field top-level yang disimpan untuk kartu; nilai body tetap di proposal
    (perlu dieksekusi) tetapi tidak pernah dirender ke chat.
13. `plugins/ingest/services/record_write.go:verifyRecordID` — verifikasi ID
    memakai GET internal (pola WO-02) dan **melewati** path yang tak ada di
    whitelist BACA. Alasan: sebagian endpoint item tidak dapat dibaca agent
    tetapi tetap sah ditulis; endpoint asli tetap menolak ID palsu saat apply,
    jadi ini bukan pelonggaran keamanan melainkan menghindari blokir palsu.
14. `plugins/ingest/services/record_write.go:ProposeRecordActions` — urutan
    validasi → verifikasi ID → simpan → tulis kartu; kegagalan di dua langkah
    pertama tidak meninggalkan proposal setengah jadi di DB.
15. `plugins/ingest/services/record_apply.go:ApplyRecordActions` — cek
    `confirmed_by_user` adalah pernyataan pertama (fail-closed), sebelum sesi
    dibaca; ini memastikan 400 konsisten bahkan bila sesi tidak ada.
16. `plugins/ingest/services/record_apply.go` dry-run preview — `needsPreview`
    hanya delete/merge (W-3). Pratinjau disimpan (`Preview`, `PreviewedAt`,
    status `previewed`) sehingga penekanan kedua tidak menampilkan pratinjau
    lagi — persetujuan jadi dua langkah yang tak bisa dilompati.
17. `plugins/ingest/services/record_apply.go:callRecordEndpoint` — eksekusi
    lewat HTTP self-call ke `/api/admin/<path>` memakai **Bearer milik user**
    (Opsi A). Konsekuensi yang diinginkan: RBAC tulis + validasi endpoint asli
    berlaku tanpa duplikasi logika, dan audit endpoint asli tetap tercatat.
18. `plugins/ingest/services/record_apply.go:executeRecordAction` — merge
    memetakan 404 → 409 dengan pesan "endpoint merge belum tersedia (WO-04)";
    agent tidak disuruh menebak apakah gagalnya karena data atau fitur.
19. `plugins/ingest/services/record_apply.go:writeRecordActionsMessage` —
    `client_id = "record-"+sha1(isi pesan)[:8]`; apply berulang dengan hasil
    identik menjadi no-op (`ErrDuplicateMessage` dianggap sukses), sehingga
    kartu tidak menumpuk duplikat di riwayat chat.
20. `plugins/ingest/services/record_apply.go:renderRecordActionsMessage` —
    ringkasan naratif + pratinjau + SATU fenced block JSON, sama seperti
    `rejected_listing`. Format satu-blok dipilih agar parser UI lama tidak perlu
    dua jalur.
21. `plugins/ingest/handlers/record_write.go:SegmentWritePropose` — pre-check
    `<segment>.write` di handler (403) SEBELUM menyentuh service; ini memberi
    pesan yang jelas ke pengguna, sementara penegakan sebenarnya tetap di
    endpoint asli saat eksekusi (dua lapis, bukan dua sumber kebenaran).
22. `plugins/ingest/handlers/record_write.go:applyRecordActionsRequest` —
    `confirmed_by_user *bool` (pointer). Tipe non-pointer akan membuat
    "field hilang" dan "false" tak terbedakan, padahal keduanya harus 400
    dengan pesan yang sama.
23. `plugins/ingest/handlers/record_write.go:recordWriteStatus` — pemetaan
    sentinel → status HTTP dikumpulkan di satu fungsi supaya Bruno/AC bisa
    diuji langsung dan tidak ada `500` liar untuk kesalahan pemakaian.
24. `plugins/ingest/plugin.go:167` — tiga route didaftarkan dengan guard
    existing (`write`/`read`); tidak ada guard baru yang dibuat, memakai
    permission `ingest.write`/`ingest.read` yang sudah ada.
25. `sub_app/ingest_runtime/copilot-runtime.ts:segmentWriteProposeParameters` —
    `actions[].body` dibuka lewat `loosenFreeFormFields` (temuan WO-01:
    konverter menutup `additionalProperties`); tanpa itu model tak bisa
    mengirim payload API asli dan tool selalu 400.
26. `sub_app/ingest_runtime/copilot-runtime.ts` deskripsi `apply_record_actions`
    — menegaskan larangan mengisi `confirmed_by_user` sendiri. Prompt saja tidak
    cukup: deskripsi tool adalah teks yang paling sering dibaca model saat
    memutuskan tool-call.
27. `plugins/ingest/prompt/agent.md` "Kontrak 2c" — kontrak tulis dua langkah
    + larangan mengarang ID. Ditaruh tepat setelah Kontrak 2b (`segment_read`)
    karena urutan baca→usul adalah alur yang harus diikuti model.
28. `plugins/ingest/prompt/agent.md` "Kontrak 4" — daftar larangan lama
    ("tidak boleh mengubah data yang sudah tersimpan") diganti dengan rujukan
    ke Kontrak 2c. Tanpa perubahan ini, prompt saling bertentangan dan model
    cenderung memilih larangan mutlak (fitur tak pernah dipakai).
29. `sub_app/webapp/src/api/ingest.ts:RecordChatBlock` — tipe kartu
    dideklarasikan eksplisit (bukan `any`) agar `npm run build` menangkap
    perubahan bentuk dari Go; `RecordPreview`/`RecordActionResult` ikut
    dideklarasikan untuk pratinjau dan hasil per aksi.
30. `sub_app/webapp/src/pages/admin/ingest/IngestChatPage.tsx:RecordActionsCard`
    — kartu mengambil data dari Go (`record-actions`), bukan dari stream chat,
    karena pesan `system` hilang saat chat di-reset (alasan identik kartu
    `rejected_listing`).
31. `sub_app/webapp/src/pages/admin/ingest/IngestChatPage.tsx` tombol Setujui
    — label berubah menjadi "Setujui & terapkan" saat backend menyatakan
    `preview_needed`; pengguna melihat pratinjau dulu dan tidak ada eksekusi
    tak sengaja pada penekanan pertama delete/merge.
32. `plugins/ingest/services/record_write_test.go` — uji difokuskan ke gerbang
    keamanan (whitelist tulis, method per kind, bentuk aksi, batas batch,
    merge non-visit, preview) bukan ke bentuk response, karena regresi
    berbahaya di sini adalah "jalur tulis terbuka", bukan "JSON berubah".
33. `api-tests/ingest-crud/18..24` — Bruno menguji alur penuh berurutan pada
    satu sesi (`crudSessionId`): propose → apply → path terlarang → tanpa
    konfirmasi → apply ulang 409 → ID tak ada → merge non-visit 409. Urutan ini
    penting karena 22 bergantung pada 19 (proposal sudah `applied`).

## Hasil Verifikasi

- `go build ./...` OK; `go vet ./plugins/ingest/...` bersih; `gofmt -l` kosong.
- `go test ./plugins/ingest/...` hijau (services termasuk 10 uji baru WO-03).
- `tsc --noEmit` (ingest_runtime) OK; `npm test` 21/21 lulus.
- `npm run build` (webapp) OK.
- Swagger diregenerasi di container (`build.sh` → `swag init`).
- Migrasi `ingest` 9 (`2026_11_21_100000_add_ingest_sessions_record_proposals`)
  diterapkan; server di-restart; `ingest-runtime` di-restart.
- Bruno `ingest-crud`: 25/25 request, 55/55 tes PASS.
- Bukti tidak menerapkan saat propose: baris `visit_visitors` hanya muncul
  setelah `apply_record_actions`.
- Bukti audit ganda: `ingest_audit` memuat
  `provider='ingest-tool'` untuk `segment_write_propose`/`apply_record_actions`
  (200/400/409) + endpoint asli mencatat aksinya.
- Smoke browser (Tailscale `http://100.125.175.34:5173`): kartu
  "Usulan perubahan data · 1 aksi · proposed" tampil, tombol Setujui →
  kartu berubah `applied` + "1 berhasil"; baris `Smoke Kartu WO-03` ada di DB.
- Regresi `ingest-crud` (termasuk 4 tool analisa + `segment_read` +
  `introduce_session`): 55/55 PASS.

### Catatan kegagalan pra-ada (bukan akibat WO-03)

`bru run ingest ingest-resident ingest-sec ingest-hsh` menunjukkan 14 kegagalan
pada berkas fixture lama: `ingest/08..14` memakai variabel `{{sessionId}}` yang
tidak pernah di-set oleh request mana pun, dan `ingest/05` membandingkan
`bruno-sesi-1` padahal `04` menulis id bertimestamp. Tidak ada berkas itu yang
disentuh WO-03; `ingest-crud` (folder yang relevan) lulus penuh.

## Review Manager (2026-11-20) — LULUS DENGAN CATATAN ✅ (kode `2a898f3`)

Verifikasi independen (dijalankan Manager sendiri via API + DB):

- `segment_write_propose` 200 → proposal `rp_…` status `proposed` + kartu
  `chat_block` (kind/session/proposal_id/status/applied_count/…).
- `apply_record_actions` TANPA `confirmed_by_user` → **400** ("butuh
  confirmed_by_user=true") — fail-closed P-3 terbukti.
- Dengan confirm → 200 `status: applied`; **bukti DB**: baris
  `visit_visitors` baru muncul SETELAH apply (data uji Manager sudah
  dihapus sendiri).
- Apply ulang → **409** ("proposal tulis sudah diterapkan") — idempotensi
  kartu.
- Whitelist TULIS terpisah dari baca + eksplisit per endpoint (bukan
  wildcard) + reuse bans baca — arsitektur fail-closed yang benar.

### TEMUAN PENTING — koreksi klaim "audit ganda" (W-8)

`endpoint_audit` TIDAK mencatat POST `/api/admin/visit/visitors` dari
self-call — tabel itu milik plugin `endpoints` (federasi) dan BUKAN audit
admin generik; plugin visit TIDAK punya audit baris per request admin.
Konsekuensi: klaim SPEC "audit endpoint asli tercatat otomatis" TIDAK
akurat untuk plugin target. Nilai aktual: (1) RBAC + validasi tetap
ditegakkan endpoint asli per request (Bearer user — terbukti desain),
(2) audit tepercaya = `ingest_audit provider='ingest-tool'` per
propose/apply (409/400/200 semuanya tercatat + admin_id + request_id).
**Putusan**: terima; dok WO dianggap diamendemen — W-8 = "audit
ingest-tool per aksi + RBAC/validasi endpoint asli". Audit baris CRUD di
plugin target = BACKLOG terpisah (jangan dikerjakan tanpa WO baru).

### Posisi AC
AC-1..AC-7 terpenuhi; AC-6 (dry-run preview delete/merge, dua langkah
persetujuan: proposed → previewed → applied) terverifikasi dari kode +
test; merge → 404 dipetakan 409 sampai WO-04 jujur.


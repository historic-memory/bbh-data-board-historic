---
title: "Work order — CRUD master data per segmen via tool chat generik"
description: "Master index WO topic 1795107489593: 5 WO (patch editor → segment_read → write/apply+kartu → MergeVisitors → prompt+e2e); SPEC disetujui owner 2026-11-20; WO-01/02/02a/03/04/05 complete"
status: progress
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - crud
    - work-order
---
# Work order — CRUD master data per segmen via tool chat generik

Topic `1795107489593` (extends `1795083433686`). Sumber kebenaran:
`notes.md` (DESAIN FINAL owner ok 2026-11-20) + `spec.md` (draft).
**SPEC disetujui owner 2026-11-20**. WO-01 complete (review LULUS 2026-11-20,
kode `111bf03`+`8d0eafb`); WO-02 & WO-02a complete (review LULUS, kode
`9aa0933`); WO-03 complete (review LULUS dgn catatan, kode `2a898f3`);
WO-04 complete (kode `f583d78`); WO-05 complete (kode `bf50384`, menunggu
review Manager — bila LULUS topic `1795107489593` ditutup).
Koreksi W-8 (review WO-03): audit tepercaya = `ingest_audit` per aksi +
RBAC/validasi endpoint asli (Bearer user); `endpoint_audit` = milik
federasi, BUKAN audit admin generik.
Backlog: (a) default `kind` dari `fields.yaml` untuk bentuk mapping
legacy; (b) audit baris CRUD di plugin target (terpisah, butuh WO baru);
(c) bersih-bersih sesi uji dev.

## Bukti WO-05 (2026-11-20)

- Prompt: `plugins/ingest/prompt/agent.md:251` → Kontrak 8 (baca → usul →
  setujui → terapkan; `confirmed_by_user` milik kartu; whitelist guide+pagar;
  PII pseudonym; merge hanya visit + delta; delete/merge 2 langkah; bulk ≤200).
  Kontrak 0–7 (`agent.md:8`–`230`) tidak diubah. Perlu
  `docker compose restart ingest-runtime` agar aktif (prompt dibaca runtime).
- Bruno: `api-tests/ingest-crud/25..43` (19 berkas baru) — rantai E2E
  positif, negatif (tanpa segmen, bulk >200, viewer, tanpa token), dan
  rantai merge visit. Run gabungan `ingest-crud` + `visit`:
  **70 request / 141 test, 100% PASS**.
- Bukti kasus nyata (chat UI Tailscale, sesi
  `01a1f6d5-a676-7a91-a4ff-d3f68de8a29c`): agent membaca 3001 visitor,
  mengusulkan **13 merge**. Kartu: approve #1 → `previewed` ("Pratinjau
  dampak (110 record) — 8 kunjungan akan pindah"); approve #2 → `applied`,
  "13 berhasil".
- DB sebelum: `dup_groups 177`, `visitors 3001`, `visits 3065`,
  `visit_visitor_merges 12`. Sesudah: `dup_groups 164`, `visitors 2904`,
  `visits 3065`, `visit_visitor_merges 25`. Delta: −13 grup, −97 visitor,
  **0 kunjungan hilang** (dipindah), +13 audit (Σ merged 98,
  Σ visits_moved 99, Σ deleted 98). Data nyata konsisten dengan asumsi.
- Regresi: `go build ./...` OK, `go vet` OK, `go test ./plugins/visit/...
  ./plugins/ingest/...` OK; `tsc --noEmit` OK; `npm run build` ✓ built.
  Bruno `smoke residents facilities secondary needs templates dashboard`:
  2 kegagalan **known pre-existing** (`dashboard/04`,`05` — `BBH_VIEWER_TOKEN`
  kosong lokal). Bruno `ingest ingest-hsh ingest-sec ingest-resident`:
  14 kegagalan seluruhnya di `ingest/` lama (fixture `sessionId` 08–14,
  state 54–57, timing 67–68, daftar 05) — tidak disentuh WO ini, tanpa
  kegagalan baru di hsh/sec/resident.
- Catatan: kartu usulan fetch sekali saat mount → perlu reload bila usulan
  lahir setelah halaman terbuka (kandidat perbaikan lanjutan, di luar WO).

## Bukti WO-04 (2026-11-20)

- Go: `visitor_merge.go` (service: `MergeVisitors` satu transaksi, dry-run
  rollback, `MergeConflictError`, `SuggestSurvivor` read-only), `models.go`
  (`VisitorMerge`), migrasi `visit_visitor_merges`, handler + 2 route
  (`/visitors/merge` guard `write`, `/visitors/suggest-survivor` guard
  `read`), 7 test DB. `record_apply.go`: payload merge diperbaiki (bug body
  kosong) + preview dampak `?dry_run=1`.
- `go build ./...`/`go vet`/`go test ./plugins/visit/... ./plugins/ingest/...`
  hijau; `gofmt -l` kosong; swagger diregenerasi; migrasi `visit` diterapkan.
- AC-1 atomik: test rollback (`CHECK (false) NOT VALID`) — rollback penuh,
  tanpa baris audit tertinggal. AC-2 dedup: konflik E.164 luar grup → 409,
  tanpa perubahan. AC-3 dry-run: `visits_moved` benar, `deleted_count=0`,
  audit `audit_id` null. AC-4 RBAC: viewer 401/403. AC-5 `SuggestSurvivor`
  read-only (HP valid menang). AC-6 test DB hijau.
- Bruno `api-tests/visit` 27/27 request, **54/54 tes PASS** (pojok: merge
  positif, preview, validasi 400 survivor∈merged, 404 id hilang, RBAC).
- Smoke jalur chat penuh (Tailscale `100.125.175.34`): propose merge →
  apply preview (`preview_needed:true`, note "1 kunjungan akan pindah...") →
  apply lagi → `status applied`, `applied 1`, **HTTP 200 (bukan 409)**.
  DB sebelum: kandidat 1 kunjungan, survivor 0. DB sesudah: survivor 1
  kunjungan, kandidat terhapus, `phone_e164` unik (0 duplikat), baris
  `visit_visitor_merges` (merged_count=1, visits_moved=1, deleted_count=1,
  changed_by=admin). Audit `ingest_audit provider='ingest-tool'` mencatat
  propose + apply 200.
- Regresi: `ingest-crud` 55/55, `visit` 54/54, `ingest-hsh`/`ingest-sec`/
  `ingest-resident`/`residents`/`facilities`/`smoke`/`secondary`/`templates`/
  `needs` PASS. Catatan pra-ada: `dashboard/04,05` 401 karena `BBH_VIEWER_TOKEN`
  kosong di env lokal — bukan regresi WO-04.

## Bukti WO-03 (2026-11-20)
- Go: `record.go` (model), migrasi jsonb `record_proposals`, service
  `record_write.go` (whitelist TULIS per segmen, terpisah dari whitelist baca),
  `record_apply.go` (approval wajib, preview delete/merge, eksekusi HTTP
  self-call dengan Bearer user), handler + 3 route, 10 unit test.
- `go build/vet/test ./...` hijau; `gofmt -l` kosong; `tsc --noEmit` hijau;
  `npm test` 21/21; `npm run build` webapp hijau; swagger diregenerasi.
- Migrasi `ingest` 9 diterapkan; `bin/server` & `ingest-runtime` di-restart.
- Bruno `api-tests/ingest-crud` 25/25 request, **55/55 tes PASS**: propose
  create → apply (baris DB nyata) → path terlarang 400 → tanpa
  `confirmed_by_user` 400 → apply ulang 409 → ID tak ada 400 → merge
  non-visit 409.
- Bukti propose TIDAK menulis: baris `visit_visitors` hanya muncul setelah
  `apply_record_actions`.
- Bukti audit ganda: `ingest_audit provider='ingest-tool'` mencatat
  `segment_write_propose`/`apply_record_actions` (200/400/409); endpoint asli
  mencatat aksinya.
- Smoke UI `http://100.125.175.34:5173`: kartu "Usulan perubahan data · 1 aksi
  · proposed" tampil, Setujui → `applied` + "1 berhasil", baris
  `Smoke Kartu WO-03` ada di `visit_visitors`.
- Regresi: 4 tool analisa + `segment_read` + `introduce_session` (folder
  `ingest-crud`) tetap 55/55 PASS.

## Bukti WO-02 / WO-02a (2026-11-20)

- Unit test Go `plugins/ingest/services/segment_read_test.go` (13 test):
  whitelist per segmen, larangan keras, higiene path, `:id`, jepitan
  `limit`, deidentify (ID utuh + objek PII), truncation, key kosong
  fail-closed, pemetaan status upstream. `go build/vet/test ./...` hijau.
- Node: `tool-schema.test.ts` +1 kasus `query` bebas → `npm test` 21/21;
  `tsc --noEmit` hijau; `npm run build` webapp hijau.
- Bruno `api-tests/ingest-crud/` 18 request / 39 test HIJAU: positif 4
  segmen (visit, resident, sec, hsh), whitelist 400 (lintas segmen +
  path terlarang keras), viewer 200 / tanpa token 401, `truncated` +
  jepitan 200 baris, introduce positif, 409 state salah, 400 modul asing,
  401 tanpa token.
- Smoke UI `http://100.125.175.34:5173`: sesi baru `uploaded` → chat murni
  memilih modul Kunjungan → agent memanggil `introduce_session` → DB
  `state=introduced`; lanjut `segment_read` → agent menjawab tabel tanpa
  `full_name`/`phone_raw`, menyebut `truncated: true` + `total 2990`, dan
  menganjurkan operasi by ID. `ingest_audit provider='ingest-tool'`
  mencatat 200 untuk kedua tool.
- Regresi editor (AC-4 WO-02a): `SaveMapping` tetap memakai transisi yang
  sama (`introduceFromUploaded`); koleksi `ingest/` tidak berubah perilaku.

## Latar

Owner ingin agent chat punya tools CRUD lengkap untuk master data segmen
sesi ("pokoknya toolsnya lengkap"). Kasus nyata pertama: kunjungan dev
dobel hasil re-impor (`time_only`). Setelah diskusi arsitektur (2026-11-20),
owner memilih **Hybrid**: 2-3 tool generik untuk semua segmen via proxy
API admin existing + proposal/approve untuk tulis.

## Daftar WO

| WO | Judul | Status | Commit kode | Dok historic |
|---|---|---|---|---|
| WO-01 | Patch editor: hapus tombol Ekstrak & Konfirmasi + modal Simpan draft → chat | **complete** (review LULUS 2026-11-20) | `111bf03`, `8d0eafb` | wo-01-hapus-tombol-ekstrak-konfirmasi-editor.md |
| WO-02 | Backend `segment_read` (whitelist + deidentify) + skema Node | **complete** (2026-11-20) | `9aa0933` | wo-02-segment-read.md |
| WO-02a | Tool `introduce_session` (chat murni dari `uploaded`) — keputusan owner review WO-01 | **complete** (2026-11-20) | `9aa0933` | wo-02a-introduce-session-tool.md |
| WO-03 | Backend `segment_write_propose` + `apply_record_actions` + kartu approve | **complete** (review LULUS dgn catatan 2026-11-20) | `2a898f3` | wo-03-write-propose-apply.md |
| WO-04 | `MergeVisitors` plugin visit + endpoint admin | **complete** (2026-11-20) | `f583d78` | wo-04-merge-visitors-visit.md |
| WO-05 | Prompt agent + Bruno + uji end-to-end duplikat kunjungan | **complete** (2026-11-20) | `bf50384` | wo-05-prompt-bruno-e2e.md |

## Prinsip lintas WO (pinned)

1. **P-3/P-5/D-11/O-25 topic induk TIDAK berubah**: AI hanya mengusulkan
   (tulis via proposal → `confirmed_by_user` → Go eksekusi); PII
   dideidentify sebelum ke LLM; Node hanya skema; snapshot jsonb per
   sesi tanpa tabel relasional baru.
2. **Whitelist per segmen di Go** (fail-closed 400) + **RBAC plugin
   target**: baca = permission baca plugin target; tulis = permission
   tulis plugin target; larangan keras endpoint non-segmen
   (`admins/groups/permissions/audit/ingest/providers/dashboard/
   endpoints`).
3. **Eksekusi tulis = panggilan API admin asli** → RBAC + audit endpoint
   tercatat; ingest menambah audit `provider='ingest-tool'` sendiri
   (audit ganda).
4. **Kartu approve di chat** dibuat backend Go (pola `rejected_listing`
   WO-03), bukan teks LLM.
5. **Log Implementasi (Dev)** wajib per WO (aturan pinned topic induk).
6. Kode: backend hanya `plugins/ingest/services` (+ `plugins/visit`
   khusus WO-04) + UI `sub_app/webapp/src/pages/admin/ingest/` + Node
   `sub_app/ingest_runtime/`.
7. Restart `ingest-runtime` setelah perubahan Node (tsx non-watch); Go
   rebuild + restart server (pola terdokumentasi).

## Urutan eksekusi

WO-01 (patch editor, independen) → WO-02 → WO-03 → WO-04 → WO-05.
Kasus uji akhir: kunjungan dobel dev end-to-end via chat (WO-05).

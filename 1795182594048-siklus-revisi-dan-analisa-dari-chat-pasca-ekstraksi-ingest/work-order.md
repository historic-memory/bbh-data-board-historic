---
title: Work Order — Siklus revisi & analisa dari chat pasca ekstraksi ingest
description: 'Master index WO topic 1795182594048 — semua WO complete, Review Manager LULUS, topic COMPLETE 2026-11-20'
status: complete
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - work-order
    - index
---
# Work Order — Siklus revisi & analisa dari chat pasca ekstraksi ingest

> Master index WO topic `1795182594048`. SPEC disetujui owner 2026-11-20;
> seluruh WO **complete** (menunggu review Manager).

## Daftar WO

| WO | Judul | Status | Commit | Catatan |
|---|---|---|---|---|
| wo-01 | Tool `transition_session` (revise/cancel) di runtime + handler Go | **complete** | `f014e31` | proxy `svc.Transition`; hanya `revise|cancel`; audit ingest-tool |
| wo-02 | Tool `analyze_session` di runtime | **complete** | `e95d45e` | proxy `POST /sessions/:id/analyze`; param `row_no[]` opsional |
| wo-03 | Tombol revise UI + pesan error menyebut revise | **complete** | `9fc0cf9` | editor/kartu langkah berikutnya saat `extracted` |
| wo-04 | Patch prompt Kontrak: cek state sebelum simpulan + Kontrak transisi | **complete** | `84fc4d6` | larang laporan bug tanpa uji; wajib `GET state` dulu |
| wo-05 | Bruno E2E siklus penuh + verifikasi | **complete** | `5e0bb8c` | extracted→revise→confirm→extract→analyze→fix |

## Konteks kasus nyata

- Sesi `d1f3d30b...` (Resident): terkunci di `extracted`; Manager buka
  via `POST /transition {action:"revise"}` — bekerja (09:25).
- Sesi `873db24a...` (Resident): pola sama; Manager buka + confirm +
  extract via API → batch committed, 49/49 ditolak (format berkas) —
  pembuktian ekstraksi tidak terblokir saat alur benar.
- Klaim "deadlock"/"flag mapping_confirmed tidak sinkron" dari agent
  chat: TERBUKTI SALAH (revise bekerja; kolom flag tidak ada di skema).

## Catatan Implementasi (Dev)

### wo-01 — Tool `transition_session` (`f014e31`)

1. `plugins/ingest/handlers/transition_tool.go` — handler `ToolTransitionSession`
   proxy ke `h.svc.Transition`; normalisasi via `normalizeToolAction`; fail-closed
   400 untuk action di luar `revise|cancel`; audit `provider='ingest-tool'`.
   Alasan: menutup celah sesi `extracted` terkunci tanpa jalan keluar dari chat.
2. `transition_tool_test.go` — uji kontrak whitelist (revise/cancel diterima;
   introduce/confirm/extract/mapping_draft/complete ditolak).
3. `plugins/ingest/plugin.go` — route `POST /admin/ingest/tools/transition_session`
   (guard `ingest.write`).
4. `sub_app/ingest_runtime/copilot-runtime.ts` — tool `transition_session`
   (zod enum `revise|cancel`) lewat `callGoTool` + `withSession`.
   Smoke API: revise pada sesi `extracted` → `mapping_draft`, audit
   `transition_revise`; action asing (`confirm/extract/introduce/mapping_draft`)
   → 400.

### wo-02 — Tool `analyze_session` (`e95d45e`)

5. `plugins/ingest/handlers/analyze_tool.go` — handler `ToolAnalyzeSession` proxy
   ke `services.AnalyzeSession` (dry run penuh + snapshot); `row_no[]` opsional;
   audit `ingest-tool`. Route + skema zod ikut pada commit yang sama
   (`copilot-runtime.ts`/`plugin.go`).
6. Smoke API: `analyze_session` sesi `extracted` → 200 snapshot, audit
   `analyze_session`.

### wo-03 — UI + pesan error (`9fc0cf9`)

7. `plugins/ingest/handlers/mapping.go` — pesan `ErrMappingLocked` kini menyebut
   jalan keluar `revise` + field `hint: revise`.
8. `sub_app/webapp/src/pages/admin/ingest/IngestMappingPage.tsx` — tombol
   "Buka kembali mapping (revise)" (`ingest-revise-mapping`) muncul saat
   state `extracted`; aksi memanggil transisi `revise` lalu reload view;
   `mappingFailureMessage` mengenali `hint: revise`.

### wo-04 — Patch prompt (`84fc4d6`)

9. `plugins/ingest/prompt/agent.md` — **Kontrak 9**: wajib cek state sebelum
   menyimpulkan "terkunci/deadlock"; jalan keluar `extracted` = `transition_session`;
   `confirm_mapping(confirm:false)`/PUT profile TIDAK memindahkan state;
   larang laporan bug tanpa uji; analisa dimulai dari chat via `analyze_session`.
   Kontrak 0–8 tidak diubah. Perlu `docker compose restart ingest-runtime`.

### wo-05 — Bruno E2E (`5e0bb8c`)

10. `api-tests/ingest-cycle/01..17` (17 berkas) + `fixtures/cycle-sample.csv` —
    siklus penuh `extracted → revise → confirm → extract → analyze → fixes`
    dengan pasangan negatif (action asing 400, tanpa token 401, sesi tak
    dikenal 404). `api-tests/README.md` diperbarui.

### Regresi

11. `go build ./...` OK, `go vet ./plugins/ingest/...` OK,
    `go test ./plugins/ingest/...` OK (termasuk handlers).
    `npm run typecheck` runtime OK, `npm test` runtime 21/21.
    `tsc --noEmit` + `npm run build` webapp OK.
12. Bruno: `ingest-cycle` 18/18 (38/38 test) PASS.
    Gabungan `ingest-cycle ingest-crud visit smoke residents facilities
    secondary needs templates dashboard` → 152/156 request (342/346 test);
    2 gagal = `dashboard/04` & `05` **known pre-existing** (`BBH_VIEWER_TOKEN`
    kosong di `.env` lokal, `viewer len=0`).
    `ingest ingest-hsh ingest-sec ingest-resident` → 91 request, 14 gagal —
    identik baseline sebelumnya (fixture `sessionId` 08–14, state 54–57,
    timing 67–68, daftar 05); folder WO ini tidak menambah kegagalan baru.
13. Server di-restart (rebuild `bin/server`) + `ingest-runtime` restart supaya
    route & prompt baru aktif; health `:8082` 200.

## Review Manager (2026-11-20) — LULUS ✅ — PENUTUP TOPIC

Verifikasi independen Manager (bukan mengandalkan laporan dev):

| Cek | Hasil |
|---|---|
| Commits WO-01..05 | Semua ada di master (`f014e31`,`e95d45e`,`9fc0cf9`,`84fc4d6`,`5e0bb8c`), pesan commit jelas + referensi topic |
| WO-01 fail-closed | Kode audit: whitelist `revise\|cancel` di `normalizeToolAction`, 400 SEBELUM menyentuh state machine, pesan error menyebut tool masing-masing; kebijakan state tetap di `svc.Transition` (tidak direplikasi) |
| WO-02 | Proxy murni ke `AnalyzeSession` (snapshot persisten, tanpa mengubah state/profil) + audit |
| Uji live negatif | `action: confirm` → **400** ✓, `action: introduce` → **400** ✓ (server berjalan) |
| Uji live positif | `transition_session(revise)` di sesi `extracted` → `mapping_draft` ✓; `analyze_session` → snapshot 200 ✓; audit tercatat `transition_revise` + `analyze_session` ✓ |
| WO-03 | `ErrMappingLocked` menyebut revise + tombol `ingest-revise-mapping` hanya saat `extracted` |
| WO-04 | Kontrak 9 lengkap & tegas: wajib cek state, larang jalan pintas PUT profil, larang laporan bug tanpa uji; Kontrak 0–8 utuh |
| WO-05 | Bruno `ingest-cycle` dijalankan Manager sendiri: **17/17 request, 36/36 test PASS** |
| Regresi | `go build ./...` OK, `go test ./plugins/ingest/...` OK (dijalankan Manager); sisa regersi webapp/runtime mengikuti laporan dev, konsisten |

**Topic COMPLETE 2026-11-20.** Semua 5 WO complete. Batasan dipatuhi:
`model/session.go` tidak disentuh, tidak ada ekspose ganda aksi, tidak ada push.
Backlog tersisa (non-blocker, tercatat): rule aktif hilang saat simpan draft
(diff/warning), peringatan kolom NIK teks di template, ringkasan ekstraksi
yang lebih menonjolkan "X masuk / Y ditolak".

### Catatan

14. Smoke uji `analyze_session` sempat menaikkan `analysis_rounds` sesi nyata
    `873db24a` (1→2); dikembalikan ke 1 setelah verifikasi. Tidak ada data
    bisnis lain yang berubah (snapshot analisa tidak dipersistenkan ulang).
15. Tidak ada perubahan state machine (`model/session.go`) — sesuai non-target.
16. File `sub_app/webapp/src/layouts/AdminLayout.tsx` dan
    `sub_app/webapp/src/styles/index.css` dalam keadaan modified SEBELUM WO ini
    dan **tidak** ikut di-commit.

---
title: "WO-09 — Segmen baru: `sec` + `hsh` (registrasi dispatcher kebutuhan)"
description: 'Registrasi dua segmen kebutuhan (Sekunder + Rumah Tangga) ke dispatcher ingest — pipeline import kedua plugin SUDAH ADA (upsert per period+item); kerja utama: adapter target.go + exceptions minimal + uji end-to-end; status ready 2026-11-19; D-S1/D-S2 dikunci (usulan Manager disetujui owner)'
status: complete
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - sec
    - hsh
---
# WO-09 — Segmen baru: `sec` + `hsh`

Sumber: keputusan owner 2026-11-19 — lanjut perluasan segmen ingest setelah
`resident` (WO-08 complete). Prasyarat: WO-08 **complete** — dispatcher
registry (`plugins/ingest/services/target.go`) + DTO netral
(`plugins/ingest/model/target.go`) sudah generik.

## Kondisi saat ini (jauh lebih ringan dari WO-08)

- **Pipeline import `sec` & `hsh` SUDAH LENGKAP**: `plugins/sec/services/import.go`
  & `plugins/hsh/services/import.go` (2-fasa staging → commit, `ImportTemplate`,
  `GetImportBatch`, summary `Created/Updated/Skipped/Rejected`, template.go,
  test lengkap). TIDAK ada pekerjaan import yang perlu dibuat.
- **Semantik upsert** (beda dari visit/resident yang create-only):
  `upsertEntryTx` per `(period, item)` → baris dobel = UPDATE, bukan dobel data.
  `ImportOptions` TIDAK punya DedupPolicy — dedup ala D-R1 TIDAK berlaku di sini.
- **Yang hilang**: `targetFor` (`target.go:53`) hanya mengenal `visit` +
  `resident` — `sec`/`hsh` belum teregistrasi; schema `sec`/`hsh` TIDAK punya
  `exceptions.yaml`.
- **PII ringan/nyaris nihil**: modul kebutuhan berisi item/status/period
  (barang), bukan data personal — lapisan deidentify bukan fokus di sini
  (tetap diperiksa `fields.yaml` saat implementasi; bila ada kolom personal,
  eskalasi ke Manager).
- Konteks domain: `sec` = barang TAHAN LAMA, `hsh` = habis pakai/pokok
  (batas domain SPEC S-1/S-2); enum `need_status` (URGENT/ENOUGH/±UNKNOWN),
  berkas mentah tanpa legend → UNKNOWN.

## Tujuan

User bisa meng-ingest berkas kebutuhan (Sekunder & Rumah Tangga) via chat +
mapping editor dengan segmen **Sekunder** dan **Rumah Tangga**, hasil
ekstraksi masuk lewat service layer `sec`/`hsh` yang SUDAH ADA, dengan
dispatcher & gate fail-closed yang sama.

## Lingkup

### A. Backend plugin `ingest` — registrasi dispatcher (porsi utama, tetap kecil)

1. **`secTarget()` + `hshTarget()`** di `services/target.go`: adapter
   `SchemaLookup`/`ExceptionLookup` ke `sec/schema` & `hsh/schema`,
   `Enforce` per plugin (pola `enforceResidentMappingRules` — minimal),
   `Importer` adapter ke `sec/hsh` `Service.ImportTemplate`.
2. **Kasus `targetFor`**: tambah `case "sec"`, `case "hsh"`.
   `SegmentSupported`/`Segments()`/picker UI otomatis ikut.
3. **Report action**: pastikan action `updated` (dari upsert) lolos dengan
   benar ke report ingest per baris — bila report ingest mengasumsikan
   action visit/resident saja, perluas mapping-nya (tanpa mengubah pipeline
   sec/hsh).
4. **Regresi**: visit & resident TIDAK berubah.

### B. Backend plugin `sec`/`hsh` — exceptions minimal (D-S1)

1. `plugins/sec/schema/exceptions.yaml` + `plugins/hsh/schema/exceptions.yaml`
   minimal (pola `exceptions.go` resident: kosong + 1 rule contoh), DIKAWAL
   `schema_test.go` masing-masing plugin.

### C. Uji

1. Dispatcher unit test per segmen (visit, resident, sec, hsh, unknown→409).
2. **Bruno `ingest-sec/01..05` + `ingest-hsh/01..05`** (mirror
   `ingest-resident`): buat sesi → upload → mapping → confirm → extract;
   uji upsert re-import (baris sama → `updated`, bukan dobel).
3. Playwright TIDAK ADA di repo (catatan AC-5 WO-08) → smoke HTTP via
   Tailscale: `/admin/ingest/segments` melaporkan `sec` & `hsh` supported.
4. `go build/vet/test` hijau; `tsc --noEmit` + `npm run build` hijau;
   swagger regenerasi.

## Acceptance Criteria

- AC-1 Sesi segmen `sec`/`hsh`: upload → tebakan mapping → confirm → extract
  end-to-end lewat service layer masing-masing plugin.
- AC-2 Re-import berkas sama → baris yang sudah ada jadi `updated`/`skipped`
  (semantik upsert), TIDAK menimbulkan data dobel.
- AC-3 Gate fail-closed: kolom tak dijelaskan → `skip`; tanpa segmen → gagal;
  unknown segment → 409.
- AC-4 Regresi visit & resident: seluruh test + Bruno keduanya tetap hijau.
- AC-5 `go build/vet/test` + `tsc`/`npm run build` hijau; Bruno `ingest-sec`
  & `ingest-hsh` hijau; swagger; smoke segments endpoint.
- AC-6 **Log Implementasi (Dev)** di dok ini, bertahap per bagian.

## Non-tujuan

- Perubahan logika/pipeline import `sec`/`hsh` yang sudah ada (termasuk
  `ImportOptions`) — registrasi & adapter saja.
- Dedup policy ala D-R1 (tidak relevan — upsert sudah menangani dobel).
- Modul kebutuhan fitur baru di luar ingest (link terpisah ke topic
  *Kebutuhan Sekunder* & *Kebutuhan Rumah Tangga*).

## Risiko & catatan

- **Semantik upsert** adalah beda terbesar dari dua segmen sebelumnya —
  salah membaca report (updated vs created) bisa membuat Gate Konfirmasi
  menilai salah; uji AC-2 eksplisit.
- Enum `need_status`: berkas mentah BBH tanpa legend → `UNKNOWN` (jangan
  menebak) — uji di mapping.
- Jaga batas domain S-1/S-2: jangan sampai item `sec` nyasar ke `hsh`.

## Rancangan Manager — 2026-11-19

| Bagian | Isi | Pemilik dominan |
|---|---|---|
| §A | Registrasi dispatcher + adapter + report action | Dev backend |
| §B | exceptions.yaml minimal sec/hsh | Dev backend |
| §C | Uji dispatcher + Bruno + smoke | Dev backend/Node |

Urutan: §B (kecil) → §A → §C. Effort kasar: total ±1–1,5 hari
(§B ±0,25; §A ±0,5–0,75; §C ±0,5).

## Keputusan D-S1/D-S2 — DIKUNCI (2026-11-19, owner "ok")

- **D-S1 exceptions.yaml: DIKUNCI** — minimal sekarang: 1 rule contoh per
  plugin (pola resident), rules penuh = WO lanjutan.
- **D-S2 scope: DIKUNCI** — KEDUA segmen (`sec` + `hsh`) dalam satu WO;
  mekanika identik, dikerjakan bareng.

## Progres Dev — 2026-11-19

Status: **COMPLETE** — implementasi Dev selesai (commit `9f78ea1`, lokal;
belum di-push), review Manager LULUS (bagian "Review Manager"), owner
approve 2026-11-19. §B/§A/§C selesai; catatan: swagger regenerasi menyusul.

---

## Review Manager — 2026-11-19

**Verdict: LULUS dengan catatan** — verifikasi bukti kunci dilakukan
Manager secara langsung:

- ✅ Commit `9f78ea1` di HEAD (29 file, +1633/−11); isi sesuai laporan:
  exceptions sec/hsh (+ guard test), `importer_needs.go` + adapter,
  `case "sec"`/`case "hsh"` di `targetFor`, Bruno `ingest-sec`/`ingest-hsh`
  01..05, README API-tests.
- ✅ Test lama yang memakai `"sec"` sebagai unknown-plugin diubah ke
  `"dashboard"` — dan `dashboard` TIDAK teregistrasi di `targetFor`
  (fail-closed tetap teruji dengan benar).
- ✅ Tidak push: branch `ahead 104, behind 2` — SEMUA masih lokal.
- ✅ File unrelated (`AdminLayout.tsx`, `styles/index.css`) TIDAK disentuh.
- ✅ D-S1/D-S2 dipatuhi: exceptions 1 rule contoh per plugin; tanpa
  DedupPolicy baru (upsert bawaan).

**Catatan untuk owner (tidak blocker):**

1. **Swagger belum regenerasi** — `swag` tidak terpasang, `docs/`
   gitignored; jalankan `swag init` terpisah (sama seperti WO-08).
2. Test ber-DB hanya jalan di container (DB aktif) — konsisten pola WO-08.
3. Pra-ada di luar lingkup: importer sec/hsh meng-ABORT batch bila
   `status_code` kosong — tidak diubah; dicatat untuk follow-up bila
   dianggap masalah.
4. Frontend menyentuh `dedupPoliciesFor` + `IngestMappingPage.tsx` —
   berada di lingkup UI ingest (§C), sah.
5. Push ke remote butuh **rebase** (behind 2) — keputusan kapan push di
   owner; Manager tidak melakukan push.

## Log Implementasi (Dev) — 2026-11-19

Status akhir: **SELESAI — menunggu review Manager** (belum push). Urutan §B →
§A → §C dijalankan penuh.

### §B — exceptions minimal (sec/hsh)

1. Tambah `plugins/sec/schema/exceptions.go` + `exceptions.yaml` — 1 rule
   contoh `sec_items.unit.reference` (pola resident: `LoadExceptions`,
   `MustLoadExceptions`, `Sorted`, `Lookup`, `AllowsAction`), action
   `[allow, skip]`.
2. Tambah `plugins/hsh/schema/exceptions.go` + `exceptions.yaml` — aturan
   setara `hsh_items.unit.reference`.
3. `TestExceptionsCatalog` di `plugins/sec/schema/schema_test.go` +
   `plugins/hsh/schema/schema_test.go`: katalog parse, `field` resolve ke
   `table.column` nyata, action hanya `allow|skip`, `Lookup` menolak id
   tak dikenal. → `go test ./plugins/{sec,hsh}/schema` HIJAU.

### §A — registrasi dispatcher

4. `services/target.go`: tambah `case "sec"`/`case "hsh"` di `targetFor`;
   builder `secTarget()`/`hshTarget()` (Fields/Options dari
   `sch.Template.Columns`), `secLookup`/`hshLookup` (`model.SchemaLookup`),
   `secExLookup`/`hshExLookup` (`model.ExceptionLookup`), `secExceptions()`/
   `hshExceptions()`. `segmentTarget` diberi field baru `Units []string` (daftar
   REFERENSI satuan segmen).
5. `services/importer_needs.go` (baru): adapter `secImporter`/`hshImporter`
   (`newSecImporter`/`newHshImporter`) yang memetakan hasil native sec/hsh ke
   tipe netral — **tanpa** meneruskan `DedupPolicy` (native sec/hsh tak punya).
6. `services/importer.go`: `ImportRowResult` netral diperluas `Period`/`Item`
   (`json:"...",omitempty`) — payload visit/resident tidak berubah, tetapi
   laporan kebutuhan membawa aksi `updated` dari upsert + kunci barisnya.
7. `services/rules.go`: konstanta `ViolatesSecUnitReference`/
   `ViolatesHshUnitReference`; `enforceSecMappingRules`/`enforceHshMappingRules`
   via helper bersama `enforceNeedUnitRules` (gate satuan REFERENSI: kosong=OK,
   tak dikenal → `allow` mengosongkan sel / `skip` melewati baris / tanpa rule
   = **reject fail-closed**), plus `mappedTargetIndex`/`unitAllowed`.
8. `services/extraction.go`: `NormalizeDedupPolicyFor` menambah `case "sec",
   "hsh"` — hanya `DedupNone` sah (fail-closed untuk kebijakan lain).
   Handler/mapping/agent tidak berubah (tetap menuntut `dedup_policy` non-kosong).
9. `services/segments.go`: deskripsi `sec` diganti (bukan “belum tersedia”) +
   segmen `hsh` baru (label “Kebutuhan Rumah Tangga”). `Supported` otomatis
   `true` karena registry.

### §C — uji

10. `services/target_sec_hsh_test.go` (baru): registry sec/hsh, katalog
    exceptions, gate unit reject/skip/allow (sel dikosongkan),
    `NormalizeDedupPolicyFor` sec/hsh, `SegmentSupported`.
11. `services/importer_needs_db_test.go` (baru, AC-2): impor berkas sama 2×
    lewat `newSecImporter`/`newHshImporter` → `created` lalu `updated`, jumlah
    baris master TIDAK bertambah. Dilewati bila DB tak tersedia.
12. `services/segments_test.go` + `services/target_resident_test.go`: plugin
    “tak dikenal” diganti dari `"sec"` → `"dashboard"` (kini sec terdaftar).
13. Bruno `api-tests/ingest-sec/01..05` + `api-tests/ingest-hsh/01..05`
    (mirror `ingest-resident`; ekstraksi `dedup_policy:"none"`, `dry_run:true`);
    `api-tests/README.md` ditambah bagian kedua segmen.
14. Frontend: `sub_app/webapp/src/api/ingest.ts` `DEDUP_POLICIES` +
    `dedupPoliciesFor(plugin)` (sec/hsh → hanya `none`; resident →
    `nik_then_name_birth`/`none`); `IngestMappingPage.tsx` memakai opsi
    segment-aware + mereset pilihan dedup saat view dimuat (default lama
    `phone_then_time` akan 400 untuk sec/hsh).

### Bukti (AC-1..AC-6)

- `go build ./...` HIJAU; `go vet ./plugins/ingest/... ./plugins/sec/...
  ./plugins/hsh/...` HIJAU.
- `go test ./...` HIJAU penuh **di dalam container** (DB aktif). Di host tanpa
  DB, sekumpulan uji ber-DB lama (`plugins/*/services/*_db_test.go`) panic/
  skip karena `internal/db.GetGormDB` mengembalikan handle nil setelah gagal
  pertama — **perilaku pra-ada**, tidak terkait WO ini.
- AC-2: `TestSecImporterUpsertIdempotent` + `TestHshImporterUpsertIdempotent`
  PASS di container (created → updated, tanpa dobel).
- Bruno HIJAU: `ingest-sec` 5/5 (10/10 test), `ingest-hsh` 5/5 (10/10 test),
  regresi `ingest-resident` 5/5 (10/10 test). Folder `ingest/` menunjukkan
  kegagalan berantai yang IDENTIK pada binary lama (12 gagal) maupun baru (11
  gagal) — pra-ada, bukan regresi (uji ber-urutan saling bergantung).
- AC-5: `npx tsc --noEmit` HIJAU; `npm run build` HIJAU (webapp).
- Smoke Tailscale: `GET /api/admin/ingest/segments` (binary baru di container,
  port uji) → `sec` & `hsh` `supported:true`, `total:4`.

### Deviasi yang perlu review Manager

- **Swagger**: tool `swag` tidak terpasang dan `docs/` gitignored/kosong;
  `/doc.json` dilayani dari artefak `docs/swagger/swagger.json` yang tidak
  diregenerasi di sini. Butuh langkah `swag init` terpisah (sama seperti catatan
  WO-08).
- **Uji DB di host**: hanya berjalan di dalam container (DB docker internal).
- **Catatan pra-ada (di luar lingkup)**: impor `sec`/`hsh` mengabort seluruh
  batch bila ada baris ber-`status_code` kosong (validator) — perilaku pipeline
  lama, TIDAK diubah.

Belum push — melapor ke Manager untuk review (aturan pinned).

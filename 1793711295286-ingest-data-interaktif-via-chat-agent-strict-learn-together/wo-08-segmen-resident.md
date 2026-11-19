---
title: "WO-08 — Segmen baru: `resident` (porting pipeline import)"
description: 'Segmen ingest kedua setelah visit: porting pipeline import (staging 2-fasa + gates + dedup + PII) ke plugin resident, generalisasi dispatcher ekstraksi ingest, rules minimal; complete 2026-11-19 (owner approve); D-R4: birth_date TANPA encrypt'
status: complete
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - resident
    - import
---
# WO-08 — Segmen baru: `resident`

Sumber: keputusan owner 2026-11-19 — segmen ingest berikutnya setelah `visit`
adalah `resident`. Prasyarat: WO-03 (CP-3) & WO-06 **complete** — picker
segmen (WO-06 §A.1) sudah meng-enum plugin teregistrasi tanpa perubahan
skema; yang belum ada adalah pipeline import-nya.

## Kondisi saat ini

- **Katalog field siap**: `plugins/resident/schema/fields.yaml` ada (single
  source of truth P-13; header Excel Indonesia di `template.columns`, D-7);
  `schema.go` + `schema_test.go` lengkap.
- **Pipeline import BELUM ada** di resident — `services/import.go` +
  `ingest_hooks.go` hanya di visit (pola: validasi → staging 2-fasa
  `*_import_batches`/`*_import_rows` → commit; report per baris).
- **Ekstraksi ingest masih hardcode `visit`** — `plugins/ingest/services/extraction.go:105`
  (`session.TargetPlugin != "visit"`), dispatch langsung ke
  `visitservices.New(s.db).ImportTemplate(...)`.
- **`exceptions.yaml` hanya di visit** (rules O-20, WO-03); resident belum punya.
- Gates & katalog mapping (`MappingOptions(plugin)`, `evaluateGates`,
  `ErrUnknownTarget`/`ErrDuplicateTarget`/`ErrPolicyRelaxed`) SUDAH generik
  per plugin — dicek terhadap `fields.yaml` plugin target.
- Catatan penamaan: `fields.yaml` resident menyebut "WO-08/WO-09" milik
  **stream WO internal plugin resident (topic lain)** — nomor WO di topic ini
  independen; selalu rujuk path, bukan nomor.

## Tujuan

User bisa meng-ingest berkas data resident via chat + mapping editor dengan
segmen **Resident**: hasil ekstraksi masuk lewat service layer resident
(validasi → staging → import) dengan gerbang fail-closed yang sama dengan
visit, tanpa mengubah model mapping v2 maupun state machine.

## Lingkup

### A. Backend plugin `resident` — porting pipeline import (porsi besar)

1. **Migrasi** tabel staging `resident_import_batches` / `resident_import_rows`
   (pola `plugins/visit/model` — kolom raw JSON + parsed values, status batch,
   template_version).
2. **`services/import.go`**: `ImportTemplate(name string, data []byte, opts
   ImportOptions) (*ImportResult, error)` + `commitImport` + `GetImportBatch`
   (+ `ImportSummary`/`ImportRowResult` untuk enforcement jejak Gate Aturan);
   resolve branch key bila pola resident memerlukan; normalisasi enum via
   `enums.go` (gender/religion/blood_type/delivery_method/dll — codes HARUS
   persis, header Indonesia dari `template.columns`).
3. **`services/ingest_hooks.go`**: helper ter-export untuk ingest (pola
   `ExportedNormalizePhoneE164`/`ExportedParseVisitSchedule` di visit) —
   parse tanggal/NIK/phone khas resident.
4. **Dedup policy resident — MENUNGGU KEPUTUSAN owner (D-R1)**; implementasi
   mengikuti pola `dedupKey`/`findVisitByDedupKey` di visit.
5. **Template**: pastikan generator XLSX/CSV resident mengikuti
   `fields.yaml` (pembanding: `plugins/visit/services/template.go`).
6. **`plugins/resident/schema/exceptions.yaml`** minimal (D-R3) — pola
   `exceptions.go` visit; boleh mulai kosong + satu rule contoh.

### B. Backend plugin `ingest` — generalisasi dispatcher segmen (porsi kecil)

1. `extraction.go`: ganti hardcode `visit` dengan **dispatch per plugin**
   (map/registry importer per `TargetPlugin`; interface kecil menyerupai
   `ImportTemplate(...)`). Unknown target → fail-closed 409.
2. Regresi `evaluateGates`/`MappingOptions` untuk resident:
   `ErrPolicyRelaxed` dicek terhadap `fields.yaml` resident (kolom PII wajib
   — NIK, data anak — TIDAK bisa dilonggarkan).
3. Guard `segment_warning` (WO-06) otomatis bekerja untuk katalog resident —
   cukup diuji, bukan diubah.

### C. Agent & UI (porsi kecil)

1. `agent.md`/prompt: tanpa perubahan besar — katalog dikirim via payload;
   UJI konteks segmen "Resident" muncul di system prompt & tebakan mapping.
2. UI: badge/picker otomatis dari endpoint segmen; uji label & alur upload
   segmen resident di daftar sesi/editor/chat.

## Acceptance Criteria

- AC-1 Sesi segmen `resident`: upload → tebakan mapping → confirm → extract
  jalan end-to-end lewat service layer resident (bukan jalur visit).
- AC-2 Gate fail-closed: kolom tak dijelaskan → `skip`; policy PII
  `fields.yaml` resident tidak bisa dilonggarkan (`ErrPolicyRelaxed`).
- AC-3 Dedup sesuai keputusan D-R1; re-import idempoten (regresi AC-8 CP-2).
- AC-4 Dispatcher: segmen `visit` TIDAK regresi — seluruh test visit hijau.
- AC-5 `go build/vet/test` hijau; `tsc --noEmit` + `npm run build` webapp
  hijau; Bruno end-to-end segmen resident; Playwright smoke via Tailscale
  (`BASE_URL=http://100.125.175.34:5173`); swagger + Bruno untuk endpoint baru.
- AC-6 **Log Implementasi (Dev)** diisi bertahap per bagian (aturan pinned).

## Non-tujuan

- Multi-segmen dalam satu sesi (opsi A/C analisa — dicadangkan).
- `exceptions.yaml` lengkap O-20 untuk resident (WO lanjutan).
- Modul resident lain (dokumen, kelahiran/admisi) — hanya target yang
  diputuskan di D-R2.
- Perubahan model mapping v2, state machine, atau UI editor inti.

## Risiko & catatan

- **Data paling sensitif**: resident memuat NIK & data anak — deidentify
  (D-2/D-5) WAJIB sebelum payload LLM; kolom ber-flag `is_encrypt` hanya boleh
  menambah proteksi; audit + RBAC tetap.
- Dedup salah → data ganda/hilang; kalau ragu, pilih kebijakan paling ketat.
- Enum codes domain (mis. kode agama) harus persis — kesalahan header mapping
  adalah sumber bug terbesar; preview ±5 baris wajib dipakai.

## Rancangan Manager — 2026-11-19 (disetujui owner "ok lanjut")

| Bagian | Isi | Pemilik dominan |
|---|---|---|
| §B | Dispatcher ekstraksi ingest + regresi visit (dikerjakan DULU — interface) | Dev backend |
| §A | Pipeline import resident (migrasi, import.go, hooks, template, rules) | Dev backend |
| §C | Uji agent/UI + dokumen | Dev Node/frontend |

Urutan: §B → §A → §C. Effort kasar: §B ±0,5 hari; §A ±2–3 hari; §C ±0,5–1 hari.

## Keputusan D-R1..D-R3 — DIKUNCI (2026-11-19, owner "ok lanjut")

- **D-R1 Dedup policy resident: DIKUNCI** — `NIK` → `nama + tanggal_lahir`
  → tanpa dedup (phone keluarga TIDAK dianggap identitas unik). Catatan
  penting owner: **mesin dedup TIDAK diimplement ulang** — mekanisme
  staging 2-fasa → commit (dedupKey in-file + find*ByDedupKey ke DB,
  `plugins/visit/services/import.go:247-299`) dipakai ulang apa adanya;
  yang beda per modul hanya KUNCI-nya, bukan tempat logic-nya.
- **D-R2 Target data: default Manager aktif** — data core resident
  (tabel resident) dulu; alur penerimaan = WO terpisah (kaitan topic
  *Import Data Penerimaan Resident (CSV Notion & Template XLSX)*).
  Dapat direvisi owner sebelum §A mulai.
- **D-R3 `exceptions.yaml`: default Manager aktif** — minimal (kosong +
  1 rule contoh, pola visit); rules penuh O-20 = WO lanjutan.
  Dapat direvisi owner sebelum §A mulai.

## Progres Dev — 2026-11-19

Status: **COMPLETE** — owner approve 2026-11-19. Implementasi Dev selesai
(commit `ac3ca5e`, lokal; belum di-push). Verifikasi Manager LULUS
(bagian "Review Manager"), D-R4 diputuskan owner (lihat bawah).

**Prinsip eksekusi (dari diskusi owner):** ini replikasi — SEMUA pola dari
visit dipakai ulang apa adanya (staging 2-fasa, mesin dedup + kebijakan D-R1,
gates fail-closed, report per baris, template). Yang berubah hanya:
kunci dedup (D-R1), katalog field resident (sudah ada di `fields.yaml`),
enum resident, dan PII policy per fields.yaml resident.

| Bagian | Status |
|---|---|
| §B Dispatcher ekstraksi | **complete** — approve owner 2026-11-19 |
| §A Pipeline import resident | **complete** — approve owner 2026-11-19 |
| §C Uji agent/UI | **complete** — approve owner 2026-11-19 |

---

## Review Manager — 2026-11-19

**Verdict: LULUS dengan catatan** — verifikasi bukti kunci dilakukan
Manager secara langsung (bukan full code-review baris-per-baris):

- ✅ Commit `ac3ca5e` ada di HEAD; isi sesuai laporan (ingest target.go +
  model/target.go, resident import.go/ingest_hooks.go, exceptions.yaml,
  Bruno `residents/15..17` + `ingest-resident/01..05`).
- ✅ Dispatcher fail-closed: `targetFor` ok=false untuk plugin tak
  teregistrasi; pemanggil menolak 409.
- ✅ D-R1 diterapkan: `nik_then_name_birth` (default; `""` = perilaku lama
  sama) + `DedupNone`; mesin dedup visit dipakai ulang, hanya kunci yang
  berbeda — sesuai keputusan owner.
- ✅ Tidak push: branch `ahead 103, behind 2` — SEMUA masih lokal.
- ✅ File unrelated (`AdminLayout.tsx`, `styles/index.css`) TIDAK disentuh.

**Catatan untuk owner (tidak blocker):**

1. **D-R4 — DIKUNCI owner (2026-11-19: "enggak usah")** —
   `residents.birth_date` TETAP tanpa flag `encrypt` (tanggal lahir =
   quasi-identifier; NIK sudah ketat). Tidak ada perubahan kode.
2. `isDigits16` di `ingest/services/rules.go` duplikasi
   `residentservices.ExportedValidNIK` — unifikasi di follow-up kecil.
3. **Deviasi AC-5**: repo tidak memiliki suite Playwright, sehingga smoke
   browser diganti HTTP smoke via Tailscale (terima sebagai bukti setara;
   dicatat agar AC tidak dianggap terlewat diam-diam).
4. Push ke remote butuh **rebase** (behind 2) — keputusan kapan push ada di
   owner; Manager tidak melakukan push.

## Log Implementasi (Dev) — 2026-11-19

Semua pola dari `visit` dipakai ulang apa adanya (staging 2-fasa, mesin dedup,
Gate Aturan, report per baris, template); yang berubah hanya kunci dedup (D-R1),
katalog field/enum resident, dan policy PII per `fields.yaml` resident.

### §B — Dispatcher ekstraksi `ingest` (selesai)

1. Registry segmen tunggal `targetFor(plugin)` di `services/target.go`
menggantikan hardcode `visit` (`extraction.go:105`); entri `visit` + `resident`,
unknown → `false`.
2. `RunExtraction` dispatch per segmen (`target.Importer` + `target.Enforce`);
segmen tanpa importer → `ErrExtractionTargetUnsupported` → **409** (fail-closed).
Visit tetap lewat jalur & logika yang sama.
3. Tipe netral `model/target.go` (`SchemaLookup`/`ExceptionLookup`/`FieldSpec`/
`ExceptionSpec`) memutus impor-siklus: `ingest/model` tak lagi bergantung schema
visit/resident.
4. `MappingDocument.Validate(sch SchemaLookup)` & `RuleSet.Validate(list
ExceptionLookup)` kini generik; adapter tipis memetakan `fields.yaml`/
`exceptions.yaml` apa adanya → perilaku visit tidak berubah.
5. `ExceptionInfo` menjaga bentuk JSON **identik** dengan `schema.Exception` lama
(`id/field/rule/label/doc/action`) → editor mapping tidak berubah.
6. `schemaHeaderCatalog(plugin)`/`guessColumn(plugin,…)` dibuat sadar-segmen
(baca `targetFor(plugin).Options`), bukan hardcode katalog visit.
7. Penegak Gate Aturan dipecah: `enforceVisitMappingRules` (verbatim) +
`enforceResidentMappingRules` (minimal, D-R3) via `EnforceMappingRules(plugin,…)`.
8. PII fail-closed (AC-2): field `encrypt: true` di `fields.yaml` ditolak bila
`kind: none` (`ErrPolicyRelaxed`) untuk segmen mana pun.
9. `handlers/mapping.go` + `handlers/agent_tools.go` memetakan
`ErrExtractionTargetUnsupported` → **409**.
10. Adapter importer netral (`visitImporter`/`residentImporter`) + DTO netral
(`ImportOptions/Summary/RowResult/Result`); baris netral punya `resident_id`
opsional (`omitempty`) sehingga JSON lama tak berubah.
11. Uji: `target_resident_test.go` (registry, gate NIK fail-closed/skip/allow,
policy PII, segmen tak dikenal 409), `segments_test.go` (resident `supported`,
sec unsupported, konteks label/description). Seluruh suite `visit` hijau (AC-4).

### §A — Pipeline import `resident` (selesai)

12. Migrasi staging `resident_import_batches`/`resident_import_rows` (postgres,
pola visit); **diterapkan ke DB dev** (`migrate up --plugin resident` → versi 2).
13. `services/import.go`: `ImportTemplate` 2-fasa (staging audit → commit dalam
satu transaksi) + `commitImport`/`GetImportBatch`; mesin dedup visit dipakai
ulang — yang beda per modul hanya KUNCI (D-R1).
14. Dedup D-R1: `dedupKeyResident` (in-file) + `findResidentByDedupKey` (DB):
`NIK` → `lower(nama)+tanggal_lahir` → tanpa dedup; HP keluarga **tidak pernah**
identitas unik. Kebijakan `none` mematikan dedup nama+tanggal (NIK tetap dijaga
unique-index DB).
15. `sanitizeNIK` + `demoteNIKIssues`: NIK non-16-digit **tidak lagi menolak
baris** — jadi catatan + disimpan NULL; fase 2 (`createImportedResident`) juga
memakai `sanitizeNIK` supaya master & staging konsisten.
16. `services/ingest_hooks.go`: `ExportedValidNIK`, `ExportedParseTemplateDate`,
`normalizeGender`/`normalizeEnum`/`normalizeBloodType` (dipakai bersama validator).
17. `resolveFacilityID`: cocokkan `facilities.name` ATAU `facility_aliases.alias`
(case/space-insensitive); **tidak pernah** membuat fasilitas/alias baru; tak
cocok → NULL.
18. Enum/status: `admissions.admitted_at` jadi `effective_at` riwayat status bila
ada, selain itu `time.Now()`; `resident_code` eksplisit salah/gender-mismatch →
baris **DITOLAK** (tanpa regenerasi diam-diam).
19. `handlers/import.go` + rute: `POST /api/admin/resident-template/import`
(`resident.write`), `GET /api/admin/resident/import-batches/:id` (`resident.read`)
+ header `X-Import-*`.
20. `schema/exceptions.{yaml,go}` minimal (1 rule `residents.nik.length`, action
`allow`|`skip`) — D-R3.

### §C — Agent/UI & verifikasi (selesai)

21. `agent.md` tak berasumsi `visit` (katalog dikirim via payload); komentar &
deskripsi swagger tool `run_extraction`/`/mapping/extract` diperbarui netral-segmen.
22. UI ingest membaca `/admin/ingest/segments` (picker/badge) — label **Resident**
muncul otomatis; **tanpa perubahan kode frontend**.
23. Uji baru: `services/import_unit_test.go` (dedup key, sanitize NIK, demotion,
pointer int, enum) + `services/import_db_test.go` (idempotensi, dedup in-file,
`none`, demotion NIK, dry run, resolusi fasilitas, audit batch).
24. Bruno: `residents/15..17` (template, import dry-run, get batch) + folder
`ingest-resident/01..05` (sesi segmen resident → upload → mapping → confirm →
extract lewat jalur resident). **Semua hijau**.

### Verifikasi (AC-1..AC-6)

- `go build ./...`, `go vet ./plugins/ingest/... ./plugins/resident/...`: hijau.
- `go test ./...` (dengan env DB): hijau — visit tidak berubah (AC-4).
- `npx tsc --noEmit` + `npm run build` (webapp): hijau.
- `migrate up --plugin resident`: `resident_import_batches`/`_rows` ada.
- Swagger regenerasi (`swag init -g cmd/server/main.go --parseDependency
  --parseInternal`); `docs/` gitignored.
- Bruno `residents/15..17` + `ingest-resident/01..05`: hijau (AC-1, AC-3, AC-5).
- Smoke HTTP via Tailscale (`http://100.125.175.34:8082`): `/admin/ingest/segments`
  menandai `resident` `supported: true`; `resident-template?format=csv`
  mengembalikan header template. (Catatan: repo ini **tidak** punya suite
  Playwright — `playwright.config`/`*.spec.ts` tak ada; smoke browser diganti
  smoke HTTP Tailscale.)

### Untuk Manager (butuh keputusan/penjadwalan WO lanjutan)

- **`residents.birth_date` sengaja TIDAK diberi `encrypt: true`.** Bila tanggal
  lahir + nama dianggap kombinasi PII yang wajib dideidentifikasi, tambahkan
  `encrypt: true` pada `residents.birth_date` (dan `mothers.full_name` sudah).
- **`isDigits16`** di `ingest/services/rules.go` menduplikasi
  `residentservices.ExportedValidNIK`. Bisa disatukan (import `resident/services`
  dari `ingest/services` sudah lazim); visit tidak disentuh.
- **Data uji**: rangkaian Bruno/uji memakai `dry_run` → tidak ada data master
yang tertinggal; baris staging `resident_import_*` dari smoke dibiarkan sebagai
jejak audit (aman dihapus bila perlu).

Jabatan: Dev Agent
Nama Tugas: WO-08 segmen ingest `resident`
Sayonara

---
title: "Mekanisme mapping berkas (ingest strict)"
description: "Analisa alur mapping kolom berkas: state machine, profil v2, dua jalur (editor & chat), gate, ekstraksi via dispatcher generik, dan segmen yang didukung (visit/resident/sec/hsh)"
status: complete
created: "2026-11-18"
updated: "2026-11-19"
tags:
  - ingest
  - mapping
  - arsitektur
---
# Mekanisme mapping berkas

## Tujuan

Dokumen belajar alur kedua flow ingest strict: setelah berkas ter-upload,
bagaimana kolom berkas dipetakan ke field modul tujuan sampai siap ekstraksi.

## Sumber

- Topic induk: `1793711295286` (Ingest data interaktif via chat).
- Kode: `plugins/ingest/` (Go), `sub_app/webapp/` (SPA editor), `sub_app/ingest_runtime/` (Node).
- Temuan awal (2026-11-18); **direvisi 2026-11-19** untuk mencerminkan
  perubahan besar topic induk: analisa multi-segmen (keputusan B+),
  WO-06 segmentasi sesi eksplisit, WO-07 penanganan `extracting` stale,
  WO-08 segmen `resident`, WO-09 segmen `sec`+`hsh`, WO-05 custom
  provider LLM, dan WO-10 housekeeping retensi.

## 1. Posisi di state machine (`model/session.go`)

```
uploaded → introduced → mapping_draft → mapping_confirmed → extracting → extracted
                              ↑______________ revise ______________↑
```

- Mapping hidup di `mapping_draft` → `mapping_confirmed`.
- Gate `confirm` hanya sah dari `mapping_draft`; gate `extract` hanya dari
  `mapping_confirmed` (fail-closed, tidak bisa lompat state).
- `revise` kembali dari `mapping_confirmed`/`extracted` ke `mapping_draft`.
- State dijaga CHECK constraint di DB — menambah state = migrasi baru.
- **WO-07 (2026-11-19)**: sesi yang macet di `extracting` (stale) punya
  jalur pemulihan eksplisit — cancel + delete — agar tidak mengunci
  daftar sesi.

## 2. Model mapping (`model/mapping.go`)

Format editor **v2** (MappingVersion = 2), daftar kolom ekspresif:

```json
{"version":2,"columns":[
  {"source":"Nama *","target":"visit_visitors.full_name","kind":"name","is_encrypt":true,"guess":true},
  {"source":"Catatan bebas","target":"skip","kind":"none","skip":true}
]}
```

- `target` = `tabel.field` plugin tujuan, atau `skip` (diabaikan saat ekstraksi).
- `kind` = tipe PII (name/phone/email/address/ssn/credit_card/generic/none).
- `is_encrypt` = nilai tidak disimpan apa adanya (proteksi tambahan).
- `guess` = tebakan agent, belum dikonfirmasi user.
- Legacy CP-2: objek `{ "Kolom Sumber": "tabel.field" }` — `MergeIntoLegacy`
  menerjemahkan v2 ke bentuk legacy untuk payload agent + ekstraksi.

- **Penyimpanan**:
- jsonb yang SAMA dengan profil chat (C-4: satu profil, bukan salinan).
- **Append-only per versi** — `LatestProfile` membaca versi terakhir;
  riwayat profil tidak hilang.
- Validasi fail-closed: `ErrUnknownTarget` (field tidak ada di schema),
  `ErrDuplicateTarget` (dua kolom ke satu field), `ErrPolicyRelaxed`
  (melonggarkan policy `fields.yaml` → ditolak).

**Keputusan multi-segmen (2026-11-19) — TIDAK ada profil v3:** hasil
analisa `analisa-mapping-multi-segmen.md` di topic induk memilih **opsi
B+** = satu sesi per segmen (`target_plugin` per sesi sudah ada), dipilih
eksplisit oleh user, konteks segmen selalu diberikan ke AI. Model mapping
v2, state machine, dan pipeline ekstraksi **tidak berubah**; yang
berkembang adalah jumlah segmen (lihat §6) dan UX-nya (WO-06).

## 3. Dua jalur menuju profil yang sama

### a. Editor manual (`IngestMappingPage.tsx`)

- `GET /admin/ingest/sessions/:id/mapping` → `MappingView`:
  - header + **≤5 baris sample yang sudah dideidentifikasi di Go**
    (`sampleTable`, `services/mapping.go:174`) — tidak ada jalur PII mentah
    ke browser.
  - `Targets` (field tujuan plugin), `Rules`, `Exceptions`, `Gates`,
    `ExtractReady`.
  - **Warning guard kecocokan berkas vs segmen** (WO-06 §A.4, fail-open):
    bila header berkas tidak ada yang cocok dengan katalog field segmen,
    `MappingView` membawa warning — editor hanya memperingatkan, TIDAK
    menolak upload; keputusan tetap di manusia.
- `PUT /sessions/:id/mapping` → `SaveMapping` (simpan draft append-only).
- `POST /sessions/:id/mapping/confirm` → `ConfirmMappingEditor`
  (transisi state).
- Katalog field: `GET /admin/ingest/mapping/options`
  (`MappingOptions(plugin)`).
- **Picker segmen (WO-06)**: sesi lahir dengan `target_plugin` yang
  dipilih user secara eksplisit — tanpa default diam-diam; segmen
  terkunci setelah sesi punya berkas/profil (ganti → 409, buat sesi
  baru); badge segmen tampil di daftar sesi, editor, dan chat.

### b. Via chat (agent strict) — tool di Node, EKSEKUSI di Go

Tiga tool, satu-satunya kemampuan agent pada sesi strict
(`handlers/agent_tools.go`, guard `ingest.write` + audit
`provider='ingest-tool'`):

1. **`get_mapping_guess`** → Go kirim header + ≤5 sample baris aman
   (sudah dideidentifikasi) + daftar field tujuan. Agent menebak mapping;
   `guess: true`.
2. **`confirm_mapping`** → simpan profil (append-only) lalu transisi ke
   `mapping_confirmed`.
3. **`run_extraction`** → wajib kebijakan dedup eksplisit sesuai segmen,
   impor lewat service layer plugin tujuan (via dispatcher §5).

Klien tidak mendefinisikan tool — RBAC + audit tetap di backend (CP-2).
Konteks segmen juga selalu masuk payload agent (WO-06 §C): katalog field
yang dikirim hanya milik segmen sesi; bila berkas tampak campur, agent
menyarankan membuat sesi terpisah per segmen (tidak mengarang mapping
lintas modul).

## 4. Gate sebelum ekstraksi (`evaluateGates`, services/mapping.go:301)

- Kolom wajib terpetakan.
- **Kebijakan dedup — kini PER SEGMEN** (generalisasi pasca WO-08/09):
  - `visit`: `phone_then_time` / `time_only` / `none` — tanpa keputusan →
    ekstraksi ditolak (real case BBH: 177 baris tanpa `phone_e164`,
    impor akan menebak → lebih baik fail-closed).
  - `resident`: kebijakan dedup sendiri (D-R1, keputusan owner) —
    implementasi mengikuti pola `dedupKey`/`findVisitByDedupKey` visit.
  - `sec`/`hsh`: **TANPA dedup policy** — semantik importnya upsert per
    `(period, item)` (`upsertEntryTx`), baris dobel = `updated`, bukan
    data dobel; D-R1 tidak relevan.
- Policy PII tidak boleh dilonggarkan dari `fields.yaml` (D-5); untuk
  `resident`, kolom PII wajib (NIK, data anak) tidak bisa dilonggarkan
  (`ErrPolicyRelaxed`); `birth_date` sengaja TANPA encrypt (D-R4).
- Rules/exceptions O-20 kini per plugin: `exceptions.yaml` ada di 4
  plugin (`visit`/`resident`/`sec`/`hsh`); penegakan deterministik Go
  di titik `Enforce` masing-masing segment target.
- `ExtractReady` dihitung dari gates; juga ada `scanViolations` untuk
  enforcement report.

## 5. Ekstraksi (`services/extraction.go` + `services/target.go`)

- **TIDAK LAGI hardcode `visit`** — pasca WO-08, ekstraksi dispatch per
  plugin lewat **dispatcher generik** `plugins/ingest/services/target.go`
  (`targetFor(plugin)` dengan DTO netral `model/target.go`):
  `SchemaLookup` (validasi target/katalog), `ExceptionLookup` (rules),
  `Enforce` (penegak rules), dan `Importer` (adapter ke
  `Service.ImportTemplate(...)` plugin tujuan).
- Berkas **ASLI** dibaca dari storage di Go (bukan sample) → service layer
  plugin tujuan: validasi → staging 2-fasa (`*_import_batches`/`*_import_rows`)
  → commit; report per baris (`created/updated/skipped/rejected`).
- Node/klien tidak pernah menyentuh jalur ini.
- Jalur terpisah dari deidentify: safe-payload hanya untuk sample/chat.
- Unknown target → fail-closed 409.

## 6. Segmen yang didukung (per 2026-11-19)

`targetFor` (`plugins/ingest/services/target.go`) kini mengenal **4
segmen**; menambah segmen baru = registrasi adapter (pola WO-09), tanpa
perubahan model mapping:

| Segmen | Pipeline import | Dedup | exceptions.yaml | Catatan |
|---|---|---|---|---|
| `visit` | 2-fasa staging, create-only | phone_then_time/time_only/none | ✔ (paling lengkap, 4 rule) | Sesi pilot; 1.599 visits ter-import |
| `resident` | 2-fasa staging, porting pipeline visit (WO-08) | D-R1 (keputusan owner) | ✔ | PII wajib tak bisa dilonggarkan; `birth_date` tanpa encrypt (D-R4) |
| `sec` | SUDAH ADA (upsert per period+item) | tanpa policy (upsert) | ✔ minimal | WO-09; barang tahan lama |
| `hsh` | SUDAH ADA (upsert per period+item) | tanpa policy (upsert) | ✔ minimal | WO-09; habis pakai/pokok |

Konteks lain yang ikut berkembang di topic induk (tidak mengubah
mekanisme mapping): WO-05 — key provider LLM kini tersimpan di DB
(`ingest_providers`, AES-GCM, sumber `INGEST_LLM_*` jadi fallback env);
WO-10 — housekeeping/retensi (status `pending`, menunggu keputusan
O-14/D-T1) untuk pembersihan sesi & file upload.

## Inti desain

**Go satu-satunya yang melihat isi berkas dan menulis data bisnis.**
Klien dan agent hanya melihat sample terdeidentifikasi dan mengirim
KEPUTUSAN mapping; chat dan editor menulis ke dokumen profil yang sama
(C-4), sehingga hasil tebakan agent selalu bisa ditinjau/diubah di editor.
Dispatcher generik (`target.go`) membuat segmen baru murah: adapter ke
pipeline import yang sudah ada + exceptions minimal — model mapping v2
dan state machine tidak pernah berubah.

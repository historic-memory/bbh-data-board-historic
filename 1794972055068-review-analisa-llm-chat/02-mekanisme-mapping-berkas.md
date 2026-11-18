---
title: "Mekanisme mapping berkas (ingest strict)"
description: "Analisa alur mapping kolom berkas: state machine, profil v2, dua jalur (editor & chat), gate, dan ekstraksi"
status: draft
created: "2026-11-18"
updated: "2026-11-18"
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
- Temuan (2026-11-18).

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

**Penyimpanan**:
- jsonb yang SAMA dengan profil chat (C-4: satu profil, bukan salinan).
- **Append-only per versi** — `LatestProfile` membaca versi terakhir;
  riwayat profil tidak hilang.
- Validasi fail-closed: `ErrUnknownTarget` (field tidak ada di schema),
  `ErrDuplicateTarget` (dua kolom ke satu field), `ErrPolicyRelaxed`
  (melonggarkan policy `fields.yaml` → ditolak).

## 3. Dua jalur menuju profil yang sama

### a. Editor manual (`IngestMappingPage.tsx`)

- `GET /admin/ingest/sessions/:id/mapping` → `MappingView`:
  - header + **≤5 baris sample yang sudah dideidentifikasi di Go**
    (`sampleTable`, `services/mapping.go:174`) — tidak ada jalur PII mentah
    ke browser.
  - `Targets` (field tujuan plugin), `Rules`, `Exceptions`, `Gates`,
    `ExtractReady`.
- `PUT /sessions/:id/mapping` → `SaveMapping` (simpan draft append-only).
- `POST /sessions/:id/mapping/confirm` → `ConfirmMappingEditor`
  (transisi state).
- Katalog field: `GET /admin/ingest/mapping/options`
  (`MappingOptions(plugin)`).

### b. Via chat (agent strict) — tool di Node, EKSEKUSI di Go

Tiga tool, satu-satunya kemampuan agent pada sesi strict
(`handlers/agent_tools.go`, guard `ingest.write` + audit
`provider='ingest-tool'`):

1. **`get_mapping_guess`** → Go kirim header + ≤5 sample baris aman
   (sudah dideidentifikasi) + daftar field tujuan. Agent menebak mapping;
   `guess: true`.
2. **`confirm_mapping`** → simpan profil (append-only) lalu transisi ke
   `mapping_confirmed`.
3. **`run_extraction`** → wajib kebijakan dedup eksplisit, impor lewat
   service layer `visit`.

Klien tidak mendefinisikan tool — RBAC + audit tetap di backend (CP-2).

## 4. Gate sebelum ekstraksi (`evaluateGates`, services/mapping.go:301)

- Kolom wajib terpetakan.
- Kebijakan dedup eksplisit: `phone_then_time` / `time_only` / `none`.
  Tanpa keputusan → ekstraksi ditolak (real case BBH: 177 baris tanpa
  `phone_e164`, impor akan menebak → lebih baik fail-closed).
- Policy PII tidak boleh dilonggarkan dari `fields.yaml` (D-5).
- `ExtractReady` dihitung dari gates; juga ada `scanViolations` untuk
  enforcement report.

## 5. Ekstraksi (`services/extraction.go`)

- Berkas **ASLI** dibaca dari storage di Go (bukan sample) → service layer
  plugin `visit`: validasi → staging → import.
- Node/klien tidak pernah menyentuh jalur ini.
- Jalur terpisah dari deidentify: safe-payload hanya untuk sample/chat.

## Inti desain

**Go satu-satunya yang melihat isi berkas dan menulis data bisnis.**
Klien dan agent hanya melihat sample terdeidentifikasi dan mengirim
KEPUTUSAN mapping; chat dan editor menulis ke dokumen profil yang sama
(C-4), sehingga hasil tebakan agent selalu bisa ditinjau/diubah di editor.

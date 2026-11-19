---
title: "Analisa mapping multi-segmen (satu berkas, banyak modul)"
description: "Analisa kebutuhan satu berkas ingest berisi banyak segmen data (keuangan, bayi, kebutuhan): dampak model mapping, validasi, gate, ekstraksi, dan opsi desain"
status: draft
created: "2026-11-19"
updated: "2026-11-19"
tags:
  - ingest
  - mapping
  - arsitektur
---
# Analisa mapping multi-segmen

## Tujuan

Menganalisa kebutuhan: satu berkas ingest bisa berisi **banyak segmen data**
(keuangan, bayi, kebutuhan, dll.), sehingga mapping kolom tidak lagi terkunci
ke satu modul tujuan per sesi.

## Kondisi saat ini (kenapa belum bisa)

- Sesi ingest mengikat **satu** `target_plugin`
  (`plugins/ingest/model/session.go:116`, kolom `target_plugin`):
  satu berkas = satu modul tujuan.
- Profil mapping v2 (`model/mapping.go`) menyimpan `target` sebagai
  `tabel.field` **tanpa prefix plugin** — dengan dua modul, `full_name`
  menjadi ambigu.
- Ekstraksi menulis via service layer **satu** plugin
  (`plugins/visit/services/import.go`), dengan gate & kebijakan dedup
  yang dihitung per satu modul (`evaluateGates`, `services/mapping.go`).
- Katalog opsi mapping satu plugin: `MappingOptions(plugin)`
  (`GET /admin/ingest/mapping/options`).
- Fail-closed yang sudah ada: `ErrUnknownTarget`, `ErrDuplicateTarget`,
  `ErrPolicyRelaxed` (dicek terhadap `fields.yaml` satu modul).

## Dampak perubahan (jika multi-segmen)

1. **Model mapping (v3?)** — profil perlu grup per segmen, mis.
   `segments: [{plugin:"visit", columns:[...]}, {plugin:"finance", columns:[...]}]`,
   atau `target` berubah jadi `plugin.tabel.field`. Append-only per versi
   tetap berlaku (cukup versi profil baru).
2. **Validasi fail-closed** — `ErrUnknownTarget`/`ErrDuplicateTarget`
   harus tahu validasi per-schema plugin; `ErrPolicyRelaxed` dicek terhadap
   `fields.yaml` modul masing-masing segmen.
3. **Ekstraksi multi-importer** — routing kolom ke service layer beberapa
   plugin; perlu transaksi yang jelas antar segmen (lihat keputusan di bawah).
4. **Gate & dedup** — kebijakan dedup (`phone_then_time`/`time_only`/`none`)
   saat ini per modul: satu kebijakan global atau per segmen.
5. **UI editor** (`IngestMappingPage.tsx`) — pengelompokan kolom per segmen
   (bukan satu dropdown rata), katalog opsi lintas plugin.
6. **Agent chat (prinsip C-4 tetap)** — `get_mapping_guess` kirim katalog
   multi-plugin; tebakan agent tetap ditinjau/direvisi di editor;
   `confirm_mapping`/`run_extraction` tetap tool backend Go.

## Keputusan desain

**Pendekatan yang dipilih: satu sesi per segmen, dipilih eksplisit oleh
user, dengan konteks segmen diberikan ke AI** (keputusan 2026-11-19,
menggantikan rancangan multi-segmen dalam satu sesi).

Alasan (dari diskusi 2026-11-19): paling mudah dipahami user — jika user
mau insert data A, user memilih segmentasi A dan mengurusi itu sampai
selesai. Ketika user upload Excel A lalu memilih segmentasi A di map
editor, AI memahami konteksnya. Backend praktis tidak berubah karena
`target_plugin` per sesi sudah ada.

Yang perlu dilengkapi:

1. **Pemilihan segmentasi eksplisit & terlihat** — user memilih segmen
   saat membuat sesi (atau di awal flow); pilihan itu tampil jelas di
   map editor + chat, bukan tersembunyi di database.
2. **Konteks segmen ke AI** — saat user upload Excel A di sesi A, agent
   selalu tahu "ini sesi segmen A" dalam promptnya; AI tidak menebak
   segmen dari isi berkas. Jika berkas tampak campur, AI berpesan
   membuat sesi terpisah per segmen.
3. **Guard kecocokan berkas vs segmen** — bila kolom berkas tidak ada
   yang mirip field segmen sesi, editor memperingatkan sebelum confirm.
4. **Ekstraksi tetap per sesi (partial commit antar sesi)** — gate
   fail-closed tetap ketat per modul (kolom wajib, policy PII, dedup);
   kegagalan satu sesi tidak menjatuhkan sesi segmen lain — laporan
   enforcement mengikuti pola `scanViolations`.

## Opsi pendekatan

- **A. Profil multi-segmen (v3)** — satu sesi, satu berkas, mapping
  dikelompokkan per segmen. Paling bersih; menyentuh model, validasi,
  ekstraksi.
- **B / B+. Satu sesi per segmen (DIPILIH)** — sesi tetap single-target
  (`target_plugin` sudah ada), tapi dipilih eksplisit oleh user dan
  konteksnya selalu diberikan ke AI. B+ menambahkan: picker segmen yang
  terlihat, pesan AI saat berkas tampak campur, guard kecocokan berkas
  vs segmen. Nol perubahan model mapping; hanya UX + prompt agent.
- **C. Hybrid** — sesi multi-target, ekstraksi tetap dijalankan per segmen
  dengan gate per segmen. Dicadangkan untuk masa depan bila kebutuhan
  "satu berkas campur" benar-benar sering terjadi.

## Rekomendasi

Opsi **B+ (satu sesi per segmen, eksplisit, konteks ke AI)** — paling
mudah dipahami user dan backend praktis tidak berubah. Work order
implementasi dibuat setelah analisa ini disetujui.

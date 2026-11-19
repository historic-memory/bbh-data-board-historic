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

**Ekstraksi multi-segmen: per segmen (partial commit) dengan gate
fail-closed per segmen** (disetujui 2026-11-19).

- Gate tetap ketat per modul (kolom wajib, policy PII, dedup) sebelum
  segmen itu boleh diekstraksi.
- Kegagalan satu segmen tidak menjatuhkan segmen lain — laporan
  enforcement per segmen (mengikuti pola `scanViolations`).
- Konsisten dengan fail-closed: yang ditolak memang tidak masuk,
  tanpa membuat satu kolom salah membatalkan impor segmen lain.

## Opsi pendekatan

- **A. Profil multi-segmen (v3)** — satu sesi, satu berkas, mapping
  dikelompokkan per segmen. Paling bersih; menyentuh model, validasi,
  ekstraksi.
- **B. Pre-split** — tetap satu sesi per modul, berkas sama di-upload ke
  beberapa sesi. Nol perubahan backend, UX berat, berkas mentah dibaca
  berulang.
- **C. Hybrid** — sesi multi-target, ekstraksi tetap dijalankan per segmen
  dengan gate per segmen (kompromi A + transaksi per segmen).

## Rekomendasi

Opsi **C (hybrid)**: model profil multi-segmen, eksekusi ekstraksi per
segmen dengan gate per segmen. Work order implementasi dibuat setelah
analisa ini disetujui.

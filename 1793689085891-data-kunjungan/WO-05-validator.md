---
title: WO-05 — Validator umpan balik kunjungan
description: POST /api/admin/visit-template/validate — parse & laporan baris bermasalah tanpa menulis DB.
type: work-order
status: pending
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-05 — Validator umpan balik kunjungan

**Prasyarat:** WO-04.

## Lingkup

1. `services/validate.go` (pola hsh/sec): tolak `.xls` (ekstensi/magic OLE2); parse xlsx/csv; `template_version` mismatch → file-level error.
2. Aturan validasi dari `fields.yaml` blok `validation:`:
   - `Nama` required.
   - `Nomor HP` — regex longgar (karakter `+0-9 ()-`); pola tak dikenal → **warning**, bukan penolakan (data sumber memang kacau).
   - `Jumlah Pengunjung` — bebas; tak terparse → **warning**.
   - `Status Kunjungan` — label `Not started`/`Done` (dan ID: `Belum dimulai`/`Selesai` bila ada) → map ke enum; tak dikenal → error.
   - `Waktu Kunjungan` — parse `DD Month YYYY HH:MM` bulan ID+EN; gagal → error; kosong → warning.
3. Feedback: xlsx (sheet HASIL+RINGKASAN) / csv (+`hasil_validasi`, `alasan_ditolak`); header `X-Validate-*`; `?strict=1` → 422.
4. Route `POST /api/admin/visit-template/validate` (guard `visit.read`), TIDAK menulis DB.
5. Fixture `api-tests/fixtures/visit-broken-template.csv`.

## Acceptance Criteria

- AC-1: Test — baris valid lulus; tanggal bulan `Mei`/`Agustus`/`Desember` terparse; format tak dikenal ditolak; HP aneh hanya warning.
- AC-2: curl — mixed file → total/valid/rejected benar; strict 422; tanpa token 401.

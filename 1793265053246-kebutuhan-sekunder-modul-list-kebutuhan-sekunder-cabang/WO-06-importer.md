---
title: "WO-06: Importer (staging → upsert)"
description: "Endpoint impor template sec: fasa staging + upsert idempoten, alias-first matching, auto-create item/alias, tolak .xls."
status: pending
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - import
  - idempotency
---

# WO-06: Importer (staging → upsert)

## Referensi
- SPEC: S-12, S-18, S-19, §Alur Impor, GAP-2.
- Pola acuan: importer 2 fasa `plugins/hsh` WO-06 (`services/import.go`) / `plugins/resident`.

## Tugas
1. **Migrasi staging** `sec_import_batches` + `sec_import_rows`
   (batch = berkas unggahan; rows = baris mentah + status validasi + pesan + aksi).
2. **Fasa 1 — staging**: baca `.xlsx`/`.csv` (pola WO-05), simpan baris mentah, jalankan validator,
   tulis laporan (dibuat/diperbarui/dilewati/ditolak) **tanpa** menyentuh master/entri.
3. **Fasa 2 — upsert** (transaksi):
   - **Periode**: cocokkan `code`; buat bila belum ada (periode **boleh** auto-create dari kolom,
     kategori **tidak**). Periode hasil impor dibuat **nonaktif** (jangan mencuri "≤1 periode aktif").
   - **Item**: cocokkan urutan **alias** → **name persis (case/space-insensitive)** → **buat item baru**
     (S-19) + catat nama asli sebagai **alias**. Isi `brand`/`specification`/`unit`/`est_price` bila ada.
   - **Entri**: upsert per `(period_id, item_id)`; isi `status_code`, `quantity`, `unit_price`, `note`,
     `recorded_by`.
4. **Idempotensi**: menjalankan impor berkas sama 2x tidak menggandakan baris.
5. **Penolakan `.xls`**: deteksi BIFF8 → 400 + pesan pra-konversi (S-18).
6. **Endpoint**: `POST /admin/sec-template/import` (guard `sec.write`),
   multipart `file` + `?format=`, `?dry_run=1`. Respons: id batch + ringkasan.
   `GET /admin/sec/import-batches/:id` untuk membaca laporan.
7. **Test**: impor valid → count master/entri sesuai; impor ulang → tidak bertambah; item baru → alias
   tercatat; `.xls` → 400; baris invalid → dilaporkan & tidak menggagalkan baris valid.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/sec/...` hijau.
- curl impor → ringkasan; `SELECT count(*)` sebelum & sesudah impor ulang (tidak berubah);
  alias item baru muncul; `.xls` → 400; `?dry_run=1` → tidak menyentuh master.

## Kriteria Selesai
- Impor idempoten; staging menyimpan laporan yang bisa diaudit.
- Auto-create item + alias berjalan; kategori tidak auto-create.
- `.xls` ditolak jelas; tidak ada penulisan DB pada fasa validasi.

## Bukti Pelaksanaan
_(diisi saat WO dikerjakan)_

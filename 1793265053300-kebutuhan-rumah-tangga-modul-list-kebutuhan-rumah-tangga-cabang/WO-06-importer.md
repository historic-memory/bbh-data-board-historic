---
title: "WO-06: Importer (staging → upsert)"
description: "Endpoint impor template needhousehold: fasa staging + upsert idempoten, alias-first matching, auto-create item/alias, tolak .xls."
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
- SPEC: N-12, N-16, N-17, §Alur Impor, GAP-1/GAP-2.
- Pola acuan: importer 2 fasa plugin `resident` (staging tabel dulu, baru cek idempotensi).

## Tugas
1. **Migrasi staging** `needhousehold_import_batches` + `needhousehold_import_rows`
   (batch = berkas unggahan; rows = baris mentah + status validasi + pesan + aksi).
2. **Fasa 1 — staging**: baca `.xlsx`/`.csv` (pola WO-05), simpan baris mentah, jalankan validator,
   tulis laporan (dibuat/diperbarui/dilewati/ditolak) **tanpa** menyentuh master/entri.
3. **Fasa 2 — upsert** (transaksi):
   - **Periode**: cocokkan `code`; buat bila belum ada (kategori/periode bergantung keputusan O-1;
     periode **boleh** auto-create dari kolom, kategori **tidak**).
   - **Item**: cocokkan urutan **alias** → **name persis (case/space-insensitive)** → **buat item baru**
     (N-16) + catat nama asli sebagai **alias**.
   - **Entri**: upsert per `(period_id, item_id)`; isi `status_code`, `quantity`, `unit_price`, `note`,
     `recorded_by`.
4. **Idempotensi**: menjalankan impor berkas sama 2x tidak menggandakan baris.
5. **Penolakan `.xls`**: deteksi BIFF8 → 400 + pesan pra-konversi (N-17).
6. **Endpoint**: `POST /admin/needhousehold-template/import` (guard `needhousehold.write`),
   multipart `file` + `?format=`. Respons: id batch + ringkasan + tautan laporan.
   Bila perlu: `GET /admin/needhousehold/import-batches/:id` untuk membaca laporan.
7. **Test**: impor valid → count master/entri sesuai; impor ulang → tidak bertambah; item baru → alias
   tercatat; `.xls` → 400; baris invalid dari staging → dilaporkan & tidak menggagalkan baris valid.

## Bukti yang harus dilampirkan
- `go build ./...` + `go test ./plugins/needhousehold/...` hijau.
- curl impor template contoh → ringkasan; `SELECT count(*)` master/entri sebelum & sesudah impor ulang
  (tidak berubah); alias item baru muncul; `.xls` → 400 dengan pesan konversi.
- Laporan batch (per status: dibuat/diperbarui/dilewati/ditolak).

## Kriteria Selesai
- Impor idempoten; staging menyimpan laporan yang bisa diaudit.
- Auto-create item + alias berjalan; kategori tidak auto-create.
- `.xls` ditolak jelas; tidak ada penulisan DB pada fasa validasi.

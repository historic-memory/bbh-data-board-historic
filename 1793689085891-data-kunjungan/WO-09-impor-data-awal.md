---
title: WO-09 — Impor data awal 1.622 baris + laporan kualitas
description: Impor satu kali CSV Notion kunjungan (setelah dedup ekspor) via endpoint WO-06; laporan statistik parse & anomali.
type: work-order
status: pending
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-09 — Impor data awal + laporan kualitas

**Prasyarat:** WO-06 selesai & terverifikasi.

## Lingkup

1. Siapkan CSV bersih dari sumber: buang 17 duplikat ekspor persis (atau biarkan importer yang me-skip — bukti idempotensi nyata).
2. Jalankan `dry_run` dulu → review laporan (berapa baris ditolak/warning).
3. Impor sungguhan → verifikasi jumlah: visitors vs visits vs skipped; sampel acak bandingkan dgn CSV.
4. Laporan kualitas (disimpan di topic): jumlah `party_size` terisi vs NULL, jumlah HP ternormalisasi vs NULL, jumlah `scheduled_at` NULL, 10 baris `DONE`.
5. Konfirmasi angka final ke owner sebelum menyatakan selesai.

## Acceptance Criteria

- AC-1: Seluruh baris valid terimpor; idempotensi terbukti (impor 2x → 0 perubahan).
- AC-2: Laporan kualitas tertulis di topic.

---
title: Analisa Data Mentah Kunjungan
description: Hasil analisa file KUNJUNGAN HARI INI (CSV, ekspor Notion) — profil data, kualitas, dan temuan untuk desain skema.
type: notes
status: complete
created: "2026-11-03"
updated: "2026-11-03"
---

# Analisa Data Mentah Kunjungan

## 1. Sumber

- Berkas: `KUNJUNGAN HARI INI 33d90f1a8a5180228f80d17a10f57149_all.csv` (pola nama & kolom konsisten dengan **ekspor Notion** `_all.csv`, sama seperti data bayi di topic `1793093835163`).
- Kolom: `Nama, Alamat, Jumlah Pengunjung, Nomor HP, Status Kunjungan, Waktu Kunjungan`.
- Baris: **1.639** (1.638 data + header).

## 2. Profil Kolom

### 2.1 `Nama`

- 0 baris kosong.
- **85 nama muncul >1x.** Dua jenis duplikasi:
  - **Duplikat persis** (nama+alamat+HP+waktu identik): **17 baris** → data unik efektif **1.622 baris**. Contoh: `Miki Phoon` muncul 3x identik.
  - **Nama sama, kunjungan berbeda** (waktu/HP beda) — kemungkinan besar **repeat visitor** (kunjungan ulang) atau nama umum (Ayu, Putri, Ana). TIDAK bisa dibedakan hanya dari nama.

### 2.2 `Alamat`

- 7 baris kosong.
- Isi campur: negara (`Singapore`, `Germany`), kota (`Denpasar`, `Ubud`), alamat terstruktur Indonesia (`Br. Pikah, Desa Blahkiuh, Kec. Abiansemal, Kab. Badung`), bahkan **teks ucapkan terima kasih** (`Thank you. Thank you for helping us...`) — bukan alamat.

### 2.3 `Jumlah Pengunjung` — paling kacau

| Pola | Jumlah | Contoh |
|---|---|---|
| Angka bersih | 1.208 | `2`, `4`, `30` |
| Rentang `a-b` | 94 | `2-4`, `15 - 20 pax` |
| Teks campur (bisa ditebak) | ±141 | `2 orang`, `Just myself`, `4 dewasa 2 anak` |
| Tidak terparse / ambigu | ±196 | `Tanggal 7`, `maybe 7 adult and 1 baby`, `belum tahu`, `ya`, `dua`, `-+ 41 people` |
| Kosong | 4 | — |

Ada informasi tambahan yang bernilai di teks: **dewasa vs anak** (`3 adult, 1 children`), **balita**. Satu baris menyebut jumlah akan diinfokan via WA admin.

### 2.4 `Nomor HP`

| Pola | Jumlah | Catatan |
|---|---|---|
| Lokal `08xx` | 1.165 | normal Indonesia |
| Internasional `+<kode>` | 317 | SG/NL/AU/DE/FR/dll |
| Tanpa nol depan (mis. `82146904491`) | 50 | kemungkinan kehilangan `0` |
| Format lain (spasi, `06-40961768`, dll.) | 101 | perlu normalisasi |
| Ada akhiran `(WA)` | 1 | indikator WhatsApp |
| Kosong | 5 | — |

### 2.5 `Status Kunjungan`

- `Not started`: **1.629**
- `Done`: **10** — seluruhnya tanggal `23 April 2026` (kemungkinan hari pertama penggunaan form; setelah itu status tidak lagi di-update di sumber).

### 2.6 `Waktu Kunjungan`

- **1.625 baris** berformat `DD Month YYYY HH:MM (GMT+8)`; bulan **campur Indonesia & Inggris** (`Mei`, `Agustus`, `Desember` vs `April` — `April` sama di kedua bahasa).
- **14 baris kosong.**
- `GMT+8` = WITA (Bali) → sumber sudah konsisten satu zona.
- Rentang tanggal: **April 2026 – Desember 2026** → ini **booking/jadwal kunjungan**, bukan log kunjungan terjadi.
- Jam umumnya `9:00–17:00`, banyak tepat `9:00 / 15:00` (kemungkinan nilai pilihan default form).

## 3. Kesimpulan Utama

1. Ini data **antrean/booking kunjungan** (permintaan kunjungan), bukan kehadiran fisik. `Status Kunjungan` = status pemrosesan booking.
2. Kualitas `Jumlah Pengunjung` terlalu berantakan untuk dikonversi penuh → butuh **kolom angka kanonik + kolom teks mentah**.
3. Nomor HP butuh **normalisasi ke E.164** + simpan teks asli; HP bisa jadi kunci deduplikasi pengunjung.
4. Duplikasi ada dua macat: duplikat ekspor persis (dibuang via idempotensi impor) dan repeat visitor (perlu keputusan struktur data).
5. Sumber hanya punya 2 nilai status; sistem perlu enum yang lebih lengkap untuk alur kerja admin (lihat pertanyaan terbuka di `spec.md`).

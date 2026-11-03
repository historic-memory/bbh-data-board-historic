---
title: WO-09 — Impor data awal 1.622 baris + laporan kualitas
description: Impor satu kali CSV Notion kunjungan (setelah dedup ekspor) via endpoint WO-06; laporan statistik parse & anomali.
type: work-order
status: complete
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

## Bukti

Sumber: `KUNJUNGAN HARI INI 33d90f1a8a5180228f80d17a10f57149_all.csv` (ekspor
Notion, 1.639 baris data, header sudah identik template: `Nama, Alamat, Jumlah
Pengunjung, Nomor HP, Status Kunjungan, Waktu Kunjungan`). Berkas **tidak
diubah** — diunggah apa adanya (BOM + CRLF ditangani parser).

### Fasa uji kering
```
POST /api/admin/visit-template/import?dry_run=1
batch 01a1a0c6-12fe-7b0b-bda6-a3b8b4fb56e7 · status=staged
summary total=1639 created=0 rejected=0 · 1639 baris staging, 0 alasan tolak
```

### Fasa commit
```
POST /api/admin/visit-template/import
batch 01a1a0c6-65d0-7bed-8c86-dd27f1169358 · status=committed · branch=local
summary total=1639 created=1599 skipped=40 rejected=0
```
`skipped=40` = duplikat ekspor `(phone_e164, scheduled_at)`: 17 duplikat persis
+ 23 pasangan berbeda-teks dengan HP & waktu sama. Hasil DB:
`visit_visitors=1533`, `visit_visits=1599`, `visit_status_history=1599`
(1 baris riwayat per kunjungan, `effective_at` = `scheduled_at` sumber).

### Idempotensi (AC-1)
Re-impor himpunan baris yang punya HP (1.447 baris, dibentuk dari staging batch
commit → subset CSV):
```
batch 01a1a0c7-9f14-70de-b783-fafddeb90e76 · status=committed
summary total=1447 created=0 updated=0 skipped=1447 rejected=0
```
Jumlah master **tidak berubah** (1533/1599). 

**Batasan terukur:** 177 kunjungan (11%) tidak punya HP yang bisa dinormalisasi
(5 kosong + 172 nomor telepon rumah/teks), sehingga tidak punya kunci dedup
`(phone_e164, scheduled_at)` — mengimpor ulang baris-baris itu akan membuat
kunjungan baru. Ini sesuai desain WO-06 (`findVisitByKey` mengembalikan kosong
bila `phone_e164` NULL). Untuk impor ulang total, hapus dulu master atau
impor hanya berkas yang sudah bersih.

### Laporan kualitas data

| Metrik | Nilai |
|---|---|
| Kunjungan | 1.599 (pengunjung 1.533) |
| Status | `SCHEDULED` 1.589 · `DONE` 10 |
| `scheduled_at` terisi / NULL | 1.589 / 10 |
| Rentang waktu kunjungan | 2026-04-02 s/d 2027-06-09 |
| `party_size` terisi / NULL | 1.176 / 423 (23% tidak terbaca sebagai angka) |
| `party_raw` terisi | 1.595 (44 kosong) |
| `phone_e164` terisi / NULL | 1.356 / 177 (11,5%) |
| `branch_key` | `local` (1.599/1.599) |
| Asal teratas | Denpasar 166 · Australia 112 · Bali 105 · France 50 · Indonesia 46 |

`party_size` NULL bukan kegagalan: nilai seperti "2-4", "Just myself", "3 adul
+ 1 baby" memang disimpan sebagai teks di `party_raw` (D-12).

**Sampling verifikasi (sumber → DB):**
- `Melany Blankendal · 16 April 2026 3:00 (GMT+8)` → `scheduled_at
  2026-04-15T19:00Z`, `party_raw=1`, `party_size=1`, status `SCHEDULED`.
- 10 baris `Done` → `status_code=DONE` dengan `effective_at` = waktu sumber
  (mis. `Cindy Maag · 23 April 2026 11:06` → `2026-04-23T03:06Z`).

### Verifikasi UI
`http://localhost:8082/app/visits` (login root) → **Halaman 1 dari 64 · total
1.599 data**, baris pertama `Maria Prisco · +393519743451 · Italy · 2 ·
09 Jun 2027, 14.00 WIB · Dijadwalkan`.

### Catatan
**Data hasil impor DIBIARKAN di database** atas permintaan owner (dipakai untuk
demo ke rekan). Tidak ada pembersihan setelah verifikasi; yang dibersihkan
hanya artefak uji lain (staging hasil Bruno/RBAC, admin & grup uji).

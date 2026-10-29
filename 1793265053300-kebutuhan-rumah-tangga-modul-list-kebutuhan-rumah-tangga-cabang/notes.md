---
title: notes
description: "Analisis data mentah: file 'List Kebutuhan RT BBH SEPT 19 OKT 26 d.xls' + gap yang harus dijawab sebelum SPEC."
status: progress
created: "2026-10-29"
---
# Notes — Kebutuhan Rumah Tangga

## Konteks
- Modul **baru**: pencatatan **list kebutuhan rumah tangga** (perlengkapan dapur, alat kebersihan, obat/vitamin, linen, dsb.) untuk BBH.
- Status: **data mentah sudah diunggah** (2026-10-29). Analisis ada di bawah; **SPEC belum dibuat**.
- Catatan: nama topic menyebut "cabang", tetapi berkas data **belum menunjukkan dimensi cabang** — lihat pertanyaan bloker.

## Yang sudah diputuskan
- Berdiri sebagai **topic terpisah** dari "Kebutuhan Elektronik" (keputusan owner).
- Akan dibangun sebagai **plugin baru** di `plugins/<id>/` (aturan user: fitur baru = plugin, tidak menaruh kode fitur di `internal/`).

## Usulan (belum dikunci)
- **Usulan plugin id:** `plugins/needs` (satu plugin untuk kedua list, dibedakan lewat `category: electronics | household`).
  Alternatif: plugin terpisah bila struktur kedua list ternyata berbeda jauh.
- Gap yang perlu dijawab saat SPEC: sumber data (CSV/Notion?), apakah per-cabang (pakai `endpoints.key`/`INSTANCE_KEY`, bukan tabel `branches`), perlu quantity/harga/status pemenuhan, RBAC permission key, dukungan impor/template.

## Analisis data mentah (2026-10-29)

**Aset**: `List Kebutuhan RT  BBH  SEPT 19 OKT 26 d.xls` (35.840 byte, tercatat di `_meta.yaml.assets`).

### Format berkas
- **Bukan XLSX** — ini **XLS legacy (OLE2/BIFF8)**, dibuat MS Excel (author `win`, last saved **19 Okt 2026 05:26**, last printed 24 Sep 2026 20:23).
- **1 sheet** (`Sheet1`), 10 merge cell, dimensi `B1:L54`. Data aktual hanya kolom **B–E**; F–L kosong (hanya berisi fill putih).
- Agar bisa dibaca/diimpor: dikonversi dulu pakai LibreOffice (`soffice --headless --convert-to xlsx`) → terbaca sebagai xlsx. **`excelize` (dipakai plugin `resident`) TIDAK bisa membaca `.xls` BIFF** → perlu konversi manual atau dukungan eksplisit di kode impor.

### Arti warna (legend DI DALAM berkas, baris 25–27 dan 51–53)
| Fill | Arti |
|---|---|
| `FFFF99CC` (pink) | **Urgent / No stock — Out of Stock** |
| `FF003366` (navy) | **Enough Stock / Stock sudah cukup** |
| `FFFFFF00` (kuning) | header kategori |
| `FFC0C0C0` (abu) | judul board |
| putih / tanpa fill | **tidak ditandai** (status kosong/unknown) |

### Struktur
- **Judul board**: `Kebutuhan Bali Baby Home/Bali Baby Home Needs` — muncul 2x (baris 2 dan 33) → berkas berisi **2 blok** (dugaan: 2 periode snapshot, sesuai nama file "SEPT" & "19 OKT 26").
- Tata letak = **kolom-kolom daftar** (bukan tabel baris-per-item). Header kategori di baris 3 dan 34, sebagian ber-merge (`B3:C3`, `B34:C34`); satu header nyempil di tengah blok (`D17:E17` = Stationery, baris 18–21).
- **Tidak ada kolom**: quantity/jumlah, satuan, harga, tanggal, cabang, PIC, keterangan.

### Kategori & jumlah item (total ± 118 item)
| Kategori (di berkas) | Kolom | Item | urgen | cukup | kosong |
|---|---|---|---|---|---|
| Baby Needs / Kebutuhan Bayi & Diaper | B,C (r4–23) | 34 | 11 | 2 | 21 |
| Milk / Susu | D (r4–16) | 10 | 6 | 0 | 4 |
| Food / Makanan | E (r4–16) | 13 | 12 | 0 | 1 |
| Stationery & Educational Tools / ATK & Alat Edukasi | D (r18–21) | 4 | 1 | 0 | 3 |
| Medicine & Vitamin / Obat dan vitamin | B,C (r35–52) | 31 | 30 | 0 | 1 |
| **Housekeeping & Equipment / Rumah Tangga & Peralatan** | D (r35–53) | 18 | 14 | 1 | 3 |
| Linen | E (r35–43) | 8 | 6 | 0 | 2 |

### Temuan kualitas data
- Nama item **bilingual** dengan format tak konsisten:
  - separator `/` normal: `hand sanitizer/pensanitasi tangan`
  - separator hilang: `Zaitun oil for Baby's bodyMinyak zaitun untuk badan bayi`
  - urutan terbalik (ID dulu): `Telon oil/Minyak telon`, `Wet wipes/Tissue basah`
  - spasi di ujung: `Minyak telon `, `Handuk anak `
  - nama panjang + tanda kurung/kutip: `Meja kursi ceria/Kursi makan anak - pendek (Table chair "ceria"/ Chair table short)`
  - ada nilai aneh: `Buku Gambar`, `Sepidol Warna` (kemungkinan typo dari "Spidol Warna"), `Minyak kayu putih` (tanpa padanan EN)
- Typo pada **nama kategori**: `Housekeeping & Equpment` (→ Equipment), `Room Freshner Spray` (→ Freshener).
- **Item nyasar** antar kategori (mis. `Hanging Air Freshener ...` ada di bawah blok Stationery; `Meja kursi ceria` (perabot) ada di kolom Food).
- **Kelas obat bermerek** bercampur generik: `Amoxsan/yusimox Susp` vs `Amoxcillin Tab`, `PARATUSIN TAB` vs `Paratusin Tablet` (duplikat makna, beda penulisan) → perlu normalisasi/dedup.
- **Nama obat** ada yang sama antar kolom (mis. `PARATUSIN TAB` di C42 dan `Paratusin Tablet` di B37).

## Pertanyaan terbuka (BLOKER SPEC)
- [ ] **Per-cabang atau BBH-wide?** Judul board hanya "Bali Baby Home" — tidak ada dimensi cabang. Topic ini dinamai "...cabang"; perlu diputuskan: satu list global atau list per cabang (`endpoints.key`) dengan salinan item?
- [ ] **2 blok = 2 periode?** Jika ya: mana yang berlaku (September vs 19 Okt 26), dan apakah perlu riwayat periode (snapshot) atau cukup kondisi terkini?
- [ ] **Kolom quantity/satuan/harga**: tidak ada di berkas. Apakah modul cukup butuh nama+kategori+status, atau perlu jumlah diminta?
- [ ] **Kategori akhir**: 7 kategori di berkas — apakah dipakai apa adanya (termasuk `Linen`, `Medicine & Vitamin`), atau hanya sebagian (mis. `Housekeeping & Equipment` saja yang relevan untuk "rumah tangga")?
- [ ] **Sumber data berikutnya**: apakah owner akan memberi `.xlsx`/CSV (lebih ramah impor) atau tetap `.xls`?
- [ ] **Apakah item dibawa dari CSV Notion** ("Data Bayi") / sistem lain, atau murni manual seperti berkas ini?
- [ ] Integrasi dengan modul `resident` (baby needs/medicine tampak relevan) atau berdiri sendiri?

## Langkah berikutnya
1. Owner menjawab pertanyaan bloker di atas + menyiapkan data mentah **Kebutuhan Elektronik** (belum diunggah).
2. Bila `.xls` tetap dipakai: sepakati jalur konversi (LibreOffice di host vs minta `.xlsx`).
3. Manager susun `spec.md` (keputusan desain + gap) lalu `work-order.md`.
4. Konversi pembacaan bersifat sementara (di `/tmp/rt-xls`), tidak menambah berkas ke repo.


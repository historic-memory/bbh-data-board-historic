---
title: "Analisa Data Mentah Business"
description: "Analisa data core bayi BBH sebagai dasar ERD/skema tabel canonical"
status: progress
created: "2026-10-27"
---
# Analisa Data Mentah Business

## Tujuan Topic

- Menganalisa data mentah core (data bayi yayasan BBH) sebagai dasar penyusunan **skema ERD tabel canonical**.
- Integrasi LLM / AI-ETL dipindahkan ke topic terpisah: `.historic/1793096602507-integrasi-llm-ai-etl-copilotkit-jobmanager-sandbox-acp-agent/` (DIPINDAHKAN 2026-10-27).

## Konteks

- **BBH = Bali Baby Home**, yayasan sosial dengan **5 cabang**.
- Data ini = data **bayi terlantar** (salah satu data CORE). Data lain menyusul: keuangan, kunjungan, kebutuhan.
- Tantangan: tiap cabang merekam dengan cara/format berbeda -> tujuan sistem = **"titik tengah"** agar semua data punya **standar SOP**.
- Arsitektur existing mendukung: multi-DB per cabang + federasi + plugin.
- Semua WO sebelumnya (01-08) `complete`; kode HEAD `df11cea`.

## Data Mentah: `Data Bayi Yayasan ...all.csv`

- Ekspor Notion (ID Notion di nama file; kolom RM berisi URL `app.notion.com`).
- Ukuran 16.572 bytes; header + **50 baris**; **23 kolom**; ada **BOM** di header.
- Nampaknya representasi satu gaya rekaman (belum diketahui cabang mana).

### 23 Kolom
1. Nama Bayi | 2. Agama | 3. Anak Ke | 4. Berat Badan Lahir (gram) | 5. Ditolong Oleh | 6. Files & media | 7. Folder Drive | 8. Golongan Darah | 9. Jenis Kelamin | 10. Keterangan | 11. NIK Bayi | 12. NIK Ibu | 13. NOMOR RM BAYI (INTERNAL) | 14. Nama Ibu | 15. Nomor BPJS | 16. Panjang Badan (cm) | 17. Persalinan | 18. Place | 19. Status di yayasan | 20. Tanggal Lahir | 21. Tanggal Masuk Yayasan | 22. Tempat Lahir | 23. Umur

### Distribusi Nilai
- Jenis Kelamin: Perempuan 27 / Laki-laki 23
- Agama: Islam 29, Hindu 16, Kristen 3, Katolik 1, kosong 1
- Status di yayasan: Dikembalikan ke orang tua 24, Dirawat 17, Transfer ke JCH 9
- Ditolong Oleh: Dokter 28, Bidan 17, Sendiri 2, Dukun Anak 1, kosong 2
- Persalinan: Normal 34, Caesar 12, Vakum 2, Sendiri 1, kosong 1
- Golongan Darah: kosong 25, B+ 9, O+ 6, A+ 6, AB+ 2, O 2
- Tempat Lahir: 31 unik (Denpasar 13, Gianyar 6, Badung 2, Buleleng 2, dst)

### Temuan Kualitas Data
- `Place`: **50/50 kosong** -> duplikat konsep dgn `Tempat Lahir` -> kandidat dibuang.
- `Files & media`: 49/50 kosong.
- `Keterangan`: 35/50 kosong.
- `Golongan Darah`: 25/50 kosong.
- `NIK Bayi`: 27/50 kosong (yang terisi semua valid 16 digit).
- `Nomor BPJS`: 36/50 kosong.
- `No. RM`: 19/50 kosong.
- `Umur`: derived (`"4 tahun 11 bulan"`) -> tidak disimpan, dihitung dari Tanggal Lahir.
- Format tidak seragam: Tanggal Lahir `19 November 2021`, Tanggal Masuk `28/02/2022`; Berat `"2600 gram"`; Panjang `"-"` (null).

### Integritas (bagus)
- No. RM format `BBH-26-<P|L>-<nnn>`: 31 terisi, semua unik. P: 001-019 (gap 010), L: 001-013 (tanpa gap).
- Kode P/L konsisten dengan Jenis Kelamin (no mismatch).
- NIK Ibu: 48 terisi, 47 valid 16-digit (1 perlu dicek).
- Tidak ada duplikat nama bayi.

## Kandidat Skema (draft awal, belum final)
- `residents` (bayi): identitas, jenis kelamin, agama, gol darah, tgl lahir, tempat lahir
- `birth_records` (kondisi lahir): berat, panjang, persalinan, ditolong oleh, anak ke
- `mothers` / `guardians`: nama ibu, NIK ibu, BPJS (relasi ke resident)
- `intake/status_log`: status di yayasan, tanggal masuk, keterangan
- `documents`: link Drive / media
- `branches`: 5 cabang (untuk multi-DB / federasi)
- `resident_codes` / No. RM: format baku unik

## Pertanyaan Terbuka
1. CSV ini cabang mana? 5 cabang di mana saja?
2. "Titik tengah" = super-set semua field, atau irisan minimal yang bisa disediakan semua cabang?
3. "JCH" itu apa? beda "Transfer ke JCH" vs "Dirawat di yayasan"?
4. No. RM = identitas nasional sistem? Prefix cabang ikut?
5. Aset dokumen: tetap Google Drive atau pindah ke storage internal?
6. Sistem = SSOT, atau mirror dari sumber lama (Notion)?
7. Ada data mentah cabang lain untuk membuktikan variasi?

## Rencana Lanjutan
- [x] Lampirkan & analisa data mentah core (CSV bayi, 50 baris).
- [ ] Wawancara jawaban pertanyaan terbuka (khususnya cabang & makna JCH).
- [x] Susun **ERD/skema tabel canonical** -> lihat `SPEC.md` (revision 1, status draft).
- [ ] Bekukan enum/controlled vocabulary (status, agama, persalinan, ditolong oleh).
- [ ] (Terpisah) alur AI-ETL di topic LLM.

## Referensi
- **`SPEC.md`** di topic ini = desain ERD + definisi tabel (belum implementasi).
  - 10 tabel + 5 tabel referensi, 10 keputusan terbuka (D-1..D-10), 7 pertanyaan data.
  - Lihat spesifik baris pertama: Tujuan, Prinsip (P-1..P-10), ERD (mermaid), Definisi Tabel, Aturan resident_code, Pemetaan kolom sumber, View, Constraint, PII, Keputusan Terbuka, AC desain.


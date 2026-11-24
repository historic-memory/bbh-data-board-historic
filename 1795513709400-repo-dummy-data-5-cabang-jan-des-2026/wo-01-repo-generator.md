---
title: wo 01 repo generator
description: Repo baru bbh-dummy-data + generator ber-parameter 5 cabang + 300 xlsx + git
status: complete
created: "2026-11-24"
updated: "2026-11-24"
---
# wo 01 repo generator

## Ringkasan
Buat repo baru `~/workspaces/bbh-dummy-data/` (sejajar repo lain) berisi
generator ber-parameter profil cabang + hasil generate 300 berkas .xlsx
(5 cabang × 12 bulan × 5 segmen, Jan–Des 2026), lengkap dengan git.

## Keputusan user (24 Nov 2026 — TERKUNCI)
1. Baris per segmen/bulan/cabang: **30** (sama seperti dummy sebelumnya).
2. Unik NIK/HP: **unik per cabang** (BUKAN global; tanpa awalan lintas cabang).
3. Key cabang: `bali`, `jogja`, `semarang`, `lombok`, `java-children`.
4. Cabang (nama tampilan): Bali Baby Home, Jogja Baby Home, Semarang Baby
   Home, Lombok Baby Home, Java Children Home.
5. Git: repo sekaligus disiapkan ("git juga sekalian") — remote GitHub
   `AkasaTechDev/bbh-dummy-data` (gh login `rolldone` tersedia).

## Tugas
1. [x] Inisiasi repo `~/workspaces/bbh-dummy-data/`: git init, remote
       `https://github.com/AkasaTechDev/bbh-dummy-data.git` (buat via `gh`
       bila belum ada), struktur: `generate.py`, `branches/*.yaml`,
       `data/<cabang>/2026-01-januari…2026-12-desember/`, `README.md`.
2. [x] Generator `generate.py` ber-parameter: baca profil cabang dari
       `branches/*.yaml` (key, nama tampilan, seed), menyalin template asli
       dari server (`GET /api/admin/<segmen>-template?format=xlsx`, server
       :8082) — header identik 1:1, sheet `DATA` saja tanpa PETUNJUK/
       REFERENSI/dropdown. Bisa generate 1 cabang (`--branch bali`) atau
       semua; bisa pilih bulan (`--months 1-12`).
3. [x] Profil cabang: 5 YAML. Gudang dummy per cabang (3 buah, fail-closed
       wajib terdaftar di instance masing-masing — catat di README):
       contoh `Gudang Utama <Cabang> Dummy` dst. Nama fasilitas per cabang
       bebas (NULL-safe).
4. [x] Generate: 5 cabang × 12 bulan × 5 segmen = 300 berkas, 30 baris
       per segmen per bulan. Data fiktif mulai Januari 2026, tanggal
       selalu jatuh di bulan foldernya. Deterministik (seed per cabang).
5. [x] Integritas (verifikasi): NIK bayi/NIK ibu/HP ibu unik **dalam cabang**
       (360/cabang), nama resident & visit unik dalam cabang, No HP visit
       unik dalam cabang (`phone_e164`), `(periode,item)` hsh/sec unik
       dalam cabang, `(gudang,produk,tanggal,jumlah)` wh unik dalam cabang,
       satuan/enum sesuai fields.yaml.
6. [x] README: cara regenerate (unduh template → generate), tabel endpoint
       impor per segmen, prasyarat per instance (reseed kategori, daftar
       3 gudang cabang), urutan impor Jan→Des.
7. [x] Git: commit rapi (generator + profil, lalu data, lalu README),
       push ke `AkasaTechDev/bbh-dummy-data`.

## Acceptance
- 300 berkas .xlsx, header identik template, sheet DATA saja.
- Validasi read-only via `/validate` endpoint: 30/30 valid × 5 segmen ×
  12 bulan × 5 cabang (dapat dibuktikan bila instance cocok; minimal
  verifikasi struktur + validator lokal).
- Repo ter-push ke GitHub.
- Repo aplikasi `bbh-data-board` TIDAK tersentuh.

## Catatan
- Boleh berbasis `api-tests/fixtures/dummy-excel/generate.py` (repo
  aplikasi) sebagai titik awal — jangan lupa menyalin, bukan memindah.
- DB aplikasi tidak boleh diubah (hanya template download + verifikasi
  read-only).

## Catatan Implementasi (Dev)

Repo tujuan: `~/workspaces/bbh-dummy-data/` → `AkasaTechDev/bbh-dummy-data`
(private, branch `main`). Repo aplikasi `bbh-data-board` hanya dibaca.

**Generator & profil**

1. `generate.py:2-44` — docstring menjelaskan kenapa generator (header berasal
   dari `fields.yaml` tiap plugin; salin manual melenceng diam-diam) dan kenapa
   keluaran data-saja aman (validator hanya membaca sheet `DATA`; tanpa
   PETUNJUK → versi `-1` → cek versi dilewati).
2. `generate.py:78-89` — `MALE_FIRST`/`FEMALE_FIRST` 20 nama; `Eka` sengaja
   hanya di daftar laki-laki (memperbaiki bug duplikat nama pada versi repo
   aplikasi).
3. `generate.py:222-259` — `class Branch` memvalidasi profil: `region_code` &
   prefix HP 4 digit (`_validate`, baris 239), prefix ibu ≠ prefix pengunjung,
   `surnames` tepat 20 tanpa duplikat, `warehouses` tepat 3,
   `places`/`facilities` tidak kosong.
4. `generate.py:262-266` — `name_stream()` menghasilkan 400 kombinasi
   `"<depan> <keluarga> Dummy"` per gender (20 × 20) dari nama keluarga cabang.
5. `generate.py:268-285` — `resident_names()` 360 nama (180 L + 180 P,
   diselang-seling); `visit_names()` memakai offset 180 (paruh pool yang belum
   dipakai resident) sehingga nama resident dan pengunjung tidak bertabrakan.
6. `generate.py:288-342` — `load_branches()` + `_check_cross_branch()`
   menolak profil bertabrakan: `key` duplikat, `region_code`/prefix HP sama,
   atau nama keluarga beririsan antar cabang. Ini yang menjamin isolasi antar
   cabang tanpa awalan buatan seperti `D1-`, jadi data tidak "membocorkan"
   bahwa ia dummy lintas cabang.
7. `generate.py:345-387` — `resident_rows()`: NIK =
   `region_code` + 2 (kecamatan) + DDMMYY + 4 (urut); NIK ibu memakai blok
   `45`, HP ibu memakai `phone_prefix_mother`. `idx` naik monoton per cabang,
   jadi keunikan 360 per cabang terpenuhi secara konstruksi.
8. `generate.py:390-414` dan `417-437` — `hsh_rows()` & `sec_rows()`: 30 item
   sama berulang tiap bulan. Disengaja karena kunci idempotensi importer adalah
   `(periode, item)`; item baru tiap bulan hanya menggelembungkan tabel master.
9. `generate.py:440-458` — `visit_rows()`: nomor HP memakai
   `phone_prefix_visit` + `15000000 + idx`, sehingga unik 360 per cabang dan
   tidak bertabrakan dengan nomor ibu.
10. `generate.py:211-218` dan `461-486` — `WH_WAREHOUSE_SLOT` memetakan
    kategori gudang ke indeks `branch.warehouses`, jadi tiap cabang memakai
    tiga gudangnya sendiri. Kadaluarsa dikosongkan untuk `Linen Dummy` dan
    setiap baris ke-4 (kolom opsional).
11. `generate.py:500-530` — `fill()`: memuat template, memotong header sesuai
    lebar sebenarnya, menghapus semua sheet selain `DATA`, lalu
    `ws.data_validations.dataValidation = []` (kalau dropdown dibiarkan, Excel
    menganggap workbook rusak karena referensinya sudah terhapus), lalu menulis
    baris. `raise SystemExit` bila jumlah sel baris ≠ lebar header.
12. `generate.py:537-543` — `parse_months()` memvalidasi `--months 1-12`;
    `--branch` boleh diulang dan `--out` bisa diarahkan ke folder lain
    (`generate.py:551-557`), dengan penolakan key cabang tak dikenal di
    `generate.py:569-575`.
13. `generate.py:598-601` — seed per `(cabang, bulan, segmen)`:
    `branch.seed * 100003 + month * 7919 + len(segment)`. Deterministik, tetapi
    tiap cabang punya variasi hari/jam sendiri.
14. `generate.py:608-612` — penjaga: bila sebuah builder mengembalikan jumlah
    baris ≠ 30, generator berhenti (`SystemExit`) alih-alih menulis berkas
    cacat.

**Profil cabang**

15. `branches/bali.yaml`, `jogja.yaml`, `semarang.yaml`, `lombok.yaml`,
    `java-children.yaml` — masing-masing berisi `key`, `name`, `seed`,
    `region_code` (Bali 5171, Jogja 3471, Semarang 3374, Lombok 5271,
    Java 3173), `phone_prefix_mother`/`phone_prefix_visit` yang berbeda tiap
    cabang, 20 `surnames` lepas antar cabang, 10 `places`, 5 `facilities`
    (satu kosong → NULL-safe), dan 3 `warehouses`.
16. `branches/bali.yaml:1-33` — blok komentar menjelaskan arti setiap kunci,
    termasuk alasan himpunan nama keluarga harus lepas antar cabang.

**Verifikasi**

17. `verify.py:1-23` — docstring: memeriksa hal yang tidak terlihat dari
    "berkasnya ada 300". `verify.py:385` mengembalikan kode ≠ 0 bila ada
    pemeriksaan gagal, sehingga bisa dipakai di CI.
18. `verify.py:319-320` — header tiap berkas dibandingkan **sama persis**
    (daftar nilai, bukan hanya jumlah) dengan header template acuan.
19. `verify.py:321-322` dan `verify.py:109-112` — memastikan sheet hanya
    `['DATA']` dan `len(ws.data_validations.dataValidation) == 0` (dropdown
    benar-benar bersih, bukan sekadar tidak dipakai).
20. `verify.py:204-206` — akumulator keunikan **per cabang**
    (`acc = {branch: defaultdict(set)}`), bukan global; inilah yang sesuai
    keputusan user "unik per cabang".
21. `verify.py:330-356` — pemeriksaan per segmen: NIK/NIK ibu/HP ibu/nama unik
    360, nama resident vs pengunjung tidak bertabrakan, kunci hsh/sec/wh unik
    360, satuan terhadap `UNITS_HSH`/`UNITS_SEC`/`UNITS_WH` (`verify.py:59-66`,
    disalin dari `units:` tiap `fields.yaml`) dan enum
    status/persalinan/penolong/gender/status kunjungan.
22. `verify.py:115-157` — parser tanggal: `parse_daymonth()` untuk
    `DD/MM/YYYY`, `parse_visit_time()` untuk `DD <Bulan ID> YYYY HH:MM`,
    `parse_iso()` untuk tanggal gudang `YYYY-MM-DD`; semuanya memastikan bulan
    cocok dengan folder.
23. `verify.py:358-383` — bonus isolasi antar cabang: NIK, NIK ibu, HP ibu, HP
    pengunjung, nama resident, nama pengunjung harus **tidak beririsan** antara
    pasangan cabang mana pun.
24. `validate-all.sh:1-20` — header skrip menegaskan `/validate` read-only.
    Login memakai `BBH_EMAIL`/`BBH_PASSWORD`; rekap dihitung dari header
    `X-Validate-Total`/`-Valid`/`-Rejected` yang dikirim handler
    (`plugins/warehouse/handlers/validate.go:73-76`), dijumlahkan di
    `validate-all.sh:44-85`.

**README**

25. `README.md` — struktur repo, tabel 5 cabang (prefix NIK/HP/seed), tabel
    segmen & kolom, cara unduh template lalu regenerate (semua/satu
    cabang/sebagian bulan) beserta rumus determinisme, dua lapis verifikasi,
    tabel endpoint impor per segmen, prasyarat per instance (kategori hsh/sec
    via `clean-data --yes --reseed`, tiga gudang per cabang karena gudang
    fail-closed, fasilitas resident NULL-safe), urutan impor Jan→Des, dan tabel
    jaminan isi data.

**Hasil verifikasi**

- `python3 generate.py --templates /tmp/tpl5` → **300 berkas, 9.000 baris**
  (5 cabang × 12 bulan × 5 segmen × 30 baris), 3,1 MB, 6,2 detik.
- `python3 verify.py --templates /tmp/tpl5` → **68/68 pemeriksaan lulus**.
  Per cabang: NIK bayi 360/360, NIK ibu 360/360, HP ibu 360/360, nama resident
  360/360, HP pengunjung 360/360, nama pengunjung 360/360, nama resident ∩
  pengunjung = 0, kunci hsh 360/360, kunci sec 360/360, kunci wh 360/360,
  satuan & enum 0 nilai aneh.
- Isolasi antar cabang: 6 pemeriksaan (NIK bayi/NIK ibu/HP ibu/HP pengunjung/
  nama resident/nama pengunjung) semuanya tidak beririsan.
- `./validate-all.sh` → **300 berkas, 9.000 baris, 0 baris ditolak, 0 gagal
  panggil**; setiap berkas melaporkan `total=30 valid=30 ditolak=0`.
- Determinisme: regenerate penuh lalu bandingkan nilai sel seluruh 300 berkas →
  **300 dibandingkan, 0 berbeda isinya**. Perbedaan byte pada `.xlsx` hanya
  timestamp arsip zip dari openpyxl, bukan isi.
- DB aplikasi tidak berubah setelah 300 panggilan `/validate`:
  `residents=0 visit_visits=0 hsh_entries=0 wh_batches=0 admins=20`.
- `git status` repo `bbh-data-board`: bersih (tidak ada berkas tersentuh).

**Commit & push (repo `bbh-dummy-data`)**

- `0935e4c` feat(generator): generator ber-parameter 5 cabang + verifikasi
- `49c7b84` data: 300 berkas xlsx (5 cabang × 12 bulan × 5 segmen, 9000 baris)
- `8f95027` docs: README cara regenerate, verifikasi, endpoint impor, prasyarat
- Remote: `https://github.com/AkasaTechDev/bbh-dummy-data` (private),
  `main` tracking `origin/main`, working tree bersih.

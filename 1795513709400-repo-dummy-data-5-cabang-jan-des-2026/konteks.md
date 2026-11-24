---
title: konteks
description: Rencana repo terpisah untuk data dummy 5 cabang BBH (Jan-Des 2026)
status: complete
created: "2026-11-24"
updated: "2026-11-24"
---
# konteks

## Kebutuhan user
- Data dummy untuk **5 cabang** — setiap cabang punya Excel Januari–Desember
  2026 — dibuat dalam **repo sendiri** (bukan di repo `bbh-data-board`).
- Git repo juga sekaligus disiapkan ("nanti kita git juga sekalian").

## Daftar cabang (dari user, 24 Nov 2026)
1. Bali Baby Home → key disarankan: `bali`
2. Jogja Baby Home → `jogja`
3. Semarang Baby Home → `semarang`
4. Lombok Baby Home → `lombok`
5. Java Children Home → `java-children`

## Keputusan arsitektur (diskusi Manager, 24 Nov 2026)
- Lokasi: `~/workspaces/bbh-dummy-data/` — SEJAJAR dengan repo lain di
  `~/workspaces/`, BUKAN di dalam `bbh-data-board` (repo aplikasi punya
  rule kanonik container/bind-mount, jangan dibebani 300+ file data).
- Remote: sarankan `AkasaTechDev/bbh-dummy-data` (sama satu organisasi
  dengan `AkasaTechDev/bbh-data-board`).
- Struktur:
  ```
  bbh-dummy-data/
  ├── generate.py            # generator ber-parameter profil cabang
  ├── branches/
  │   ├── bali.yaml … java-children.yaml   # profil tiap cabang
  ├── data/
  │   ├── bali/2026-01-januari/ … 2026-12-desember/{resident,hsh,sec,visit,wh}.xlsx
  │   └── (jogja, semarang, lombok, java-children — pola sama)
  └── README.md
  ```
- Total: 5 cabang × 12 bulan × 5 segmen = **300 berkas .xlsx**.

## Fakta sistem yang wajib selaras
- `facilities.branch_key` ada di DB (idx_facilities_branch) — profil cabang
  harus pakai key konsisten (lihat daftar di atas).
- Plugin `federation` memakai `instanceKey` (contoh: `pusat`, `cabang-jkt`)
  — key cabang di profil dummy sebaiknya memakai nama yang sama saat
  federation diterapkan.
- Generator existing: `api-tests/fixtures/dummy-excel/generate.py` di repo
  aplikasi (basis untuk generator baru; menyalin template asli dari
  `GET /api/admin/<segmen>-template?format=xlsx`).
- Template header identik 1:1; sheet `DATA` saja (validator hanya baca
  sheet DATA; PETUNJUK hilang = versi tak dikenal = cek versi dilewati).
- Prasyarat impor per instance: kategori hsh/sec ter-seed
  (`console maintenance clean-data --yes --reseed`), gudang cabang wajib
  terdaftar (fail-closed), fasilitas boleh NULL.

## Pertanyaan terbuka (untuk WO, konfirmasi user)
1. ~~Jumlah baris per segmen/bulan/cabang — tetap 30?~~ Terjawab: **30**.
2. ~~Unik global lintas cabang (NIK/HP) dengan awalan per cabang, atau unik
   per cabang saja?~~ Terjawab: **unik per cabang**, tanpa awalan buatan.
3. ~~Awalan nomor HP per cabang (disarankan)?~~ Terjawab: **prefix seluler
   nyata per cabang** (`0812`, `0813`, `0821`, `0857`, `0878` untuk ibu;
   `0856`, `0858`, `0877`, `0881`, `0895` untuk pengunjung), bukan `+62-3xx`.

## Hasil (24 Nov 2026)

WO-01 **complete**. Repo sudah ada, ter-push, dan terverifikasi.

- Repo: `~/workspaces/bbh-dummy-data/` →
  `https://github.com/AkasaTechDev/bbh-dummy-data` (private, `main`).
- Isi: `generate.py`, `verify.py`, `validate-all.sh`, `branches/*.yaml` (5),
  `data/<cabang>/2026-01-januari…2026-12-desember/{resident,hsh,sec,visit,wh}.xlsx`,
  `README.md`, `.gitignore`.
- 300 berkas / 9.000 baris / 3,1 MB; deterministik (regenerate → isi sel
  identik).
- Keunikan per cabang dipenuhi tanpa awalan buatan: prefix wilayah NIK
  berbeda (5171/3471/3374/5271/3173), prefix HP berbeda, dan 20 nama
  keluarga yang lepas antar cabang. Generator menolak profil yang
  bertabrakan.
- Verifikasi: `verify.py` **68/68 lulus**; `validate-all.sh` **300 berkas,
  9.000 baris, 0 baris ditolak**; DB aplikasi tidak berubah
  (`residents=0 visit_visits=0 hsh_entries=0 wh_batches=0 admins=20`).
- Commit: `0935e4c` (generator) → `49c7b84` (data) → `8f95027` (README).
- Repo `bbh-data-board` tidak tersentuh (working tree bersih).

Prasyarat impor per instance cabang (detail di README repo data):

1. Kategori hsh (`baby_needs`, `milk`, `food`, `stationery`, `medicine`,
   `housekeeping`, `linen`) dan sec (`elektronik-peralatan`,
   `mainan-rekreasi`) harus ada — pulihkan dengan
   `console maintenance clean-data --yes --reseed` bila instance pernah
   dibersihkan.
2. Tiga gudang per cabang wajib terdaftar (fail-closed):
   `Gudang Utama <Cabang> Dummy`, `Gudang Obat <Cabang> Dummy`,
   `Gudang Linen <Cabang> Dummy`.
3. Fasilitas resident opsional (nama tak terdaftar → `facility_id` NULL).

## Catatan
- Eksekusi repo & generator = Dev Agent (bukan Manager).
- Topic ini dibuat saat diskusi pra-WO; WO menyusul setelah pertanyaan
  terbuka dikunci user.

---
title: konteks
description: "Rencana repo terpisah untuk data dummy 5 cabang BBH (Jan-Des 2026)"
status: create
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
1. Jumlah baris per segmen/bulan/cabang — tetap 30?
2. Unik global lintas cabang (NIK/HP) dengan awalan per cabang, atau unik
   per cabang saja?
3. Awalan nomor HP per cabang (disarankan, mis. +62-3xx dst)?

## Catatan
- Eksekusi repo & generator = Dev Agent (bukan Manager).
- Topic ini dibuat saat diskusi pra-WO; WO menyusul setelah pertanyaan
  terbuka dikunci user.

---
title: "WO-03: Verifikasi & dokumentasi dashboard agregat"
description: "Swagger (endpoint agregat), README (plugin dashboard), koleksi Bruno (smoke + folder dashboard), dan uji RBAC end-to-end 3 blok."
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
tags:
  - work-order
  - dashboard
  - dokumentasi
---

# WO-03: Verifikasi & dokumentasi dashboard agregat

## Referensi
- Pola paritas swagger: WO-09 hsh (`37e7e1f`) — `build.sh --verify-docs`, cocokkan
  jumlah route ↔ operasi swagger.
- Koleksi Bruno: `api-tests/` (folder `smoke/` read-only, folder per-plugin).

## Langkah

1. **Swagger**: anotasi handler `@Router /api/admin/dashboard/summary [get]` dengan
   definisi respons lengkap (3 blok). Regen via `build.sh` (container `app`,
   `--parseDependency --parseInternal`); `--verify-docs` harus paritas.
   Catat: `docs/` gitignored — tidak di-commit.
2. **README**: seksi "Plugin `dashboard` — Ringkasan Lintas Modul" (route, guard,
   semantik periode aktif, fail-open per blok, cara menambah blok baru).
3. **Bruno**:
   - `api-tests/smoke/09-Get-dashboard-summary.bru` (read-only, tag `smoke`).
   - Folder `api-tests/dashboard/`: 01-Get-summary (admin), 02-Get-summary-tanpa-periode
     (bila mudah direproduksi), 03-viewer (200), 04-admin-tanpa-hsh-read (hsh null).
   - Fixture data uji dibuat lalu dibersihkan (pola topic hsh/sec).
4. **RBAC e2e**:
   - Root → 3 blok terisi.
   - Admin grup viewer → 200, 3 blok terisi (viewer punya semua `.read`).
   - Admin dengan hanya `dashboard.read` (+tanpa hsh/sec) → blok `null` sesuai D-5;
     kontrol positif blok yang diizinkan.
5. **Paritas angka**: bandingkan blok hsh/sec/residents dengan endpoint existing
   pada data yang sama (AC-6/AC-7 WO-01, dipanggil ulang dari Bruno).
6. Bersihkan data & akun uji; pastikan migrasi down bersih.

## Kriteria Penerimaan

- [AC-1] Swagger regen byte-konsisten; paritas route ↔ ops OK; `build.sh` exit 0.
- [AC-2] README seksi dashboard ada dan akurat.
- [AC-3] `bru run --env Local --tags=smoke` PASS termasuk request dashboard baru.
- [AC-4] Skenario RBAC 1-4 di atas terbukti (bukti respons JSON dicatat di WO).
- [AC-5] Data/akun uji dibersihkan setelah uji.

## Hasil Implementasi (commit `8dba6c8`, 2026-11-03)

- 11 file: anotasi swagger handler (5 definisi baru: `Summary`, `NeedsBlock`,
  `ResidentsBlock`, `Period`, `CategorySummary`); README seksi "Plugin `dashboard`";
  Bruno folder `dashboard/` (7 request: summary, baseline stats, paritas stats/hsh/sec,
  viewer, limited) + `smoke/09-Get-dashboard-summary.bru`; `bruno.json` + README koleksi.
- AC-1 `build.sh --verify-docs` exit 0 → total **69 path / 113 operasi**; regen byte-identik.
- AC-3 Bruno: `--tags=smoke` 9 req/26 test PASS; `dashboard` 7 req/16 test PASS
  (06/07 auto-skip bila modul tanpa periode aktif).
- AC-4 RBAC e2e: root/viewer 3 blok; limited (`dashboard.read`+`sec.read`) →
  `hsh: null` + sec terisi.
- AC-5 data & akun uji (`dashviewer@`, `dashlimited2@`, grup uji, entri/periode/item
  sec) dibersihkan; master `sec` kembali 2 kategori / 0 item / 0 periode.
- Verifikasi Manager (2026-11-03): curl root → 3 blok benar (hsh `2026-11` total 1/
  urgent 1; sec `period: null`); tanpa token → 401.
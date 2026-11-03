---
title: "SPEC: Dashboard Ringkasan Lintas Modul (resident + hsh + sec)"
description: "Memperluas dashboard admin agar menampilkan ringkasan Kebutuhan Rumah Tangga (hsh) dan Kebutuhan Sekunder (sec) di samping resident, via satu endpoint agregat baru."
type: spec
status: complete
created: "2026-11-03"
updated: "2026-11-03"
tags:
  - dashboard
  - admin-panel
  - agregat
---

# SPEC: Dashboard Ringkasan Lintas Modul (resident + hsh + sec)

## Latar Belakang

Dashboard admin saat ini (hasil WO-07 topic `1793174442984`) hanya menampilkan data
**resident**: `GET /api/admin/residents/stats` → `{total, by_status, by_branch}`.
Modul `hsh` (Kebutuhan Rumah Tangga) dan `sec` (Kebutuhan Sekunder) sudah lengkap
(CRUD, periode, entri, summary) tetapi tidak muncul di dashboard.

Owner meminta dashboard menampilkan ringkasan modul lain juga, melalui **satu
endpoint agregat baru** (bukan SPA memanggil 3 endpoint terpisah).

## Fakta Existing (baseline)

| Sumber | Kontrak | Guard |
|---|---|---|
| `GET /api/admin/residents/stats?branch_key=` | `{total, by_status, by_branch}` | `resident.read` |
| `GET /api/admin/hsh/entries/summary?period_id=` | `{period_id, period_code, total, by_status{URGENT/ENOUGH/UNKNOWN}, by_category[]}` | `hsh.read` |
| `GET /api/admin/sec/entries/summary?period_id=` | sama bentuknya dengan hsh | `sec.read` |

- `resolvePeriod` di kedua plugin menerima **id ATAU code**; bila kosong → error
  periode tidak ditemukan (TIDAK otomatis periode aktif).
- Aturan N-2 (hsh) & S-3 (sec): **maksimal 1 periode aktif** per modul, ditetapkan
  di level aplikasi.
- RBAC: permission `resident.*`, `hsh.*`, `sec.*` sudah ada; dashboard harus tetap
  berguna untuk admin yang hanya punya sebagian permission (fail-open per bagian,
  bukan gagal total).

## Keputusan (usulan — menunggu review owner)

| # | Keputusan | Isi |
|---|---|---|
| D-1 | Endpoint agregat baru | `GET /api/admin/dashboard/summary` — satu panggilan berisi 3 blok: `residents`, `hsh`, `sec` |
| D-2 | Plugin penempat | **Plugin baru `plugins/dashboard`** (aturan user: fitur baru = plugin; `internal/` tidak diubah). Tanpa tabel/migrasi sendiri — hanya route + query baca lintas tabel |
| D-3 | Periode hsh/sec | Endpoint agregat memakai **periode aktif** tiap modul otomatis. Bila tidak ada periode aktif → blok terkait berisi `period: null` + counter nol (bukan error) |
| D-4 | Izin akses | Guard baru `dashboard.read`, di-seed via migrasi plugin `auth` (admin = read, viewer = read). **Jebakan dikenal**: permission wajib di-seed lewat migrasi plugin `auth`, bukan plugin dashboard, kalau tidak fail-closed |
| D-5 | Per-permission fail-open | Bila aktor TIDAK punya `hsh.read`/`sec.read`, blok terkait dikembalikan sebagai `null` (bukan 403 total), sehingga dashboard tetap tampil sebagian |
| D-6 | Bentuk respons | `{residents:{total,by_status,by_branch}, hsh:{period, total, urgent, enough, unknown, by_category}, sec:{...}}` — `by_status` hsh/sec diringkas jadi 3 angka karena itu yang tampil di dashboard |
| D-7 | UI | `DashboardPage.tsx` ditambah 2 kartu/blok: "Kebutuhan Rumah Tangga (periode aktif)" dan "Kebutuhan Sekunder (periode aktif)" — total + jumlah URGENT ditonjolkan, bar per kategori ringkas. Tanpa library chart (pola bar CSS WO-07) |
| D-8 | Tautan cepat | Tambah tautan cepat ke modul kebutuhan (gated `hsh.read` / `sec.read`) |
| D-9 | Tanpa regresi | Kartu resident, sebaran cabang, 5 terbaru, error state (AC WO-07) TIDAK berubah perilakunya |

## Pertanyaan Terbuka — DIKUNCI (owner setuju draft SPEC, 2026-11-03)

- **O-1 — DIKUNCI**: blok hsh/sec menampilkan **bar ringkas per kategori** (dari
  `by_category`) di samping angka total + URGENT yang ditonjolkan.
- **O-2 — DIKUNCI**: periode aktif tapi kosong entri → tampilkan angka `0` biasa
  (tanpa teks khusus), pola kartu tetap stabil.
- **O-3 — DIKUNCI**: iterasi ini cukup **resident + hsh + sec**; kartu lain
  (dokumen/fasilitas) menyusul bila diminta.
- **O-4 — DIKUNCI**: nama route `GET /api/admin/dashboard/summary` final.

Semua keputusan D-1..D-9 ikut terkunci bersama keempat poin di atas. Work order
(WO-01..WO-03) dibuat di `work-order.md` + berkas WO terpisah (status `pending`).

## Rencana Work Order (draft, final menyusul setelah review SPEC)

| WO | Isi | Prasyarat |
|---|---|---|
| WO-01 | Plugin `plugins/dashboard`: skeleton + migrasi permission `dashboard.read` di plugin `auth` + route `GET /api/admin/dashboard/summary` (agregat 3 blok, D-3/D-5) | — |
| WO-02 | UI: kartu hsh + sec di `DashboardPage.tsx`, tautan cepat (D-7/D-8) | WO-01 |
| WO-03 | Verifikasi + dokumentasi: swagger, README, Bruno (`smoke` + folder `dashboard/`) | WO-02 |

## Batasan

- Tanpa library chart; bundle tetap kecil.
- Tanpa cache/agregasi berat — query COUNT sederhana, pola sama dengan `Summary` existing.
- Endpoint agregat TIDAK boleh mengekspos data tanpa guard; fail-closed untuk endpoint utama, fail-open hanya untuk sub-blok sesuai D-5.

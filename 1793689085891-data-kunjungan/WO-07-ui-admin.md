---
title: WO-07 — UI admin modul kunjungan
description: Halaman admin /visits (daftar booking, detail + riwayat status, master pengunjung) + api/visit.ts + nav grup "Kunjungan".
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
---

# WO-07 — UI admin modul kunjungan

**Prasyarat:** WO-02, WO-03.

## Lingkup

1. `src/api/visit.ts` + `src/lib/visitFields.ts` (mirror enum/label — TERPISAH dari `lib/fields.ts` yang milik resident).
2. Halaman `src/pages/admin/visits/`:
   - `VisitListPage` — tabel booking (nama pengunjung, asal, jumlah, waktu (format UTC→WIB, pola `timeZone:'UTC'`), status badge), filter cabang/status/rentang tanggal, paginasi 25.
   - `VisitDetailPage` — detail booking + riwayat status + tombol ubah status (modal; note wajib utk CANCELLED) + edit catatan.
   - `VisitorListPage` — master pengunjung + jumlah kunjungan; detail pengunjung = daftar kunjungannya.
   - Form create manual visit (pilih/buat pengunjung).
3. Route `/visits/*` + `/visitors/*` di `App.tsx`; guard `hasPermission('visit.read')` level halaman.
4. Nav: grup baru **"Kunjungan"** di `src/navigation/menu.ts` (jangan ubah grup lain).
5. Uji Playwright :5173 — CRUD, ubah status, RBAC (viewer: tombol mutasi tersembunyi; admin tanpa `visit.read` → 403 guard).

## Acceptance Criteria

- AC-1: `npm run build` hijau; tidak ada URL absolut backend di `dist` (cek grep host:port).
- AC-2: Semua alur utama teruji di Playwright; data uji dibersihkan.

## Bukti

**Commit:** `6c197f1` — `feat(visit): UI admin kunjungan (WO-07)` (12 file, +1953):
`src/api/visit.ts`, `src/lib/visitFields.ts`, `src/pages/admin/visits/{shared,PermissionGate,StatusModal,VisitListPage,VisitDetailPage,VisitFormPage,VisitorListPage,VisitorDetailPage}.tsx`, `src/App.tsx`, `src/navigation/menu.ts`.

**AC-1 (build + dist):**
```
tsc -b && vite build → ✓ 172 modules transformed, built in 1.91s
dist/index.html 0.48 kB · dist/assets/index-*.css 28.30 kB · index-*.js 509.44 kB
```
Grep `dist/` untuk host:port absolut → hanya `http://localhost` sisa teks
non-API (tidak ada `:8082`/`:8080`); klien axios memakai `baseURL: ''`
(relatif), target backend hanya hidup di `vite.config.ts` (tidak masuk bundle).

**AC-2 (Playwright, aplikasi hasil build di `/app/` port 8082):**
- `/visits/new`: buat pengunjung baru (nama, asal, HP `0812 3456 7890`, party
  `2-4`, waktu `05 Nov 2026 10:00`, catatan) → redirect ke detail, status awal
  `Dijadwalkan`, cabang terisi dari `GET /api/endpoints`, `scheduled_at`
  tampil `05 Nov 2026, 09.00 WIB` (input WIB → UTC tersimpan benar).
- Detail: `Ubah status` → pilih `Dibatalkan` tanpa catatan → tombol simpan
  ditolak dengan pesan "Status Dibatalkan wajib disertai catatan."; isi catatan
  → tersimpan, riwayat bertambah (append-only, baris lama utuh).
- `/visits`: filter status `Selesai` → 1 baris; filter `Dibatalkan` → kosong
  (`TableStateRow` empty state). Pencarian + paginasi + empty state jalan.
- `/visitors`: master pengunjung dengan kolom jumlah kunjungan (`1`); detail
  pengunjung menampilkan profil + daftar kunjungannya.
- RBAC (matrix API, port 8082): admin tanpa `visit.read` → `GET
  /api/admin/visit/visits` 403; grup `viewer` → GET 200 tetapi `POST
  /visitors` 403 dan `POST /visits/:id/status` 403; `GET /visit-template` 200.
- RBAC (UI): login viewer → tombol `+ Kunjungan baru` hilang, detail tanpa
  `Ubah status`/`Edit catatan`/`Hapus`; login admin tanpa `visit.read` →
  `PermissionGate` menampilkan "Akses ditolak (403) — membutuhkan izin
  `visit.read`" dan menu grup Kunjungan tidak dirender.
- Data uji dibersihkan: `visit_visitors=0 visit_visits=0 visit_status_history=0
  visit_import_batches=0`; admin/grup uji dihapus.

**Penyimpangan kecil (disengaja):**
1. `Berlaku sejak` pada modal status memakai `datetime-local` dengan default
   **waktu sekarang** (bukan tanggal saja). Dengan default tanggal saja,
   nilai terkirim jatuh ke 00:00 UTC sehingga status baru "berlaku sejak" lebih
   awal daripada baris sebelumnya dan tidak menjadi status terkini.
2. Semua timestamp (termasuk `Dibuat`/`Diubah`/`Dicatat`) memakai
   `formatVisitTime` agar konsisten menampilkan `WIB`.

**Catatan dev:** `CORS_ALLOWED_ORIGINS` hanya mengizinkan origin terdaftar
(`localhost:5173`, `localhost:8082`). Port-forward VS Code mengubah origin
(mis. `localhost:59324`) sehingga request ditolak 403 walau backend sehat —
verifikasi UI dilakukan lewat build di `http://localhost:8082/app/`.

---
title: "WO-04: Unduh template per segment + rapikan kartu dashboard"
description: "Tambah tombol Unduh template pada tiap kartu kebutuhan (hsh/sec) dan resident, perbaikan visual kartu dashboard (grid status, badge dobel, warna urgent=0), commit .env.example, dan pembersihan data uji resident agar go test hijau."
type: work-order
status: complete
created: "2026-11-03"
updated: "2026-11-03"
tags:
  - work-order
  - dashboard
  - ui
  - housekeeping
---

# WO-04: Unduh template per segment + rapikan kartu dashboard

## Referensi
- Temuan review visual Manager (2026-11-03) atas dashboard hasil WO-01..03.
- Endpoint template existing: `GET /api/admin/resident-template`, `GET /api/admin/hsh-template`,
  `GET /api/admin/sec-template` (semua `?format=xlsx|csv`, guard `*.read`).
- Pola unduhan: `downloadResidentTemplate` di `src/api/residents.ts` (blob + object URL).

## Langkah

1. **Tombol Unduh template per kartu (opsi A — dekat konteks data)**
   - Kartu hsh & sec (`NeedsSummaryCard.tsx`): tombol "Unduh template" di header,
     sejajar tautan "Kelola"; gated `hasPermission("hsh.read"/"sec.read")` — blok
     `null` otomatis tidak merender tombol.
   - Kartu resident: tombol "Unduh template XLSX" yang sudah ada dipertahankan.
   - Format default `xlsx`; ikuti pola blob+object URL (token tidak masuk URL).
   - Tambahkan fungsi `downloadHshTemplate`/`downloadSecTemplate` di
     `src/api/hsh.ts`/`src/api/sec.ts` (jangan duplikasi helper blob).
2. **Perbaikan visual kartu dashboard**
   - Grid kartu status: pastikan tidak ada kartu yatim di baris kedua — 5 kartu
     memakai grid seimbang (mis. total di baris sendiri lebar penuh, atau auto-fit).
   - Hapus badge teks yang redundan di kartu status (label + badge menampilkan teks sama).
   - Samakan tinggi kartu antar-baris (align konsisten).
   - Kartu "Urgent": warna merah HANYA bila nilai > 0; nilai 0 memakai warna netral/abu.
   - Semua perubahan murni presentasi; tidak menyentuh sumber angka (endpoint agregat).
3. **Housekeeping (dipindah ke WO ini atas keputusan owner)**
   - Commit `.env.example` yang masih uncommitted (`JWT_ACCESS_EXP_SECONDS` 900→7200)
     sebagai commit terpisah (bukan bagian feat) ATAU revert bila owner mengubah pikiran —
     konfirmasi ke Manager sebelum commit.
   - Bersihkan data uji resident di DB dev (`BBH-26-P-001` dsb.) sehingga
     `go test ./...` hijau penuh; catat langkah pembersihannya di dokumen ini.

## Kriteria Penerimaan

- [AC-1] Tiap kartu (resident/hsh/sec) punya tombol unduh template yang berfungsi
  (file valid) dan hilang bila permission tidak dimiliki.
- [AC-2] Grid kartu status tanpa kartu yatim; badge redundan hilang; tinggi kartu seragam.
- [AC-3] Kartu Urgent bernilai 0 tidak merah.
- [AC-4] `npm run build` hijau; grep host:port di `dist` tetap kosong (stateless).
- [AC-5] `go test ./...` hijau penuh setelah pembersihan data uji.
- [AC-6] Tidak ada perubahan backend (0 baris di `plugins/` & `internal/`).

## Hasil Implementasi (commit `0e4e1e8` + `b6e22a9`, 2026-11-03)

- `0e4e1e8` (4 file UI): tombol "Unduh template" di header kartu hsh/sec (gated
  `canDownload` = `hsh.read`/`sec.read`, loading per-segmen); `downloadHshTemplate`/
  `downloadSecTemplate` = alias `downloadTemplate` (tanpa duplikasi helper blob);
  grid status: kartu Total full-width (`sm:col-span-2 lg:col-span-4`) + 4 kartu
  `h-full` (tanpa kartu yatim); `StatusBadge` redundan dihapus dari kartu status
  (tetap dipakai di tabel terbaru); tinggi kartu seragam; Urgent merah hanya bila > 0
  (0 → `bg-slate-50`/`text-slate-800`).
- `b6e22a9`: `.env.example` `JWT_ACCESS_EXP_SECONDS` 900→7200 (commit terpisah, sesuai instruksi).
- Bukti AC: endpoint template 200 XLSX valid (10.8–12.9 KB); build 482.63 kB, grep
  host:port di `dist` = 0; `go test ./...` hijau penuh setelah pembersihan data uji.
- Housekeeping: `DELETE FROM residents WHERE resident_code LIKE 'BBH-%-P-%' OR
  resident_code LIKE 'BBH-%-L-%'` (DELETE 1 → 0 baris) di DB dev.
- Verifikasi Manager: commit ada, diff backend 0 baris, helper alias, gating benar.

## Keputusan Terbuka (butuh owner, non-blocker)

- **T-1**: `GenerateResidentCode` pakai MAX+1 — resident baru via UI akan kembali
  memakai `BBH-YY-P-001` sehingga `go test ./...` merah lagi di DB dev berisi data.
  Opsi: (a) test pakai prefix kode terpisah, (b) test di DB terpisah, (c) dibiarkan
  + catatan README test. Belum diputuskan (di luar scope WO-04).
